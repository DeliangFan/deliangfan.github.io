---
layout: post
title:  "深入理解 nova-api 的 WSGI"
categories: Architectrue
---

本文是 [理解 WSGI 框架](http://wsfdl.com/architectrue/2013/10/14/%E7%90%86%E8%A7%A3WSGI%E6%A1%86%E6%9E%B6.html) 的下篇，重点介绍 WSGI 框架下一些常用的 Python module，最后分析 [nova](https://wiki.openstack.org/wiki/Nova) 是如何使用这些 module 构建其 WSGI 框架。

- [eventlet](http://eventlet.net/): python 的高并发网络库
- [paste.deploy](http://pythonpaste.org/deploy/): 用于发现和配置 WSGI application 和 server 的库
- [routes](https://routes.readthedocs.org/en/latest/): 处理 url mapping 的库

--------------------

# Eventlet

[Eventlet](http://eventlet.net/) 是一个基于协程的 Python 高并发网络库，和上篇文章所用的 [wsgiref](https://docs.python.org/2/library/wsgiref.html) 相比，它具有更强大的功能和更好的性能，openstack 就大量使用 eventlet 以提供并发能力。它具有以下特点：

- 使用 epoll、kqueue 或 libevent 等 I/O 复用机制，对于非阻塞 I/O 具有良好的性能
- 基于[协程(Coroutines)](https://en.wikipedia.org/wiki/Coroutine)，和进程、线程相比，其切换开销更小，具有更高的性能
- 简单易用，特别是支持采用同步的方式编写异步的代码


## Eventlet.wsgi

[Eventlet WSGI](http://eventlet.net/doc/modules/wsgi.html) 简单易用，数行代码即可实现一个基于事件驱动的 WSGI server。本例主要使用了 eventlet.wsgi.server 函数：

~~~ python
eventlet.wsgi.server(sock, site, log=None, environ=None,
                     max_size=None, max_http_version='HTTP/1.1',
                     protocol=eventlet.wsgi.HttpProtocol, server_event=None,
                     minimum_chunk_size=None, log_x_forwarded_for=True,
                     custom_pool=None, keepalive=True,
                     log_output=True, log_format='%(client_ip)s...', 
                     url_length_limit=8192, debug=True,
                     socket_timeout=None, capitalize_response_headers=True)
~~~

该函数具有众多的参数，重点介绍以下 2 个参数：

- sock: 即 TCP Socket，通常由 eventlet.listen('IP', PORT) 实现
- site: WSGI 的 application

回顾上篇文章内容，本例采用 callable 的 instance 实现一个 WSGI application，采用 eventlet.server 构建 WSGI server，如下：

~~~ python
import eventlet
from eventlet import wsgi


class AnimalApplication(object):
    def __init__(self):
        pass

    def __call__(self, environ, start_response):
        start_response('200 OK', [('Content-Type', 'text/plain')])
        return ['This is a animal applicaltion!\r\n']


if '__main__' == __name__:
    application = AnimalApplication()
    wsgi.server(eventlet.listen(('', 8080)), application)
~~~

## Eventlet.spawn

[Eventlet.spawn](http://eventlet.net/doc/basic_usage.html) 基于 greenthread，它通过创建一个协程来执行函数，从而提供并发。

~~~ python
eventlet.spawn(func, *args, **kw)
~~~

加入该函数后，样例如下：

~~~ python
import eventlet
from eventlet import wsgi


class AnimalApplication(object):
    def __init__(self):
        pass

    def __call__(self, environ, start_response):
        start_response('200 OK', [('Content-Type', 'text/plain')])
        return ['This is a animal applicaltion!\r\n']


if '__main__' == __name__:
    application = AnimalApplication()
    server = eventlet.spawn(wsgi.server,
                            eventlet.listen(('', 8080)), application)
    server.wait()
~~~

-------------------

# Paste.deploy

[Paste.deploy](http://pythonpaste.org/deploy/) 是一个用户发现和配置 WSGI server 和 application 的 python 库，它定义简洁的 loadapp 函数，用于从配置文件或者 python egg 中加载 WSGI 应用，它仅关注 application 的入口，不关心 application 的内部细节。

Paste.deploy 通常要求 application 实现一个 factory 的类方法，如下：

~~~ python
import eventlet
from eventlet import wsgi
from paste.deploy import loadapp


class AnimalApplication(object):
    def __init__(self):
        pass

    def __call__(self, environ, start_response):
        start_response('200 OK', [('Content-Type', 'text/plain')])
        return ['This is a animal applicaltion!\r\n']

    @classmethod
    def factory(cls, global_conf, **kwargs):
        return cls()


if '__main__' == __name__:
    application = loadapp('config:/path/to/animal.ini')
    server = eventlet.spawn(wsgi.server,
                            eventlet.listen(('', 8080)), application)
    server.wait()
~~~

配置文件的规则请参考[官网介绍](http://pythonpaste.org/deploy/)，相应的配置文件如下，其中 app:animal 给出了 application 的入口，pipeline:animal_pipeline 用于配置 WSGI middleware。

~~~ ini
[composite:main]
use = egg:Paste#urlmap
/ = animal_pipeline

[pipeline:animal_pipeline]
pipeline = animal

[app:animal]
paste.app_factory = animal:AnimalApplication.factory
~~~

现在我们新增一个 IPBlackMiddleware，用于限制某些 IP：

~~~ python
class IPBlacklistMiddleware(object):
    def __init__(self, application):
        self.application = application

    def __call__(self, environ, start_response):
        ip_addr = environ.get('HTTP_HOST').split(':')[0]
        if ip_addr not in ('127.0.0.1'):
            start_response('403 Forbidden', [('Content-Type', 'text/plain')])
            return ['Forbidden']

        return self.application(environ, start_response)

    @classmethod
    def factory(cls, global_conf, **local_conf):
        def _factory(application):
            return cls(application)
        return _factory
~~~

相关的配置文件为：

~~~ ini
[composite:main]
use = egg:Paste#urlmap
/ = animal_pipeline

[pipeline:animal_pipeline]
pipeline = ip_blacklist animal

[filter:ip_blacklist]
paste.filter_factory = animal:IPBlacklistMiddleware.factory

[app:animal]
paste.app_factory = animal:AnimalApplication.factory
~~~

-------------------

# Route

[Routes](https://routes.readthedocs.org/en/latest/)

Routes is a Python re-implementation of the Rails routes system for mapping URLs to application actions, and conversely to generate URLs. Routes makes it easy to create pretty and concise URLs that are RESTful with little effort.

Routes allows conditional matching based on domain, cookies, HTTP method, or a custom function. Sub-domain support is built in. Routes comes with an extensive unit test suite.

Current features:

- Sophisticated route lookup and URL generation
- Named routes
- Redirect routes
- Wildcard paths before and after static parts
- Sub-domain support built-in
- Conditional matching based on domain, cookies, HTTP method (RESTful), and more
- Easily extensible utilizing custom condition functions and route generation functions
- Extensive unit tests

-------------------

# Nova-api WSGI

## WSGI Server

~~~ python
class Server(object):
    """Server class to manage a WSGI server, serving a WSGI application."""

    default_pool_size = 1000

    def __init__(self, name, app, host='0.0.0.0', port=0, pool_size=None,
                       protocol=eventlet.wsgi.HttpProtocol, backlog=128,
                       use_ssl=False, max_url_len=None):
        """Initialize, but do not start, a WSGI server."""
        self.name = name
        self.app = app
        self._server = None
        self._protocol = protocol
        self._pool = eventlet.GreenPool(pool_size or self.default_pool_size)
        self._logger = logging.getLogger("nova.%s.wsgi.server" % self.name)
        self._wsgi_logger = logging.WritableLogger(self._logger)

        if backlog < 1:
            raise exception.InvalidInput(
                    reason='The backlog must be more than 1')

        bind_addr = (host, port)

        self._socket = eventlet.listen(bind_addr, family, backlog=backlog)

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
            'log_format': CONF.wsgi_log_format
            }

        if self._max_url_len:
            wsgi_kwargs['url_length_limit'] = self._max_url_len

        self._server = eventlet.spawn(**wsgi_kwargs)
~~~

## Application Side

~~~ python
class Loader(object):
    """Used to load WSGI applications from paste configurations."""

    def __init__(self, config_path=None):
        """Initialize the loader, and attempt to find the config.

        :param config_path: Full or relative path to the paste config.
        :returns: None

        """

        self.config_path = config_path or CONF.api_paste_config

    def load_app(self, name):
        """Return the paste URLMap wrapped WSGI application.

        :param name: Name of the application to load.
        :returns: Paste URLMap object wrapping the requested application.
        :raises: `nova.exception.PasteAppNotFound`

        """

        return paste.deploy.loadapp("config:%s" % self.config_path, name=name)
~~~

## Paste Deployment

~~~ ini
[composite:osapi_compute]
use = call:nova.api.openstack.urlmap:urlmap_factory
/v2: openstack_compute_api_v2
/v3: openstack_compute_api_v3

[composite:openstack_compute_api_v2]
use = call:nova.api.auth:pipeline_factory
noauth = faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2

[composite:openstack_compute_api_v3]
...

[filter:keystonecontext]
paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory

[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

[app:osapi_compute_app_v2]
paste.app_factory = nova.api.openstack.compute:APIRouter.factory

[app:osapi_compute_app_v3]
paste.app_factory = nova.api.openstack.compute:APIRouterV3.factory
~~~