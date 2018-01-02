---
layout: post
title:  "关于 CPU mode 选取的一些经验"
categories: OpenStack
---

在 OpenStack 中，大家对 [CPU mode](https://wiki.openstack.org/wiki/LibvirtXMLCPUModel) 的关注相对较少，多采用默认值。其实，CPU mode 的选取对云平台的影响却很大，如果考虑不周，可能会导致稳定性差，维护成本高，影响业务上云等一系列问题。本文结合博主经验，从性能，热迁移，稳定性，应用移植四个角度对 CPU mode 进行分析。

Libvirt 主要支持三种 CPU mode：

- host-passthrough: libvirt 令 KVM 把宿主机的 CPU 指令集全部透传给虚拟机。因此虚拟机能够最大限度的使用宿主机 CPU 指令集，故性能是最好的。但是在热迁移时，它要求目的节点的 CPU 和源节点的一致。
- host-model: libvirt 根据当前宿主机 CPU 指令集从配置文件 /usr/share/libvirt/cpu_map.xml 选择一种最相配的 CPU 型号。在这种 mode 下，虚拟机的指令集往往比宿主机少，性能相对 host-passthrough 要差一点，但是热迁移时，它允许目的节点 CPU 和源节点的存在一定的差异。
- custom: 这种模式下虚拟机 CPU 指令集数最少，故性能相对最差，但是它在热迁移时跨不同型号 CPU 的能力最强。此外，custom 模式下支持用户添加额外的指令集。


### 性能

三种 mode 的性能排序如下：

```
host-passthrough > host-model > custom
```

但是它们的差距到底是多少呢，[CERN](http://openstack-in-production.blogspot.com/2015/08/cpu-model-selection-for-high-throughput.html) 根据 HEPSpec06 测试标准给出了如下性能数据。

| host-passthrough | host-model | custom |
---- | --- | ---- 
100% | 95.84% | 94.73%

从上可以总结出：

- 这三种 CPU mode 的性能差距较小
- 除非某些应用对某些特殊指令集有需求，否则不太建议用 host-passthrough，原因请见后续分析。


### 热迁移

从理论上说：

- host-passthrough: 要求源节点和目的节点的指令集完全一致
- host-model: 允许源节点和目的节点的指令集存在轻微差异
- custom: 允许源节点和目的节点指令集存在较大差异

故热迁移通用性如下：

```
custom > host-model > host-passthrough
```

从实际情况来看，公司不同时间采购的 CPU 型号可能不相同；不同业务对 CPU 型号的要求也有差异。虽然互联网多采用 intel E5 系列的 CPU，但是该系列的 CPU 也有多种型号，常见的有 Xeon，Haswell，IvyBridge，SandyBridge 等等。即使是 host-model，在这些不同型号的 CPU 之间热迁移虚拟机也可能失败。所以从热迁移的角度，在选择 host-mode 时：

- 需要充分考虑既有宿主机类型，以后采购扩容时，也需要考虑相同问题
- 除非不存在热迁移的场景，否则不应用选择 host-passthrough
- host-model 下不同型号的 CPU 最好能以 aggregate hosts 划分，在迁移时可以使用 aggregate filter 来匹配相同型号的物理机
- 如果 CPU 型号过多，且不便用 aggregate hosts 划分，建议使用 custom mode


###稳定性

OpenStack 早期默认的 mode 是 none，即由 hyperviosr 选择，基本上就是 custom。后面来 host-model 成为其默认的值。从使用经验来看，host-model 和 custom 模式下的虚拟机运行稳定，而  host-passthrough 则问题比较大，特别是在 centos6 内核下，常常出现宿主机 kernel panic 问题，如：

- [Redhat-6.4_64bit-guest kernel panic with cpu-passthrough and guest numa](https://bugzilla.redhat.com/show_bug.cgi?id=1169577)

所以从稳定性出发：

- 2.6 内核及更早内核版本避免使用 host-passthrough
- custom／host-model 比较稳定

###应用移植

对应用的影响主要体现在编译型应用，如 C，C++，Golang。在物理机上编译好的二进制应用，直接移植到 custom mode 的虚拟机有可能出现 illegal instruction。其中最常见的 SSE4 类型指令集异常，因为 custom 模式下没有 SSE4 指令集，而在物理机或者其它 mode 的虚拟机是有该指令集的。从个人经验来看

- host-model 能够平滑移植绝大部分编译型二进制文件。
- custom 下的虚拟机如果出现 illegal instruction，在该虚拟机重新编译（有时需要修改编译参数）应用后，一般能正常运行。
- 如果公司存在大量编译型应用，host-model 能让业务上云更平滑些
