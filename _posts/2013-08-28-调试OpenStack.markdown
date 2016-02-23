---
layout: post
title:  "调试 OpenStack"
categories: OpenStack
---

由于 OpenStack 的所有项目都是采用 Python 开发，所以调试 OpenStack 的本质就是调试 Python，Python 的调试通常有以下两种。

- Log：方便简单，适用简单的调试
- Pdb：类似 C 语言的 gdb，提供交互调试源码功能，功能强大

----------

# Log

[OpenStack logging](http://specs.openstack.org/openstack/openstack-specs/specs/log-guidelines.html) 模块是在 [python logging](https://docs.python.org/2/library/logging.html) 基础之上做了封装，它的使用非常简单，以 nova 为例，首先需要导入相关代码文件，获取日志句柄后，即可往该句柄写入日志信息。

~~~ python
from nova.openstack.common import log as logging

LOG = logging.getLogger(__name__)

LOG.debug("Print log.")
~~~

如果文件中已经导入日志模块并获取日志句柄，直接使用该句柄即可。

OpenStack logging 模块提供了丰富的和日志相关的配置项，详情请见 [logging config options](http://docs.openstack.org/icehouse/config-reference/content/list-of-compute-config-options.html#config_table_nova_logging)

----------

# PDB

[Pdb](https://docs.python.org/2/library/pdb.html) 模块是 python 自带的库，它支持设置断点、单步调试源码、查看代码、查看 stack 片段和动态修改变量的值等功能。Pdb 使用非常简单，常用命令如下：

~~~ bash
+----------+--------------+
| commands |  Description |
+----------+--------------+
|    b     |  设断点       |
|    c     |  继续执行程序  |
|    l     |  查看当前片段  |
|    n     |  执行下行代码  |
|    p     |  打印变量的值  |
|    q     |  结束调试程序  |
+----------+--------------+
~~~

## pdb.set_trace

使用该方法时，需在断点处加入以下代码：

~~~ python
import pdb; pdb.set_trace()
~~~

以调试 nova 创建虚拟机为例，在 API 入口处加入上行代码：

~~~ python
@wsgi.response(202)
@wsgi.serializers(xml=FullServerTemplate)
@wsgi.deserializers(xml=CreateDeserializer)
def create(self, req, body):
    """Creates a new server for a given user."""

    # 加入此行代码
    import pdb; pdb.set_trace()

    if not self.is_valid_body(body, 'server'):
        raise exc.HTTPUnprocessableEntity()

    context = req.environ['nova.context']
    server_dict = body['server']
    password = self._get_server_admin_password(server_dict)
    ......
~~~

之后在 shell 中执行以下命令，nova-api 收到创建虚拟机请求时，便会进入该断点：

~~~ bash
$ /usr/bin/python /usr/bin/nova-api
~~~

## python -m pdb debug_file.py

无论是日志还是 pdb.set_trace 方法，均需要修改源代码，有没有一种方法不需要改动文件呢？答案是肯定的，pdb 还提供了另外一种调试模式：

~~~ bash
$ python -m pdb debug_file.py
~~~

依旧以调试 nova 创建虚拟机为例，步骤如下：

~~~ bash
$ /usr/bin/python -m pdb /usr/bin/nova-api

# 设置断点 b file_name.py:line
(pdb) b /usr/lib/python2.6/site-packages/nova/api/openstack/compute/servers.py:781

# 按 c 运行程序，当收到创建虚拟机请求时，便会进入断点
(pdb) c
~~~