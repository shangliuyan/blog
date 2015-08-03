title: sqlchemy如何分表
date: 2015-06-29 21:45:39
tags: sqlalchemy
---

## 背景
话说sqlalchemy真是一个非常好用的库，python orm基本上是舍我其谁了，文档还非常全面，基本上没有什么硬伤，现在也冲出了1.0版本，未来更加值得期待。
我最早用django orm，不过很快就觉得很多功能不够用，我当时用的版本是1.3.1，没有*bulk insert*也没有*锁*，没有这两个功能，好多应用就没法用django开发了。之后开始接触sqlalchemy，一直用到现在，总的体会是只有你想不到没有它做不到。

我们项目里有一个需求，就是数据按月分表，比如:2014年6月数据就存在*record_201406*表中, 其他月数据按此方法类推。这个需求如果是用sqlalchemy来获取数据，我们怎么做呢？

## 一般方法有什么问题？
一般情况下，我们很自然想到使用如下方法：

```python
	
	class RecodeDao_201406(Base):
    	__tablename__ = 'record_201406'
    	id = Column(INT(11), primary_key=True)
    ...
```

或者简化点：


```python
	
	class RecodeDao_201406(Base):
    	__table__ = Table('record_201406',
        Base.metadata, autoload=True)
```

这样实现确实没问题，但回到需求上，既然是按月分表，难不成我要每个月写一个这样的model？每月上次线？当然不行，那我们怎么解决呢？

## 官网解决方法，有什么问题？
有经验的同学可能发现，这个不就是水平*sharding*么？这么说不完全对，看一下sharding的*wiki*定义：

>A database shard is a horizontal partition of data in a database or search engine. Each individual partition is referred to as a shard or database shard. Each shard is held on a separate database server instance, to spread load.

我们这个需求只涉及单数据库，就不算*sharding*了，可以称为*partitioning*（分区），然而强大的sqlalchemy这两个情况都考虑到了，并且官网都提供了[example](http://docs.sqlalchemy.org/en/rel_1_0/orm/examples.html#examples-sharding)，我们挑对应场景的partitioning出来看看:

```python

    from sqlalchemy import *
    from sqlalchemy.orm import *
    from sqlalchemy.ext.declarative import declarative_base
    
    Base = declarative_base()
    
    class TBase(object):
        """Base class is a 'mixin'.
    
        Guidelines for declarative mixins is at:
    
        http://www.sqlalchemy.org/docs/orm/extensions/declarative.html#mixin-classes
    
        """
        id = Column(Integer, primary_key=True)
        data = Column(String(50))
    
        def __repr__(self):
            return "%s(data=%r)" % (
                self.__class__.__name__, self.data
            )
    
    class T1Foo(TBase, Base):
        __tablename__ = 't1'
    
    class T2Foo(TBase, Base):
        __tablename__ = 't2'
    
        timestamp = Column(DateTime, default=func.now())
    
    engine = create_engine('sqlite://', echo=True)
    
    Base.metadata.create_all(engine)
    
    sess = sessionmaker(engine)()
    
    sess.add_all([T1Foo(data='t1'), T1Foo(data='t2'), T2Foo(data='t3'),
                 T1Foo(data='t4')])
    
    print sess.query(T1Foo).all()
    print sess.query(T2Foo).all()

```

使用了继承的方法，抽象的好，但我们之前的问题解决了吗？没有。还是需要预定义好所有表的model类，才能正确使用，迫不得已，我们只能自己想办法了。

## 函数方法解决
经过一番探索，我得出了如下方法:

```python

    class_registry = {}                                                                                                                                                                    
    DbBase = declarative_base(bind=engine, class_registry=class_registry)
    
    def get_model(modelname, tablename, metadata=DbBase.metadata):
        """
            args:
                modelname:新model名，string类型
                tablename:数据库中表名
            usage:
              RecordDao = get_model("RecordDao_201406", "record_201406")
        """
        if modelname not in class_registry: 
            model = type(modelname, (DbBase,), dict(
                __table__ = Table(tablename, metadata, autoload=True)
            ))  
        else:
            model = class_registry[modelname]
    return model

```

每次想获取对应月表数据的*model*，调用*get_model*方法即可。这个方法一直沿用到现在，虽然有点丑陋，但却是解决了以上问题。直到sqlalchemy 0.9.1版本推出*Automap*

## Automap方法
sqlalchemy文档完备，具体可点击[Automap](http://docs.sqlalchemy.org/en/rel_1_0/orm/extensions/automap.html)，它可以自动映射数据库的表，通过数据表名映射model，简单直接，实现起来如下：

```python

    from sqlalchemy.ext.automap import automap_base

    AutoBase = automap_base()
    # reflect the tables
    AutoBase.prepare(engine, reflect=True)
    tablename = "record_201406"
    RecordDao = getattr(AutoBase.classes, tablename)

```
这样就可以了，很清晰。但是这个方法有一个缺点，*Automap*的映射虽然是自动的，但是只有在启动的时候生效，也就是说如果新建一个数据表，而没有告诉*Automap*，那这个表是找不到的。在实际使用中，可以捕获AttributeError异常，并再次调用`AutoBase.prepare(engine, reflect=True)` 刷新映射关系。
