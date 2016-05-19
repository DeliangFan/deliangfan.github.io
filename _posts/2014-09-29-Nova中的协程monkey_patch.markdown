---
layout: post
title:  "Nova 中的协程 -- context switch & monkey_patch (三)"
categories: OpenStack
---


![monkey patch](http://7xp2eu.com1.z0.glb.clouddn.com/monkey_patch.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[原图来自](https://www.10monkeys.com/us/)


# Overview


[Context Switch](https://en.wikipedia.org/wiki/Context_switch) 是另一个和并发相随的问题，协程的切换远远小于线程，上下文的切换完全由应用程序负责，操作系统并不感知。以 nova-api 为例，在操作系统看来，该进程只有一个线程，既主线程，虽然主线程里面运行着多个协程，当其中一个协程阻塞(如 IO 阻塞)时，如果没有应用程序调度切换该协程，就会阻塞该线程中其它的协程，影响并发效率。所以应用程序必须能够感知切换的时机，对于阻塞的场景，就需要一种机制来触发切换该协程，并运行其它的协程。

Eventlet 的 [Monkey Patch](http://eventlet.net/doc/patching.html#monkeypatching-the-standard-library) 动态的修改某些标准库，使得库里的阻塞函数被调用时触发切换，把 CPU 片段留给其它的协程。相关函数如下：

- eventlet.patcher.monkey_patch(os=None, select=None, socket=None, thread=None, time=None, psycopg=None)：它支持 monkey patch 8 种 module，如下：
	- os
	- select
	- socket
	- thread
	- time
	- psycopg
	- MySQLdb
	- builtins
- eventlet.patcher.is_monkey_patched(module)：检查某个 module 是否已被 monkey patch 

Eventlet 遇到如下场景时，会触发上下文切换：

- eventlet.sleep()
- 上述被 monkey patch 的 module 的某些函数

--------

# Monkey Patch in Nova

[Eventlet Best Practices](https://specs.openstack.org/openstack/openstack-specs/specs/eventlet-best-practices.html) 详细的介绍 monkey patch 的使用准则。

- Do it first or not at all：此话之意就是如果要使用 monkey patch，那么应该在 python 程序启动时使用，切勿在中途使用，避免引起问题。以 nova 为例，nova/cmd/\_\_init\_\_.py 和 nova/tests/unit/__init__.py 处使用了 monkey patch，而这两个文件，恰恰分别是服务和测试的最初入口

~~~
# TODO(mikal): move eventlet imports to nova.__init__ once we move to PBR
import os
import sys

# NOTE(mikal): All of this is because if dnspython is present in your
# environment then eventlet monkeypatches socket.getaddrinfo() with an
# implementation which doesn't work for IPv6. What we're checking here is
# that the magic environment variable was set when the import happened.
if ('eventlet' in sys.modules and
        os.environ.get('EVENTLET_NO_GREENDNS', '').lower() != 'yes'):
    raise ImportError('eventlet imported before nova/cmd/__init__ '
                      '(env var set to %s)'
                      % os.environ.get('EVENTLET_NO_GREENDNS'))

os.environ['EVENTLET_NO_GREENDNS'] = 'yes'

import eventlet
from nova import debugger

if debugger.enabled():
    # turn off thread patching to enable the remote debugger
    eventlet.monkey_patch(os=False, thread=False)
else:
    eventlet.monkey_patch(os=False)
~~~

- Monkey patching should also be done in a way that allows services to run without it：当 nova-api 运行在 apache 时，它依赖 apache 提供进程级别的并发，所以此时不应使用 monkey patch
- Monkey patching with thread=False is likely to cause problems：我们在调试时，经常会设置 monkey_patch(thread=False)，但是容易引起死锁问题，所以在生产环境中，切记设置 monkey_patch(thread=True)
- Monkey patching can cause problems running flake8 with multiple workers
- Mysql 的 Python driver: [MySQL-Python](https://github.com/farcepest/MySQLdb1) 是一个成熟稳定的库，但是它使用了一些 C 库，故无法被 monkey patch，所以每当访问数据库时，其它的协程被阻塞。到 Kilo 版本时，社区选择纯 Python 编写的 [PyMySQL](https://github.com/PyMySQL/PyMySQL) 作为 DB driver。