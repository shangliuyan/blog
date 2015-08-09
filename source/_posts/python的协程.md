title: python的协程
date: 2015-08-06 15:18:35
tags:
---
关于python的协程，网上资料还是挺多的，这里说一下我的理解吧。
## 什么是协程？
先看一下wiki的定义吧

>Coroutines are computer program components that generalize subroutines for nonpreemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations.

这个定义不太容易理解。一句话可能很难直观的说明协程这个概念，通俗讲，协程是由一系列的子程序协同完成一个任务，这些子程序可以主动挂起交出控制权，当恢复执行的时候，可以从挂起的位置继续执行，而这一切的调度由用户操作，而不是操作系统。所以有人称，*协程是用户态线程*。
协程的实现并不与操作系统相关，是语言相关的，所以可以看到主流的一些语言都有协程的实现，包括java，go等。在python中，协程是通过生成器实现的，*yeild*就可以保存当前子程序上下文，并交出控制权，使用send就可以传递数据并恢复相应子程序。这样多个生成器子程序，就可以通过yield和send相互协作完成任务。

## python的协程和生成器的关系
说到这里，可能会产生疑问，使用*yield*的函数不是生成器么？生成器就是协程么？确实如此，参见[PEP 342](https://www.python.org/dev/peps/pep-0342/)，在python 2.5以前生成器就是仅仅是迭代器函数，可以生成无限列表。但是yield保存上下文，主动交出控制权的特性已经很接近协程了，所以在python 2.5对生成器进行了几个改进：
* yield从语句变为表达式, 这个是为了传值方便
* 加入send()方法用于在恢复生成器的时候，传入值
* 加入close()方法用于结束协程
* 加入throw()方法用于传入异常
加入了send()和throw()方法，我们就可以在协程恢复的时候，传入值或者异常。有了这些特性，python从语言上就支持基本的协程功能了，当然对不同协程的控制，还要用户自己来编写。所以我们可以说，从python 2.5以后的生成器才可以用于作为协程。

## 与greenlet，gevent的关系
介绍完python在语义上对协程的支持，但实际使用中会发现，很少有用生成器方式的协程，一般用greenlets，gevent这样的package代替。为什么会这样呢？主要还是因为python2.x对协程的支持有限，要支持复杂的应用比较困难，但是在python3以后，协程会更加好用，我们看看python对协程的支持历程：
>Implementations for Python
>   * Python 2.5 implements better support for coroutine-like functionality, based on extended generators (PEP 342)
>   * Python 3.3 improves this ability, by supporting delegating to a subgenerator (PEP 380)
>   * Python 3.4 introduces a comprehensive asynchronous I/O framework as standardized in PEP 3156, which includes coroutines that leverage subgenerator delegation
>   * Python 3.5 introduces explicit support for coroutines with async/await syntax (PEP 0492).

到Python 3.5都已经有明确的异步操作方法了，但是这些都是2.x所不具备的。所以在python2.x时代，就需要其他实现方式作为补充。
greenlet就是这样一个库，它是从stackless python中剥离，支持CPython的版本的一个协程模块, 相当于python协程的增强版。在github上有使用greenlet重新实现生成器的[demo](https://github.com/python-greenlet/greenlet/blob/master/tests/test_generator.py)，大家可以体会一下greenlet的特性。但是和python 生成器协程一样，greenlet也没有控制调度的功能，如果要实现一个非阻塞的操作，还要自己实现控制调度逻辑，这就催生了gevent的产生。

> gevent is a coroutine-based Python networking library that uses greenlet to provide a high-level synchronous API on top of the libev event loop.
> Features include:
>   * Fast event loop based on libev (epoll on Linux, kqueue on FreeBSD).
>   * Lightweight execution units based on greenlet.
>   * API that re-uses concepts from the Python standard library (for example there are Events and Queues).
>   * Cooperative sockets with SSL support »
>   * DNS queries performed through threadpool or c-ares.
>   * Monkey patching utility to get 3rd party modules to become cooperative »

gevent在greenlet基础上结合libev作为事件循环，补充了协程要自己写异步调度的空缺，最大化了协程的性能。在此基础上，还提供了多种API，方便开发，甚至直接提供了一个支持协程WSGI server，bottle就支持了这个特性。另外值得一提的是它的*Monkey patch*，可以无缝将python标准库中阻塞的API包装成非阻塞的，这一特性大大提高了gevent的应用率。

## 协程带来的改变
面向对象是和现实世界构成的形式一致的，但是不同对象之间的交互，还是采用调用的关系。调用关系隐含的的是主从关系，但现实世界，很多关系的协作是对等的，比如生产者和消费者。协程就是在计算机程序设计中对这种现实反映的实现。
下面我们结合程序简要说明协程的几种应用：
* 无限列表
    假设一种情形，我们需要所有的斐波那契数列，如果不用协程基本上实现不了吧。协程实现就很方便：
    ```python
       def fib():
            first, second = 0, 1
            yield first
            yield second
            while True:
                third = first + second
                yield third
                first = second
                second = third
    ```

* 管道
    如果我们想用python实现管道怎么做呢，答案也是协程。下面这个例子来自于[python参考手册](https://books.google.co.jp/books?id=Chr1NDlUcI8C&pg=PA108&lpg=PA108&dq=coroutine+find_files+cat+grep&source=bl&ots=OCDDzmgZCq&sig=6ldU6qqPblKMl0Qv9li3fBmBsyQ&hl=zh-CN&sa=X&ved=0CCIQ6AEwAGoVChMIxL-D1ciWxwIVzQiOCh26tgyg#v=onepage&q=coroutine%20find_files%20cat%20grep&f=false)，打印指定目录下，满足格式文件中，有“python”关键字的行
    
    ```python
    
    import os
    import fnmatch
    import gzip, bz2
    
    def coroutine(func):
    """自动调用协程的next()函数"""
        def start(*args, **kwargs):
            g = func(*args, **kwargs)
            g.next()
            return g
        return start
    
    @coroutine
    def find_files(target):
        while True:
            topdir, pattern = (yield)
            for path, dirname, filelist in os.walk(topdir):
                for name in filelist:
                    if fnmatch.fnmatch(name, pattern):
                        target.send(os.path.join(path, name))
    
    @coroutine
    def opener(target):
        while True:
            name = (yield)
            if name.endswith('.gz'): f = gzip.open(name)
            elif name.endswith('.bz2'): f = bz2.BZ2File(name)
            else: f = open(name)
            target.send(f)
    
    @coroutine
    def cat(target):
        while True:
            f = (yield)
            for line in f:
                target.send(line)
    
    @coroutine
    def grep(pattern, target):
        while True:
            line = (yield)
            if pattern in line:
                target.send(line)
    @coroutine
    def printer():
        while True:
            line = (yield)
            sys.stdout.write(line)
    
    finder = find_files(opener(cat(grep('python', printer))))

finder.send('www', 'access-log*')
finder.send('otherwww', 'access-log*')
```

* 并发
    关于并发，最好的例子应该是gevent吧，大家有兴趣可以看下源码。基本的原理是，将函数变为协程，每触发I/O阻塞就yield交出控制权，并将事件注册到epoll，当I/O就绪就是用send方法，传入I/O数据，并恢复逻辑。这么描述其实和tornado、nodejs的网络模型很像，但是协程对于程序员更加友好。tornado和nodejs默认还是通过回调函数完成这个事件循环的，这样代码并不直观，但使用协程可以用同步的方式完成回调函数的工作。


