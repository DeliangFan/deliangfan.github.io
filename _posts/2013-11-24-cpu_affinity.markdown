---
layout: post
title:  "CPU Pinning 简介"
categories: OpenStack
---

![pinning](http://7xp2eu.com1.z0.glb.clouddn.com/pinned-tweet.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[原图出处](http://www.adweek.com/socialtimes/the-one-thing-missing-from-your-twitter-profile-strategy-pinned-tweets/625869)

# CPU Pinning Overview

[CPU pinning](https://en.wikipedia.org/wiki/Processor_affinity) 又称 Processor affinity，指将进程和某个或者某几个 CPU 关联绑定，绑定后的进程只能在所关联的 CPU 上运行。CPU 绑定有以下优点：

- 提高 Cache 命中率，提升性能
- 对于 NUMA 类型的 CPU，pinning 可要求进程仅访问就近内存，提升性能
- 避免进程相互影响，比如防止繁忙的进程影响未绑定的 CPU 上的其他进程。

Linux 从 2.5.8 开始提供 sched\_setaffinity 系统调用用于绑定进程和 CPU，详细介绍请见 [man page](http://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)：

~~~
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sched.h>

int sched_setaffinity(pid_t pid, size_t cpusetsize,
                             const cpu_set_t *mask);
~~~

我们也可以用 [tastset](http://linux.die.net/man/1/taskset) 命令来设置绑定：

~~~
$ tastset -p pid -c cpu_list

pid：即进程的 pid。
cpu_list：被绑定的 cpu list，如：0,5,7,9-11。
~~~

----------

# CPU Pinning for KVM

在宿主机看来，基于 KVM 的虚拟机实际上就是一个进程，CPU Pinning 同样可以限定虚拟机运行在宿主机上某些特定的 CPU 中，以提升虚拟机的性能。[Libvirt](https://libvirt.org/formatdomain.html#elementsCPUTuning) 对 CPU Pinning 提供了良好的支持：

~~~
<domain>
  ...
  <cputune>
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="1,2"/>
    <vcpupin vcpu="2" cpuset="2-3"/>
    <vcpupin vcpu="3" cpuset="7"/>
    ......
  </cputune>
  ...
</domain>
~~~

注意到 vcpupin 的两个参数：

- vcpu：表示虚拟机 vcpu 的 ID
- cpuset: 表示虚拟机某个 vcpu 所绑定的某些物理 cpu

所以 vcpu 0 可以运行在物理 cpu 0，vcpu 1 可以运行在物理 cpu (1, 2)，vcpu 2 可以运行在物理 cpu (2, 3)，而 vcpu 3 可以运行在物理 cpu 7 上。当虚拟机启动后，采用 virsh vcpuinfo 可以查询绑定的详情。

~~~
$ # virsh vcpuinfo vm

VCPU:           0
CPU:            0
State:          running
CPU time:       10.3s
CPU Affinity:   y-------
VCPU:           1
CPU:            1
State:          running
CPU time:       17.8s
CPU Affinity:   -yy-----
VCPU:           2
CPU:            3
State:          running
CPU time:       16.5s
CPU Affinity:   --yy----
VCPU:           3
CPU:            7
State:          running
CPU time:       14.2s
CPU Affinity:   -------y
~~~

![cpu_pinning](http://7xp2eu.com1.z0.glb.clouddn.com/kvm_cpu_pinning.png)

Canoical 工程师测试了 CPU Pinning 给 KVM 虚拟机带来的性能提升，结果如下，详情请见[kvm-performance-optimization-for-ubuntu](http://www.slideshare.net/janghoonsim/kvm-performance-optimization-for-ubuntu)。

![cpu_pinning_performance](http://7xp2eu.com1.z0.glb.clouddn.com/kvm-performance-optimization-for-ubuntu.jpg)
