---
layout: post
title:  "记一次 K8S 自研网络插件的泛洪问题"
categories: Kubernetes
---

## 蘑菇街 Kubernetes 网络插件简介

在物理机时代，蘑菇街基于 Neutron VLAN 的网络方案保证了容器和虚拟机，物理机处于同一个三层的网络平面，它们之间的网络是互通的，从而满足当前技术体系的要求，为业务平滑容器化奠定基础。

当 PaaS 往公有云云主机上迁移时，为了保证容器网络的互通性，即容器和云主机处于同一个三层的网络平面，我们基于腾讯云弹性网卡支持配置多个 IP 的特性，自研公有云上的 Kubernetes 网络插件，将腾讯云底层的网络能力赋予给容器。具体如下图所示：

![cni](http://wsfdl.oss-cn-qingdao.aliyuncs.com/mogu-k8s-cni.png)

首先创建 ovs br100 网桥，弹性网卡 eth1 桥接到 br100。当容器 A 创建时，网络插件为容器创建网卡，mac 地址为 mac A，并桥接到 br100。然后公有云插件向腾讯云为 eth1 申请一个 IP A，并将 IP A 配置给容器 A。

从腾讯云底层的视角来看，IP A 和 IP B 都是属于 eth1，所以宿主机外部发给容器的报文到达 eth1 时，目的 mac 均为 eth1 的 mac 地址 s，为了保证报文能够被对应的容器接收，必须将目的 mac 转换成对应容器的 mac。对于每个容器，以容器 A 为例，需要添加流表 1：

```
ovs-ofctl add-flow br100 "ip,in_port=1,nw_dst=ip_a actions=mod_dl_dst:mac_a,NORMAL"
```

但压测发现存在严重的性能问题，经和腾讯云定位分析，由于所有发出宿主机的报文均为容器的报文，eth1 未发过报文，所以云主机的母机中和 eth1 连接的网桥无法学习到 eth1 的 mac 地址，导致母机泛洪，引发限速。

为了能够让母机网桥学习到 eth1 的 mac 地址，对于每个容器发出去的报文，均将其源 mac 地址转换为 eth1 的 mac 地址，以容器 A 为例，增加流表 2：

```
ovs-ofctl add-flow br100 "ip,dl_src=mac_a actions=mod_dl_src:mac_s,NORMAL"
```

解决性能问题后，大量容器在该网络方案下运行近半年，未发现任何异常，直到一次抓包。

## Flooding 现象和原因分析

定位其它问题时，在容器 A 上竟然能抓到宿主机上其它容器入口流量的报文，其它容器也有同样的问题，即容器的入口流量出现泛洪(flooding)。

过去经验告诉我们，网桥在未学习到 mac 地址的情况下会泛洪。查询网桥的 mac 学习表后发现容器的 mac 地址果然没有被学习到：

```
$ ovs-appctl fdb/show br100
 port  VLAN  MAC                Age
    1     0  mac_gateway        64
  257     0  mac_s              0
```

发往容器 A 的报文被流表 1 修改目的地址为 mac a 后，由于网桥不知道 mac a 的设备在哪个端口，网桥将报文发向所有的端口，因此其它容器均能看到发往容器 A 的报文。

为什么容器的 mac 地址没有被学习到呢？根本原因是流表 2 将容器发出的所有 ip 类型报文的源地址修改成 eth1 的 mac 地址，导致网桥无法感知容器 mac 地址的存在，即无法学习到容器的 mac 地址。

我们曾认为由于流表 2 对 arp 报文不生效，如果容器发出 arp 报文，网桥应该能够学习到容器的 IP。事实上也是如此，但是容器的 arp 报文并没有想象中频繁，而网桥对 mac 学习表项有个缓冲时间，当过期后，如果容器还未发送 arp 报文，那么发往该容器的报文就会被泛洪。

## 解决方案

分析清楚原因后，解决问题的思路也就随之而来，方法有多种：

- 方案一：删除流表 2，宿主机上起个 crontab，每分钟从 eth1 对外发送个报文，让母机的网桥学习到 eth1 的 mac 地址。由于 eth1 没有配置 ip，最常用的 ping 报文肯定不行，但是可以利用 arping 发送 arp 报文。

```
arping -I eth1 8.8.8.8 -c 1
```

- 方案二：修改流表 1，根据目的 ip 将报文发送到指定容器所连接的网桥端口，报文在网桥上彻底依赖流表转发，不再依赖网桥学习的 mac 表项进行转发。

```
ovs-ofctl add-flow br100 "ip,in_port=1,nw_dst=ip_a actions=mod_dl_dst:mac_a,output:port_a,NORMAL"
```