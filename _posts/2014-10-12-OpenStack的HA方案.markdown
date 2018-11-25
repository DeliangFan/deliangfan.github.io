---
layout: post
title:  "OpenStack 的 HA 方案"
categories: OpenStack
---

---------------

根据服务自身状况，HA 分为 Active/Active 和 Active/Passive 两种

- Active/Active:   适合于 stateless/stateful 服务，常用 Load Balancer + Keepalive(VIP) 配置 HA
- Active/Passive: 适用于 stateful 服务，常用 Load Balancer + Keepalive(VIP) + Pacemaker + Corosync 配置 HA

---------------

# OpenStack HA 实现

![OpenStack HA](http://wsfdl.oss-cn-qingdao.aliyuncs.com/HA.png?imageView2/1/w/600/q/100)

## Database(Active/Active):
官方推荐：[MySQL with Galera](http://docs.OpenStack.org/high-availability-guide/content/ha-aa-db.html)

## AMQP(Active/Active): 
官方推荐：[RabbitMQ cluster](https://OpenStack.redhat.com/RabbitMQ)

## OpenStack API(Active/Active):

Keystone, Glance-api, Glance-registry, Neutron-server, Ceilometer-api, Dashboard 均为 stateless 服务，通过 Load Balancer + Keepalive 保证 HA，部署于 Apache server 可提高性能，Haproxy(1.5.0)支持 SSL。

- [Loadbalance for OpenStack API](http://OpenStack.redhat.com/Load_Balance_OpenStack_API)
- [Configuration SSL for haproxy](http://www.b2btech.in/implement-ssl-termination-haproxy-ubuntu-14-04)
- [Runs OpenStack API in apache](http://andy.mc.it/2013/07/apache2-mod_wsgi-OpenStack-pt-2-nova-api-os-compute-nova-api-ec2/# comment-35)

## OpenStack scheduler(Active/Active):

Nova-scheduler, nova-conductor, nova-consoleauth, nova-novncproxy, ceilometer-collector, cinder-scheduler 等均为 stateless 服务，恰当的配置与 AMQP server 连接参数即可，具体详见[社区文档](http://docs.OpenStack.org/high-availability-guide/content/_run_OpenStack_api_and_schedulers.html)

## Memcached(Active/Active):

- 提高 Dashboard 的性能
- 解决 nova-consoleauth 单点问题

## Network:

- neutron DHCP agent(Active/Passive)
- neutron L3 agent(Active/Passive)
- neutron metadata agent(Active/Passive) 
- neutron DVR

[以上三者均由 Pacemaker + Corosync 配置](http://docs.OpenStack.org/high-availability-guide/content/ch-network.html)。
 
## Storage:

[Ceph 保证 Image storage, Volume storage, Nova backend 可靠性](http://www.ceph.com/docs/next/rbd/rbd-OpenStack/)。
