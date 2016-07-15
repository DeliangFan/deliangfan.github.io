 ---
layout: post
title:  "Cgroup 与虚拟机的 QoS"
categories: OpenStack
---


[Linux Control Group 简介](http://wsfdl.com/linux/2015/05/21/%E7%90%86%E8%A7%A3control_group.html) 一本介绍了如何使用 cgroup 限制进程的资源：cpu，memory，block IO 和 network IO。在宿主机看来，基于 qemu-kvm 的虚拟机就是一个进程，为了避免某些虚拟机占用过多的资源而影响了其它的虚拟机，业界常用 cgroup 来限制虚拟机占用的资源。libvirt 是管理虚拟机的一个重要库，它同样依赖 cgroup 限制虚拟机的资源，本文主要介绍如何使用 libvirt 限制虚拟机的资源。

----------

# Limit CPU

以下是

~~~
<domain>
  ...
  <cputune>
    <shares>2048</shares>
    <period>1000000</period>
    <quota>-1</quota>
    <emulator_period>1000000</emulator_period>
    <emulator_quota>-1</emulator_quota>
  </cputune>
  ...
</domain>
~~~

----------

# Limit Memory

~~~
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

- hard_limit
- soft_limit
- swap_hard_limit
- min_guarantee

---------

# Limit Block IO


~~~
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

- read_bytes_sec：虚拟机的读带宽上限
- write_bytes_sec：虚拟机的写带宽上限
- read_iops_sec：虚拟机的每秒读取次数上限
- write_iops_sec：虚拟机的每秒写次数上限

-----------

# Limit Network IO 

~~~
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