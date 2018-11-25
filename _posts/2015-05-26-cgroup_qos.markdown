---
layout: post
title:  "Cgroup 与虚拟机的 QoS"
categories: OpenStack
---


[Linux Control Group 简介](http://wsfdl.com/linux/2015/05/21/%E7%90%86%E8%A7%A3control_group.html) 一文介绍了如何使用 cgroup 限制进程的资源：cpu，memory，block IO 和 network IO。在宿主机看来，基于 qemu-kvm 的虚拟机就是一个进程，为了避免某些虚拟机占用过多的资源而影响了其它的虚拟机，业界常用 cgroup 限制虚拟机占用的资源。libvirt 是管理虚拟机的一个重要库，它同样依赖 cgroup 限制虚拟机的资源，本文介绍如何使用 libvirt 限制虚拟机的资源。

----------

# Limit CPU

~~~ xml
<domain>
  ...
  <cputune>
    <shares>2048</shares>
    <period>1000000</period>
    <quota>-1</quota>
  </cputune>
  ...
</domain>
~~~

- shares：相对权重，share 为 2048 的虚拟机获得的 CPU 时间以比 1024 的多一倍。
- quota：取值为 [1000, 18446744073709551]，表示 CPU 的总配额 
- period：取值为 [1000, max(1000000, quota)]，表示在 quota 下，分配给虚拟机的 CPU 时间。

----------

# Limit Memory

~~~ xml
<domain>
  ...
  <memtune>
    <hard_limit unit='G'>1</hard_limit>
    <soft_limit unit='M'>128</soft_limit>
    <swap_hard_limit unit='G'>2</swap_hard_limit>
    <min_guarantee unit='bytes'>67108864</min_guarantee>
  </memtune>
  ...
</domain>
~~~

以下 4 个参数可用于限制虚拟机的内存资源：

- hard_limit：虚拟机内存的硬上限(KB)，qemu-kvm 强烈建议不设置该参数
- soft_limit：虚拟机内存的软上限(KB)，只有当宿主机内存资源不足时才会限制虚拟机的内存
- swap_hard_limit：虚拟机 swap 分区上限(KB)
- min_guarantee：虚拟机占用宿主机的最低内存(KB)，该参数仅支持 VMware ESX 和 OpenVZ

---------

# Limit Block IO


~~~ xml
<domain>
  ...
  <blkiotune>
    <device>
      <path>/dev/sda</path>
      <read_bytes_sec>10000</read_bytes_sec>
      <write_bytes_sec>10000</write_bytes_sec>
      <read_iops_sec>20000</read_iops_sec>
      <write_iops_sec>20000</write_iops_sec>
    </device>
  </blkiotune>
  ...
</domain>
~~~

以下 4 个参数可用于[限制 block IO](http://libvirt.org/formatdomain.html#elementsBlockTuning) 资源。

- read_bytes_sec：虚拟机读带宽上限(Bytes/s)
- write_bytes_sec：虚拟机写带宽上限(Bytes/s)
- read_iops_sec：虚拟机每秒读块设备次数上限
- write_iops_sec：虚拟机每秒写块设备次数上限

Note：非本地存储的虚拟机由于走的是网络 IO，无法用此方法限速，可以采用 dom.setBlockIoTune() 方法限制其 block IO。

-----------

# Limit Network IO 

~~~ xml
<domain>
  <devices>
    <interface type='network'>
      <source network='default'/>
      <target dev='vnet0'/>
      <bandwidth>
        <inbound average='1000' peak='5000' floor='200' burst='1024'/>
        <outbound average='128' peak='256' burst='256'/>
      </bandwidth>
    </interface>
  </devices>
  ...
</domain>
~~~

限制入口流量的参数有：

- average：平均带宽(KB/s)
- peak：峰值带宽(KB/s)
- burst：峰值速率时接收流量的上限(KB)
- floor：能保证的最低流量(KB)


限制出口流量的参数有：

- average：平均带宽(KB/s)
- peak：峰值带宽(KB/s)
- burst：峰值速率时发送流量的上限(KB)
