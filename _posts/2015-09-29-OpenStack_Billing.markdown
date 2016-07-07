---
layout: post
title:  "浅谈如何设计一个秒级的 OpenStack 计费系统"
categories: OpenStack
---

![](http://7xp2eu.com1.z0.glb.clouddn.com/openstack_billing.jpg)

经历了五年的成长，OpenStack 的 IaaS 服务愈发成熟，在私有云和公有云领域得到较为广泛的应用，与此同时，相应的计费方案也呼之欲出，特别对于公有云。但是纵观社区的计费项目，它们在功能、成熟度等方面有待提升，加上不同场景下的计费方式差异较大，所以社区的计费项目有时可能无法很好的满足需求。按需按量是云计算的一大重要特点，如果计费系统能够提供更为精细的时间粒度，比如秒级，将有助于提升云平台的弹性。本文浅谈如何设计一个计费系统，并给出一个简单的 demo —— [koala](https://github.com/DeliangFan/koala)。

------------

# 计费资源

在设计计费系统前，首先需要明白哪些资源需要计费，以及这些资源的特点。OpenStack 的服务覆盖了多个领域，从 IaaS 到 PaaS，例如：

- IaaS
  - Compute: instance
  - Network: floating ip, router, load balance
  - Storage: volume, snapshot, image, instance snapshot, object storage.
  - Alarm
- PaaS
  - Database
  - Big Data
  - Redis

本文重点关注 IaaS 计算、存储和网络资源的计费，它们的计费规则通常可以抽象为：

~~~
费用 = 时间 * 规格单价

例如：
虚拟机 = 时间 * 规格单价
存储 = 时间 * (容量 * 每GB单价)
路由器 = 时间 * 规格单价
~~~

但是 floating ip 是一种比较特殊的资源，因为它的计费方式多样：按流量或按带宽计费。如果按带宽计费，可用上述公式。在按流量计费的场景下，它的计费规则为：

~~~
费用 = 流量 * 流量单价 
~~~

--------------

# 理想的计费系统

凡是和钱挂钩的系统，在设计上都必需小心翼翼，三思而后行，这类系统对高可靠、准确和安全性有着严格的要求，OpenStack 的计费系统也不例外。本人认为，一个理想的计费系统必须满足以下需求：

- 准确性：准确的记录资源的使用状况，并生成账单。
- 高可靠性：计费系统需设计成高可靠，避免单点故障。
- 安全性：全方位保证计费系统的安全，比如采用 HTTPS，ACL 等。

也应该尽可能满足以下需求：

- 实时性
- 支持多 region
- 可扩展性：每当有新的云服务上线，这些服务应当可方便地接入到计费系统。
- 秒级计费：精确到秒级计费，提升云平台的弹性。
- 容错能力：本着宁可漏计，不可多计的原则，合理的处理数据的采集异常、发送异常等场景。
- 子账户：支持子账户。

社区主要有两个计费项目，[cloudkitty](https://wiki.openstack.org/wiki/CloudKitty) 和 [stacktach](https://readthedocs.org/projects/stacktach/)。Cloudkitty 依赖 ceilometer 的 sample 计费，只能提供分钟级别的粒度，无法记录资源状态的变更时间点，实时性弱，并且受限于 ceilometer 的性能，支持的资源有限，扩展性差。Stacktach 的计费功能基本不可用。

所以，本文提出另外一种计费系统，它依赖 ceilometer 的 event 信息进行计费。

------------

# 基于 event 的计费系统

博文 [ceilometer 的 sample 和 event](http://wsfdl.com/openstack/2015/09/12/Ceilometer_event_meters.html) 分析了 sample 和 event 这两类数据的特点。

sample 数据虽然为计费提供了一定的信息，但是并不能提供完整的信息，如：无法提供虚拟机的状态，shut off 和 running 状态的计费价格不一致；无法提供虚拟机准确的创建时间，关机时间，即无法提供秒级的计费等。

event 类型数据同样记录了资源的重要信息，event 的数据类型可分为两类：

- 状态变更类型 event：所有的资源，每当其被创建、更新或者删除等时，会及时发出 notification 事件。
- audit 类型 event：nova, cinder 和 neutron 等服务会周期性(每小时)统计各个资源的状态，并发出 notification 事件。 

![instance_event](http://7xp2eu.com1.z0.glb.clouddn.com/instance_state.png)

从全局出发，和 event 相比，sample 具有如下缺点，如：

- 计费指标不全面，没有 volume、floatingIP、snapshot、router 相关的计量数据。
- 无法提供虚拟机的状态，shutoff 和 running 状态的计费价格不一致。
- 无法提供虚拟机准确的创建时间，关机时间，即无法提供秒级的计费。
- 数据量太大问题，sample 一般为数分钟采集一次，其中诸多数据对计费而言，毫无意义。

而采用 event 计费，可有如下优点：

- 准确性：因为 event 记录了资源的状态变更，所以具有高准确性。
- 秒级计费
- 实时性高：秒级的实时性。
- 高扩展性：OpenStack 各个服务基本上都支持 notification。 


下图展示一个基于 event 的计费系统样例的架构：

![koala](http://7xp2eu.com1.z0.glb.clouddn.com/billing_koala.png)

从上可以看出，该计费系统全局唯一，每当各个 region 下的 ceilometer-notification-agent 收集到 event 时，会把这些 event 先过滤，然后做一定处理后经 https 发送至计费系统。如下发给计费系统的一个 event 样例：

~~~ json
{
    "region": "bj",
    "resource_id": "ca9ef7fe-889d-462e-91e4-f05a60abc739",
    "resource_name": "lg",
    "resource_type": "volume",
    "tenant_id": "33294fe9cd6c4150b43b38cd92ea17c5",
    "event_type": "create",
    "event_time": "2015-09-25T07:54:38.235522"
    "content": {"size": 100}
}
~~~

每当计费系统接收到请求后，生成账单信息。如果计费信息发送失败，ceilometer 可标志该 event。如下是根据上述 event 生成的账单信息。

~~~ json
[
    {
        "consumption": 0.0022200000000000002,
        "description": "Resource has been deleted",
        "end_at": "2015-09-25T08:01:48.629053",
        "resource_id": "723566f3-db38-4e37-bdc7-fb0d33856468",
        "start_at": "2015-09-25T08:01:39.504316",
        "unit_price": 0.88800000000000001
    }
]
~~~

在上述架构下，计费系统有如下优点：

- 支持多 region
- 低耦合：计费系统和 OpenStack 的组件解耦，仅通过 https 协议通信，低耦合下计费系统可以不关心 OpenStack 的版本。

这种架构要求 ceilometer 发送计费信息，势必需往 ceilometer 增添代码，这是其中的一个缺点。计费系统本身在实现和部署上还需要注意以下问题：

- 高可靠：koala 就是按照高可靠标准设计的，它只有一个进程 koala-api，可以部署在多个节点。
- 高容错性：它依赖处理计费时具体的逻辑实现。
- 安全性