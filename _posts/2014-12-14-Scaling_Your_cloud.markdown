---
layout: post
title:  "详解 Nova 中的 Region, Cell, AZ 和 Host Aggregates"
categories: OpenStack
---

---------------------

为了提供规模化、分布式部署、资源优化利用和兼容 AWS 的功能，openstack 引入了 Region，Cell，Availability Zone(AZ) 和 Host Aggregates Zone(HAZ) 四个概念，其中 Region 和 AZ 是从公有云大哥 AWS 引入，Cell 是为了扩充一个 Region 下的集群的规模而引入的，Host Aggregates 是优化资源调度和利用引入的。这四个概念均和集群部署相关，某些地方含义有相近之处，对于初学 openstack 的童鞋来说容易混淆和模糊，本文将详细分析这四个概念。

从部署层次来说，它们有以下关系：

>Region > Cell > Availabiliy Zone > Host Aggregates

![az-region-cell](http://7xp2eu.com1.z0.glb.clouddn.com/az_cell_region.jpg)

-----------------------
#Region

顾名思义，Region 直译过来就是区域，地域的概念，而事实上，AWS 按地域(国家或者城市)设置一个 Region，每个 Region 下有多个 Availability Zone。Openstack 同样支持 Region 的概念，支持全球化部署，比如为了降低网络延时，用户可以选择特定的 Region 来部署服务。各个 Region 之间的计算资源、网络资源、存储资源都是独立的，但所有 Region 共享账户用户信息，因为 Keystone 是实现 openstack 租户用户管理和认证的功能的组件，所以 Keystone 全局唯一，所有 Region 共享一个 Keystone，Keystone endpoint 中存储了访问各个 Region 的 URL。

![region](http://7xp2eu.com1.z0.glb.clouddn.com/region.jpg)


-----------------------

#Cell

Cell 概念的引入，是为了扩充单个 Region 下的集群规模，主要解决 AMQP 和 Database 的性能瓶颈，每个 Region 下的 openstack 集群都有自己的消息中间件和数据库，当计算节点达到一定规模(和IBM，easystack，华为等交流的数据是300~500)，消息中间件就成为了扩展计算节点的性能瓶颈。Cell 的引入就是为了解决单个 Region 的规模问题，每个 Region 下可以有多个 Cell，每个 Cell 维护自己的数据库和消息中间件，所有 Cell 共享本 Region 下的 nova-api，共享全局唯一的 Keystone。
 
官网手册提到 Cell 不成熟（Considered experimental），巴黎峰会也提到 Cell 的痛点，虽然现在已进入 K 版本迭代开发中了，但是本人还未听说业界成熟使用 Cell 的案例。关于 Cell [此文](http://www.ibm.com/developerworks/cn/cloud/library/1409_zhaojian_openstacknovacell/index.html)有更详细的介绍。

![cell](http://7xp2eu.com1.z0.glb.clouddn.com/cell.jpg)


-----------------------

#Availability Zone & Host Aggregates Zone

之所以把 AZ 和 HAZ 放到一同分析，是因为二者的概念实在类似。AWS 每个 Region 下有多个 AZ。Openstack 也引入了 AZ 的概念，我个人理解 AZ 的引入是基于安全的角度考虑，比如我们定义一个机房为一个 AZ，把该机房所有计算节点纳入到一个 AZ 中，其中一个机房因为某种原因down 掉，不会影响其它机房的虚拟机和网络；同时， AZ 对用户来说是一个可见的概念，用户创建虚拟机时，可以明确指出在哪个 AZ，用户可以通过在多个 AZ 创建虚拟机来保证高可靠性。

HAZ 也是把一批具有共同属性的计算节点划分到同一个 Zone 中，HAZ 可以对 AZ 进一步细分，一个 AZ 可以有多个 HAZ。 同一个 HAZ 下的机器都具有某种共同的属性，比如高性能计算，高性能存储(SSD)，高性能网络(支持SRIOV等)。HAZ 和 AZ 另一个不同之处在于 HAZ 对用户不是明确可见的，用户在创建虚拟机时不能像指定 AZ 一样直接指定 HAZ，但是可以通过在 Instance Flavor 中设置相关属性，由 nova-scheduler 调度根据该调度策略调度到满足该属性的的 Host Aggregates Zones 中。

![az](http://7xp2eu.com1.z0.glb.clouddn.com/az-haz.jpg)

-----------------------

#How to use availability zone and host aggregates zone 

##How to use availability zone

Nova 调用创建 HAZ 的 API 创建 AZ，即在创建 HAZ 时，定义一个 AZ。

```bash
$ nova aggregate-create HAZ-01 AZ-01
+----+-------+-------------------+-------+----------+
| Id | Name  | Availability Zone | Hosts | Metadata |
+----+-------+-------------------+-------+----------+
| 3  | HAZ-01|  AZ-01            |       |          |
+----+-------+-------------------+-------+----------+
```

创建好 AZ 后，把计算节点加入到该 HAZ，因为 HAZ 属于 AZ，因此新增的计算节点也属于该 AZ。

```bash
$ nova aggregate-add-host 3 compute01
+----+-------+-------------------+---------------+--------------------------------+
| Id | Name  | Availability Zone |      Hosts    |            Metadata            |
+----+-------+-------------------+---------------+--------------------------------+
| 3  | HAZ-01|  AZ-01            | ['compute-1'] | {'availability_zone': 'AZ-01'} |
+----+-------+-------------------+---------------+--------------------------------+
```

创建虚拟机时，指定 AZ 名字即可。

```
$ nova boot –flavor m1.small –image cirros –availability-zone AZ-01 vm
```

##How to use host aggregates zone

nova.conf 新增 AggregateInstanceExtraSpecsFilter filters，如下：

```
scheduler_default_filters=AggregateInstanceExtraSpecsFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter
```

创建一个 HAZ：

```bash
$ nova aggregate-create HAZ-SSD
+----+--------+-------------------+-------+----------+
| Id |  Name  | Availability Zone | Hosts | Metadata |
+----+--------+-------------------+-------+----------+
| 4  | HAZ-SSD|       None        |       |          |
+----+--------+-------------------+-------+----------+
```

设置 Metadata 属性：

```bash
$ nova aggregate-set-metadata 4 ssd=true
+----+--------+-------------------+-------+-----------------+
| Id |  Name  | Availability Zone | Hosts |    Metadata     |
+----+--------+-------------------+-------+-----------------+
| 4  | HAZ-SSD|       None        |  []   | {'ssd': 'true'} |
+----+--------+-------------------+-------+-----------------+
```

添加计算节点到 HAZ：

```bash
$ nova aggregate-add-host 4 compute02
+----+--------+-------------------+---------------+-----------------+
| Id |  Name  | Availability Zone |      Hosts    |    Metadata     |
+----+--------+-------------------+---------------+-----------------+
| 4  | HAZ-SSD|       None        | ['compute02'] | {'ssd': 'true'} |
+----+--------+-------------------+---------------+-----------------+
```

创建 flavor：

```bash
$ nova flavor-create m1.ssd auto 4096 10 2 –is-public true
+----+--------+-----------+------+-----------+------+-------+------------+-----------+---------------+
| ID |  Name  | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor| Is_Public | extra_specs   |
+----+--------+-----------+------+-----------+------+-------+------------+-----------+---------------+
|    | m1.ssd | 4096      | 10   |    0      |      |   2   |    1.0     |    True   |      {}       |
+----+--------+-----------+------+-----------+------+-------+------------+-----------+---------------+
```

设置 flavor 属性：

```bash
$ nova flavor-key m1.ssd set ssd=true

$ nova flavor-show m1.ssd
+----+--------+-----------+------+-----------+------+-------+------------+-----------+------------------+
| ID |  Name  | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor| Is_Public |   extra_specs    |
+----+--------+-----------+------+-----------+------+-------+------------+-----------+------------------+
|    | m1.ssd | 4096      | 10   |    0      |      |   2   |    1.0     |    True   | {'ssd': 'true'}  |
+----+--------+-----------+------+-----------+------+-------+------------+-----------+------------------+
```

采用 m1.ssd 创建虚拟机：

```bash
$ nova boot –flavor m1.ssd –image cirros vm_ssd
```

-----------------

# Reference
1. http://docs.openstack.org/openstack-ops/content/scaling.html
2. http://www.ibm.com/developerworks/cn/cloud/library/1409_zhaojian_openstacknovacell/index.html
3. http://kimizhang.wordpress.com/tag/availability-zone/
4. http://blog.chinaunix.net/xmlrpc.php?r=blog/article&uid=20940095&id=4064233