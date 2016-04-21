---
layout: post
title:  "如何设计 Multi-Region"
categories: OpenStack
---

我于 15 年 3 月在 [OpenStack 安装与运维](http://www.meetup.com/China-OpenStack-User-Group/events/221099393/)的 Meetup 上分享了乐视云计算 Multi-Region 的部署方案(PPT 请见[此处](http://files.meetup.com/10602292/Letv-OpenStack%20Mutil-Region.pdf))，之后不少朋友咨询我具体的实现，于是将设计要点和流程整理成本文，以供大家参考，欢迎指正。

----------------------------

# What is multi-region

Region 者，区域也，表示该区域中云资源或服务的集合，源于 AWS。事实上，AWS 按国家、地区或者城市定义 region，每个 region 的各个服务均有独自的 endpoint，对外提供服务。

![aws](http://7xp2eu.com1.z0.glb.clouddn.com/aws_region.png)

为了兼容 AWS，OpenStack 同样支持 region 概念，所谓的 multi-region 就是将多个 region 统一管理，具有以下特点：

- 所有 region 仅共享唯一的 identity(keystone)。
- 各个 region 之间的资源相互独立，比如: region A 的卷无法挂载给 B 的虚拟机；私有网络无法跨 region 等。
- region 下的每个服务都有对应的 endpoint。

![region](http://7xp2eu.com1.z0.glb.clouddn.com/region.jpg)

--------------------------

# Why use multi-region

使用 multi-region 具有以下优点：

- 支持全球化部署，比如为降低网络时延，用户可以选择所需的 region 来部署服务。
- 提供地域级别的容灾。
- 由于 region 之间共享相同的用户信息，用户只需一次登录，即可访问所有 region 的服务。
- 受消息中间件或者数据库性能的影响，OpenStack 集群的物理节点达到一定规模时，将集群拆分为数个 region 是一个不错的选择。

--------------------------

# How to design the multi-region

本节主要介绍 multi-region 的部署流程以及所需注意的事项，最终提供统一的、易扩展的管理平台。该设计主要涉及以下点：

- Endpint Design
- Choose Token
- Deployment Tips
- OpenStack Version Tips

## Endpoint Design

每个 region 的服务均有自己的 endpoint，包括public url、 admin url 和 internal url，除 keystone 以外，其它服务的三种 url 可设置为一样。良好的命名风格有助于用户对服务的理解，以 public url 为例，推荐格式为：

~~~
https:// + service + region + domain + version~~~

- https: 默认端口为 443，支持 TSL，提高安全性。- service: compute, image, identity 和 network 等等。- region: country + city + id，例如：
 	- 中国北京一区：cn-bj-1 	- 美国洛杉矶二区：us-la-2- domain: lecloud.com- version: v1, v2, v3

以乐视云北京一区集群 nova 的 endpoint 为例，根据以上命名规则，其 endpoint 为：

~~~
https://compute.cn-bj-1.lecloud.com/v2/%(tenant_id)s
~~~

全局唯一的 keystone 的 endpoint 可命名为：

~~~
https://identity.lecloud.com/v3
~~~

采用 keystone endpoint-list 可例出所有的 endpoints，如：

~~~
$ keystone endpoint-list
+-----------+-----------+-----------+-------------+----------+------------+
|   id      |  region   | publicurl | internalurl | adminurl | service_id |
+-----------+-----------+-----------+-------------+----------+------------+
| 034...552 | RegionOne |  ...      |   ...       | ...      | ...        |
| 08f...6c0 | RegionOne |  ...      |   ...       | ...      | ...        |
 ...
| 334...254 | RegionTwo |  ...      |   ...       | ...      | ...        |
| 42f...1c5 | RegionTwo |  ...      |   ...       | ...      | ...        |
+-----------+-----------+-----------+-------------+----------+------------+
~~~

注：

- 域名：如果未申请域名，可在 /etc/hosts 中配置解析。
- https 证书：如果未申请证书，可用 openssl 自制。

## Deployment Tips



## OpenStack Version Tips