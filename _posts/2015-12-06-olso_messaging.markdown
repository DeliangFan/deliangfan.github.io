---
layout: post
title:  "理解 RPC 之 oslo.messaging"
categories: OpenStack
---

[RPC(Remote Procedure Call)](https://en.wikipedia.org/wiki/Remote_procedure_call) 是一种远程过程调用协议，它允许一台主机透明的调用另外一台主机的函数(子程序)。RPC 广泛的应用在分布式系统中，它允许系统根据功能和业务解耦成多个模块，提高了系统的扩展性和部署的灵活性等。

[Oslo.messaging](https://wiki.openstack.org/wiki/Oslo/Messaging) 是 OpenStack 非常重要 RPC 公共库，它基于 [AMQP(Advanced Message Queuing Protocol) 协议](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)，为同一个项目内的各个进程之间的通信提供了高级消息队列通信的 API，如 nova-api 和 nova-scheduler，cinder-api 和 cinder-volume 等。本文主要介绍 AMQP 和 oslo.messaging 库。

-------------------

# RPC Overview

RPC 首次在 SUN 公司的 Solaris UNIX 操作系统上，我们称调用者为 client，被调用者为 server，client 和 server 可以分布在不同的主机中，每当 client 调用 server 时，server 端创建一个线程处理 client 端的请求，所以 server 端是一个并发服务器。

RPC 支持同步和异步这两种调用：

- 同步调用：client 发送请求后保持阻塞，直到获取 server 端的返回结果后才继续往下执行。
- 异步调用：client 发送请求后继续执行后续步骤。

~~~
        +--------+          Call            +--------+
        |        |   ------------------>    |        |
        | Client |                          | Server |
        |        |   <------------------    |        |
        +--------+        Response          +--------+
~~~

关于 RPC 更为详细的介绍和使用请参见 [《UNIX 网络编程 卷2 进程间通信》](https://book.douban.com/subject/4118577/) 第五部分。

-------------------

# AMQP Overview

上节介绍的 RPC 具有简单易用的特点，但是存在两个缺点：

- Client 需要知道 server 端的 IP/PORT 才能通信，采用 IP/PORT 标志 server 端不利于系统的维护和扩展。
- 但有大量的 client 和 server 时，client 和 server 端的关系网庞大复杂，不利于系统的维护和扩展。

AMQP 是应用层协议，它在 client 和 server 端引入了消息中间件，解耦了 client 和 server 端，支持大规模下的消息通信，其模型如下：

~~~
        +----------+           +--------+           +----------+
        | Producer |   <--->   |  AMQP  |   <--->   | Consumer |
        | (Client) |           | Server |           | (Server) |
        +----------+           + -------+           +----------+
~~~

![AMQP Overview](http://7xp2eu.com1.z0.glb.clouddn.com/amqp_fanout.png)

在 AMQP 的术语中，我们把 client 端称为 producer，server 端称为 consumer。除此外，还有以下概念：

- Topic: 即消息，由消息头和消息体组成，消息头部包含了 routing-key 等属性，exchange 就是根据 routing-key 把消息发送到对应的队列。
- Exchange: 即消息转发器，producer 先把消息发送到 exchange 中，exchange 根据消息属性把消息转发到相应的队列。
- Queue: 即消息队列，接收和存储由 exchange 转发来的消息，以供 consumer 消费。
- Bind: 即 queue 和 exchange 之间的绑定，绑定时定义了转发规则。

AMQP 支持三种调用：

- Call: 同步调用，但是过程稍微复杂，producer 发送消息后立刻创建一个 direct consumer, 该  direct consumer 阻塞于接收返回值。对端的 consumer 接收并处理 producer 的消息后，创建一个 direct producer，它负责把处理结果发送给 direct producer，如下图。  

![AMQP Overview](http://7xp2eu.com1.z0.glb.clouddn.com/amqp_call.png)

- Cast: 异步调用，producer 发送消息后继续执行后续步骤，consumer 接收处理消息，如下图。

![AMQP Overview](http://7xp2eu.com1.z0.glb.clouddn.com/amqp_cast.png)

- Fanout: 相当于广播，producer 可把消息发送给多个 consumer，属于异步调用范畴，如下图。

![AMQP Overview](http://7xp2eu.com1.z0.glb.clouddn.com/amqp_fanout.png)

-------------------

# Oslo Messaging

OpenStack 有三大常用消息中间件，RabbitMQ，QPID 和 ZeroMQ，这三种的参数和接口各异，不利于直接使用，所以 oslo.messaging 对这三种消息中间件做了抽象和封装，为上层提供统一的接口。

~~~

                     +----------------+
                     | Olso Messaging |
                     +----------------+
        -------------------------------------------
            +-----------+  +------+  +--------+
            | Rabbit MQ |  | Qpid |  | ZeroMQ |
            +-----------+  +------+  +--------+
~~~ 

Oslo.messaging 抽象出了两类数据：

- Transport: 消息中间件的基本参数，如 host, port 等信息。
  - RabbitMQ:host, port, userid, password, virtual_host, durable_queues, ha_queues 等。
  - Qpid: hostname, port, username, password, protocol (tcp or ssl) 等。
  - ZeroMQ: bind_address, host, port, ipc_dir 等。
- Target: 主要包括 exchange, topic, server (optional), fanout (defaults to False) 等消息通信时所用到的参数。

Oslo.messaging 代码框架清晰，简单易读，下面是一个简单样例。

Server 端代码如下：

~~~
from oslo_config import cfg
import oslo_messaging
import time

class ServerControlEndpoint(object):

    target = oslo_messaging.Target(namespace='control')

    def __init__(self, server):
        self.server = server

    def stop(self, ctx):
        if self.server:
            self.server.stop()

class TestEndpoint(object):

    def test(self, ctx, arg):
        return arg

transport = oslo_messaging.get_transport(cfg.CONF)
target = oslo_messaging.Target(topic='test', server='server1')
endpoints = [
    ServerControlEndpoint(None),
    TestEndpoint(),
]
server = oslo_messaging.get_rpc_server(transport, target, endpoints,
                                       executor='blocking')
try:
    server.start()
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    print("Stopping server")

server.stop()
server.wait()
~~~

Client 端代码如下：

~~~ python
import oslo_messaing as messaging

transport = messaging.get_transport(cfg.CONF)
target = messaging.Target(topic='test', version='2.0')
client = messaging.RPCClient(transport, target)
client.call(ctxt, 'test', arg=arg)
~~~


相关文章推荐：

- [AMQP and Nova](http://docs.openstack.org/developer/nova/rpc.html)
- [Oslo Messaging](https://wiki.openstack.org/wiki/Oslo/Messaging)
- [RabbitMQ: Remote Procedure Call](https://www.rabbitmq.com/tutorials/tutorial-six-python.html)