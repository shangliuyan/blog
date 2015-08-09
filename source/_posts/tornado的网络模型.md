title: tornado的网络模型
date: 2015-07-19 18:36:45
tags: [python,tornado]
---
在网站逐步发展的过程中，很可能就会遇到[C10K](http://www.kegel.com/c10k.html)问题，python中一个比较流行的解决方式是通过tornado这个web server解决。
tornado是一个非阻塞的Web服务器，下面我会结合源码，对tornado进行一下结构来说明。

## 基础理论
在这之前，我们需要先弄清楚几个概念，便于我们理解tornado。
* 同步、异步、阻塞和非阻塞这几个概念的区别和联系。这个问题网上内容很多，可以看下[知乎这个问题的讨论](http://www.zhihu.com/question/19732473)，个人认为还是不错的，几个高票的回答从不同角度说明了问题，看过后应该会有比较清晰的认识。
* epoll的原理。这个是tornado的核心，在网上前人说了很多。我推荐这个[知乎讨论]（http://www.zhihu.com/question/20122137）,@蓝形参和@张亚伟的回答结合看，应该就能理解大概的原理

## 核心模块
tornado的核心模块分为三部分：

* httpserver - 服务于 web 模块的一个非常简单的 HTTP 服务器的实现
* iostream - 对非阻塞式的 socket 的简单封装，以方便常用读写操作
* ioloop - 核心的 I/O 循环

我们结合tornado官网的*hello world*，看看整个过程到底是怎么进行的。
hello_world.py:


```python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.web

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    if __name__ == "__main__":
        application = tornado.web.Application([
            (r"/", MainHandler),
        ])
        http_server = tornado.httpserver.HTTPServer(application)
        http_server.listen(8888)                                                                                                                              
        tornado.ioloop.IOLoop.instance().start()
```

当我们运行python hello_world.py，先实例化一个tornado.web.Application实例*application*(这个类并不是核心范畴，所以在这里暂且不做说明，可以简单理解为对数据请求进行url路由，并且生成返回数据的一个handler。在hello_world.py中，我们对根目录url的请求，返回“hello world”内容的数据)。

然后将*application*作为参数，实例化tornado.httpserver.HTTPServer，赋值给http_server，http_server再调用listen方法。
这个过程发生了什么，我们来看看httpserver.py。


```python

    def __init__(self, request_callback, no_keep_alive=False, io_loop=None,
                 xheaders=False, ssl_options=None):
        """Initializes the server with the given request callback.

        If you use pre-forking/start() instead of the listen() method to
        start your server, you should not pass an IOLoop instance to this
        constructor. Each pre-forked child process will create its own
        IOLoop instance after the forking process.
        """
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.io_loop = io_loop
        self.xheaders = xheaders
        self.ssl_options = ssl_options
        self._socket = None
        self._started = False

    def listen(self, port, address=""):
        """Binds to the given port and starts the server in a single process.

        This method is a shortcut for:

            server.bind(port, address)
            server.start(1)

        """
        self.bind(port, address)
        self.start(1)
```

实例化方法除了将*application*赋值为request_callback外，都是使用的默认值，注意这里*io_loop=None*。listen方法比较简单，绑定socket然后调用start方法。start方法代码比较多，只看主要逻辑。

```python
   if not self.io_loop:
       self.io_loop = ioloop.IOLoop.instance()
   self.io_loop.add_handler(self._socket.fileno(),
                            self._handle_events,
                            ioloop.IOLoop.READ)
```
终于出现io_loop，在实例化httpserver的时候io_loop=None，这里将ioloop.IOLoop.instance()赋值给ioloop。值得一提的是ioloop采用单实例模式，所以ioloop.IOLoop.instance()就是返回ioloop的实例。现在跳转到ioloop.py。

```python

    def add_handler(self, fd, handler, events):
        """Registers the given handler to receive the given events for fd."""
        self._handlers[fd] = handler
        self._impl.register(fd, events | self.ERROR)

```

可以看到add_handler方法将fd文件描述符和events事件注册到_impl上，这个_impl可以认为是epoll类（根据操作系统不同可能为kqueue），另外将handler赋值给self._handlers[fd]，_handlers[fd]相当于文件描述符和回调函数对应的字典，当事件触发的时候会调用。回到httpserver.py，我们调用的是

```python

   self.io_loop.add_handler(self._socket.fileno(),
                            self._handle_events,
                            ioloop.IOLoop.READ)
```

文件描述符是监听端口的socket，事件READ可以认为I/O是数据读取就绪，回调函数为self._handle_events:

```python

    def _handle_events(self, fd, events):                                                                                                                     
        while True:
            try:
                connection, address = self._socket.accept()
            except socket.error, e:
                if e[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                raise
            if self.ssl_options is not None:
                assert ssl, "Python 2.6+ and OpenSSL required for SSL"
                connection = ssl.wrap_socket(
                    connection, server_side=True, **self.ssl_options)
            try:
                stream = iostream.IOStream(connection, io_loop=self.io_loop)
                HTTPConnection(stream, address, self.request_callback,
                               self.no_keep_alive, self.xheaders)
            except:
                logging.error("Error in connection callback", exc_info=True)
```

现在我们可以梳理一下整个过程，httpserver监听一个端口，并把这个文件描述符通过ioloop注册到epoll，只要收到请求，epoll回调_handle_events方法。这个方法做了什么呢？
根据请求数据，创建了一个socket连接*connection*，然后将这个connection作为文件描述符实例化另一个核心模块*iostream*，赋值为steam，然后使用steam和self.request_callback(tornado.web.Application实例的*application*)作为主要参数，实例HTTPConnection。
HTTPConnection负责HTTP协议部分，它的I/O使用iostream，通过iostream read方法读取数据解析数据包，然后调用application生成返回数据，在调用iostream write方法将数据返回。这个过程的I/O事件注册就靠iostream.
我们看下实例iostream的代码

```python

    def __init__(self, socket, io_loop=None, max_buffer_size=104857600,
                 read_chunk_size=4096):
        self.socket = socket
        self.socket.setblocking(False)
        self.io_loop = io_loop or ioloop.IOLoop.instance()
        self.max_buffer_size = max_buffer_size
        self.read_chunk_size = read_chunk_size
        self._read_buffer = ""
        self._write_buffer = ""
        self._read_delimiter = None
        self._read_bytes = None
        self._read_callback = None
        self._write_callback = None
        self._close_callback = None
        self._state = self.io_loop.ERROR                                                                                                                      
        self.io_loop.add_handler(
            self.socket.fileno(), self._handle_events, self._state)

    def read_until(self, delimiter, callback):
        """Call callback when we read the given delimiter."""
        assert not self._read_callback, "Already reading"
        loc = self._read_buffer.find(delimiter)
        if loc != -1:
            self._run_callback(callback, self._consume(loc + len(delimiter)))
            return
        self._check_closed()
        self._read_delimiter = delimiter
        self._read_callback = callback
        self._add_io_state(self.io_loop.READ)


    def write(self, data, callback=None):
        """Write the given data to this stream.
        """
        self._check_closed()
        self._write_buffer += data
        self._add_io_state(self.io_loop.WRITE)
        self._write_callback = callback

    def _add_io_state(self, state):
        if not self._state & state:
            self._state = self._state | state
            self.io_loop.update_handler(self.socket.fileno(), self._state) 
```

可以看到在实例iostream的时候，我们就将返回给客户端的文件描述符注册到ioloop上，但是事件是ERROR。在write和read_until方法中，都调用了_add_io_state方法，这个方法负责更新对应文件描述符的注册事件。

现在我们来看看tornado所谓的单线程主要的任务调度逻辑，ioloop中start方法。

```python

    def start(self):
        ...
        while True:
        ...
            try:
                event_pairs = self._impl.poll(poll_timeout)
            except Exception, e:
                # Depending on python version and IOLoop implementation,
                # different exception types may be thrown and there are
                # two ways EINTR might be signaled:
                # * e.errno == errno.EINTR
                # * e.args is like (errno.EINTR, 'Interrupted system call')
                if (getattr(e, 'errno') == errno.EINTR or
                    (isinstance(getattr(e, 'args'), tuple) and
                     len(e.args) == 2 and e.args[0] == errno.EINTR)):
                    logging.warning("Interrupted system call", exc_info=1)
                    continue
                else:
                    raise
        ...
            self._events.update(event_pairs)
            while self._events:
                fd, events = self._events.popitem()
                try:
                    self._handlers[fd](fd, events)
                except (KeyboardInterrupt, SystemExit):
                    raise
                except (OSError, IOError), e:
                    if e[0] == errno.EPIPE:
                        # Happens when the client closes the connection
                        pass
                    else:
                        logging.error("Exception in I/O handler for fd %d",
                                      fd, exc_info=True)
                except:
                    logging.error("Exception in I/O handler for fd %d",
                                  fd, exc_info=True)
```

在这里 ```event_pairs = self._impl.poll(poll_timeout)```，陷入epoll，然后```while self._events```之后的代码，运行触发事件后回调函数。
现在我们通过一张图看看整个流程：
![tornado-httpserver图](/img/tornado-httpserver.png)

## 还有什么

我们看到在tornado中，无论运行什么库，只要涉及I/O，都要注册到ioloop上，这样才能发挥异步I/O的作用，否则tornado也会阻塞。所以tornado会有很多第三方库，所以在实际使用中，我们有必要学习一下第三方库的使用。
