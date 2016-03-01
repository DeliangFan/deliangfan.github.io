---
layout: post
title:  "深入理解 nova-api 的 WSGI"
categories: Architectrue
---

--------------------

# Eventlet


[Eventlet](http://eventlet.net/) is a concurrent networking library for Python that allows you to change how you run your code, not how you write it.

- It uses epoll or kqueue or libevent for highly scalable non-blocking I/O.
- Coroutines ensure that the developer uses a blocking style of programming that is similar to threading, but provide the benefits of non-blocking I/O.
- The event dispatch is implicit, which means you can easily use Eventlet from the Python interpreter, or as a small part of a larger application.
- It's easy to get started using Eventlet, and easy to convert existing applications to use it. Start off by looking at examples, common design patterns, and the list of the basic API primitives.

## Eventlet.wsgi

![Eventlet WSGI](http://eventlet.net/doc/modules/wsgi.html)

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

~~~ python
eventlet.wsgi.server(sock, site, log=None, environ=None, max_size=None, max_http_version='HTTP/1.1',
                     protocol=<class 'eventlet.wsgi.HttpProtocol'>, server_event=None,
                     minimum_chunk_size=None, log_x_forwarded_for=True, custom_pool=None, keepalive=True,
                     log_output=True, log_format='%(client_ip)s - - [%(date_time)s] "%(request_line)s" %(status_code)s.6f', 
                     url_length_limit=8192, debug=True, socket_timeout=None, capitalize_response_headers=True)
~~~

	
- sock – Server socket, must be already bound to a port and listening.
- site – WSGI application function.
- log – File-like object that logs should be written to. If not specified, sys.stderr is used.
- environ – Additional parameters that go into the environ dictionary of every request.
- protocol – Protocol class. Deprecated.
- custom_pool – A custom GreenPool instance which is used to spawn client green threads. If this is supplied, max_size is ignored.
- log_format – A python format string that is used as the template to generate log lines. The following values can be formatted into it: client_ip, date_time, request_line, status_code, body_length, wall_seconds. The default is a good example of how to use it.

## Eventlet.spawn

~~~ python
eventlet.spawn(func, *args, **kw)
This launches a greenthread to call func. Spawning off multiple greenthreads gets work done in parallel. The return value from spawn is a greenthread.GreenThread object, which can be used to retrieve the return value of func. See spawn for more details.
~~~

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
    server = eventlet.spawn(wsgi.server, eventlet.listen(('', 8080)), application)
    server.wait()
~~~

-------------------

# Paste.deploy

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
    server = eventlet.spawn(wsgi.server, eventlet.listen(('', 8080)), application)
    server.wait()
~~~

~~~ ini
[composite:main]
use = egg:Paste#urlmap
/ = animal_pipeline

[pipeline:animal_pipeline]
pipeline = animal

[app:animal]
paste.app_factory = animal:AnimalApplication.factory
~~~

## Middleware

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
    def factory(cls, global_conf, **local_conf):
        return cls()


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


if '__main__' == __name__:
    application = loadapp('config:/path/to/animal.ini')
    server = eventlet.spawn(wsgi.server, eventlet.listen(('', 8080)), application)
    server.wait()
~~~

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
/: oscomputeversions
/v1.1: openstack_compute_api_v2
/v2: openstack_compute_api_v2
/v3: openstack_compute_api_v3

[composite:openstack_compute_api_v2]
use = call:nova.api.auth:pipeline_factory
noauth = faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2

[composite:openstack_compute_api_v3]
use = call:nova.api.auth:pipeline_factory
noauth = faultwrap sizelimit noauth_v3 ratelimit_v3 osapi_compute_app_v3
keystone = faultwrap sizelimit authtoken keystonecontext ratelimit_v3 osapi_compute_app_v3
keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v3

[filter:faultwrap]
paste.filter_factory = nova.api.openstack:FaultWrapper.factory

[filter:noauth]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddleware.factory

[filter:noauth_v3]
paste.filter_factory = nova.api.openstack.auth:NoAuthMiddlewareV3.factory

[filter:ratelimit]
paste.filter_factory = nova.api.openstack.compute.limits:RateLimitingMiddleware.factory

[filter:ratelimit_v3]
paste.filter_factory = nova.api.openstack.compute.plugins.v3.limits:RateLimitingMiddleware.factory

[filter:sizelimit]
paste.filter_factory = nova.api.sizelimit:RequestBodySizeLimiter.factory

[filter:keystonecontext]
paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory

[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

[app:osapi_compute_app_v2]
paste.app_factory = nova.api.openstack.compute:APIRouter.factory

[app:osapi_compute_app_v3]
paste.app_factory = nova.api.openstack.compute:APIRouterV3.factory

[pipeline:oscomputeversions]
pipeline = faultwrap oscomputeversionapp

[app:oscomputeversionapp]
paste.app_factory = nova.api.openstack.compute.versions:Versions.factory
~~~