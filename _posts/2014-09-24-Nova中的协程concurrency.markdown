---
layout: post
title:  "Nova 中的协程 -- 并发 (一)"
categories: OpenStack
---

- Nova 版本：Icehouse
- Eventlet 版本：0.18.2
- Oslo.messaging 版本：1.3.0

----------

# Overview

OpenStack 由 python 语言编写，主要依赖[协程处理并发事务](http://docs.openstack.org/developer/nova/threading.html)。协程([Coroutine](http://www.dabeaz.com/coroutines/Coroutines.pdf))又称为用户态线程，完全由应用程序负责调度，其上下文切换的开销远远小于线程；由于多线程的效率因[全局解释锁(Global Interpreter Lock)](https://wiki.python.org/moin/GlobalInterpreterLock) 而大打折扣，所以 python 多用协程而非线程处理并发。

[Eventlet](http://eventlet.net/) 是一个高性能的网络库，它依赖两个关键的库：

- greenlet: 协程库，提供并发能力
- epoll/kqueue: 基于事件驱动的网络库，处理网络请求

Eventlet 常用的 API 如下：

- eventlet.spawn(func, *args, **kw)：启动一个协程并获取其返回值
- eventlet.spawn_n(func, *args, **kw)：启动一个协程，但不获取返回值
- eventlet.spawn_after(seconds, func, *args, **kw)：过段时间启动一个协程并获取返回值
- eventlet.sleep(seconds=0)：切换运行中的协程，进入 sleep 状态

Nova 依赖 evenlet 完成各种并发任务，它的进程可分为两类：

- WSGIService: 接收和处理 http 请求，依赖 eventlet.wsgi 的 wsgi server 处理 http 请求，nova-api 是唯一的 WSGIService 类型的进程
- Service: 接收和处理 rpc 请求，非 nova-api 以外的其它 nova 进程，如 nova-scheduler，nova-compute 和 nova-conductor 等

无论是 WSGIService 还是 Service 类型的进程，每当接收到一个请求(http 或 rpc)，都会在线程池中分配一个协程处理该请求，本文主要剖析这两种 Service 的并发，虽然 nova 也会启动某些协程处理其它的任务，但不在本文讨论范围内。

----------

# WSGIService

Nova-api 由 nova/cmd/api.py 启动，它初始化一个 WSGIService(由 nova/service.py 定义) 对象。

~~~ python
def main():
    ...
    launcher = service.process_launcher()
    for api in CONF.enabled_apis:
        should_use_ssl = api in CONF.enabled_ssl_apis
        if api == 'ec2':
            server = service.WSGIService(api, use_ssl=should_use_ssl,
                                         max_url_len=16384)
        else:
            # Initiate a WSGI server.
            server = service.WSGIService(api, use_ssl=should_use_ssl)
        launcher.launch_service(server, workers=server.workers or 1)
    launcher.wait()
~~~

WSGIService 最终调用 nova/wsgi.py 中 Server 类的 start 方法，启动一个 WSGI application，监听和处理 http 请求：

~~~ python
    def start(self):
        """Start serving a WSGI application.

        :returns: None
        """

        wsgi_kwargs = {
            'func': eventlet.wsgi.server,
            'sock': self._socket,
            'site': self.app,
            'protocol': self._protocol,
            'custom_pool': self._pool,
            'log': self._wsgi_logger,
            'log_format': CONF.wsgi_log_format,
            'debug': False
            }

        if self._max_url_len:
            wsgi_kwargs['url_length_limit'] = self._max_url_len

        self._server = eventlet.spawn(**wsgi_kwargs)
~~~

注意 wsgi_kwargs 中的参数 func，它的值为 eventlet.wsgi.server，在 eventlet/wsgi.py 的定义如下：

~~~ python
def server(sock, site,
           ......,
           capitalize_response_headers=True):
    """Start up a WSGI server handling requests from the supplied server
    socket.  This function loops forever.  The *sock* object will be
    closed after server exits, but the underlying file descriptor will
    remain open, so if you have a dup() of *sock*, it will remain usable.

    :param sock: Server socket, must be already bound to a port and listening.
    :param site: WSGI application function.

    ......
    """

    ......

    try:
        serv.log.info("(%s) wsgi starting up on %s" % (
            serv.pid, socket_repr(sock)))
        while True:
            try:
                client_socket = sock.accept()
                client_socket[0].settimeout(serv.socket_timeout)
                serv.log.debug("(%s) accepted %r" % (
                    serv.pid, client_socket[1]))
                try:
                    pool.spawn_n(serv.process_request, client_socket)
                    ......
    finally:
        pool.waitall()
        ......
~~~

看，是不是看到熟悉的一幕了！sock.accept() 监听请求，每当接收到一个新请求，调用 pool.spawn_n() 启动一个协程处理该请求，精简版的逻辑如下：

~~~ python
while True:
	client_socket = sock.accept()
	pool.spawn_n(serv.process_request, client_socket)
~~~

-----------

# Service

Service 类型的进程同样由 nova/cmd/* 目录下某些文件创建：

- nova-compute: nova/cmd/compute.py
- nova-scheduler: nova/cmd/scheduler.py
- nova-conductor: nova/cmd/conductor.py
- ......

作为消息中间件的消费者，它们监听各自的 queue，每当有 rpc 请求来临时，它们创建一个新的协程处理 rpc 请求。以 nova-compute 为例，启动时初始化一个 Server(由 nova/service.py 定义) 对象。

~~~ python
def main():
    ......
    server = service.Service.create(binary='nova-compute',
                                    topic=CONF.compute_topic,
                                    db_allowed=CONF.conductor.use_local)
    service.serve(server)
    service.wait()
~~~

在 Icehouse 版本时，nova 的各个组件依赖 oslo.messaging 访问消息服务器，通过 oslo/messaging/server.py 初始化一个 MessageHandlingServer 的对象，监听消息队列。

~~~ python
class MessageHandlingServer(object):
    """Server for handling messages.

    Connect a transport to a dispatcher that knows how process the
    message using an executor that knows how the app wants to create
    new tasks.
    """

    ......

    def start(self):
        """Start handling incoming messages.

        This method causes the server to begin polling the transport for
        incoming messages and passing them to the dispatcher. Message
        processing will continue until the stop() method is called.

        The executor controls how the server integrates with the applications
        I/O handling strategy - it may choose to poll for messages in a new
        process, thread or co-operatively scheduled coroutine or simply by
        registering a callback with an event loop. Similarly, the executor may
        choose to dispatch messages in a new thread, coroutine or simply the
        current thread. An RPCServer subclass is available for each I/O
        strategy supported by the library, so choose the subclass appropriate
        for your program.
        """
        if self._executor is not None:
            return
        try:
            listener = self.dispatcher._listen(self.transport)
        except driver_base.TransportDriverError as ex:
            raise ServerListenError(self.target, ex)

        self._executor = self._executor_cls(self.conf, listener,
                                            self.dispatcher)
        self._executor.start()

    def wait(self):
        """Wait for message processing to complete.

        After calling stop(), there may still be some some existing messages
        which have not been completely processed. The wait() method blocks
        until all message processing has completed.
        """
        if self._executor is not None:
            self._executor.wait()

    ....
~~~ 

上述的对象又初始化一个 EventletExecutor(由 oslo/messaging/_executors/impl_eventlet.py) 类型的 excuete 对象，它调用 self.listener.poll() 监听 rpc 请求，每当接收到一个请求，创建一个协程处理该请求。

~~~ python
class EventletExecutor(base.ExecutorBase):

    """A message executor which integrates with eventlet.

    This is an executor which polls for incoming messages from a greenthread
    and dispatches each message in its own greenthread.

    The stop() method kills the message polling greenthread and the wait()
    method waits for all message dispatch greenthreads to complete.
    """

    ......

    def start(self):
        if self._thread is not None:
            return

        @excutils.forever_retry_uncaught_exceptions
        def _executor_thread():
            try:
                while True:
                    incoming = self.listener.poll()
                    spawn_with(ctxt=self.dispatcher(incoming),
                               pool=self._greenpool)
            except greenlet.GreenletExit:
                return

        self._thread = eventlet.spawn(_executor_thread)
~~~

简化版的逻辑如下：

~~~ python
while True:
    incoming = self.listener.poll()
    spawn_with(ctxt=self.dispatcher(incoming),
               pool=self._greenpool)
~~~