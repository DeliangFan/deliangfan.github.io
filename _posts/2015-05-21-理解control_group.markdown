---
layout: post
title:  "Linux Control Group 简介"
categories: Linux
---

[Linux Control Group](https://en.wikipedia.org/wiki/Cgroups)(简称 cgroup)是一个用于限制、统计和隔离进程的资源的特性，它于 2006 年由 Google 的两位工程师开发，之后合入 2.6.24 版本的内核。那时 docker 正在 Google 内部兴起，本人推测正是前者催生了 cgroup。本文重点介绍如何用 cgroup 限制进程的资源。

在虚拟化领域，如 qemu-kvm 和 linux container，cgroup 用常用来限制以下类型的资源：

- CPU time：进程占用的 CPU 时间
- Memory：进程占用的物理内存
- Block IO：进程访问块设备的 bandwidth 或 IOPS
- Network IO：进程访问网络的 bandwidth 或 packages 的数量

以 ubuntu 14.04 为例，安装 cgroup:

~~~ bash
$ apt-get install cgroup-bin cgroup-lite libcgroup1
~~~

安装完成后，cgroup 默认挂载在 /sys/fs/cgroup 上，该目录下共有 11 个 subsystem，关于它们的介绍请见[官网文档](https://help.ubuntu.com/lts/serverguide/cgroups-fs.html)，更为详细的介绍请见 [redhat resource management guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/pdf/Resource_Management_Guide/Red_Hat_Enterprise_Linux-6-Resource_Management_Guide-en-US.pdf)，本文将用到 blkio, cpu, memory 和 net_cls 这四个 subsystem。

~~~ bash
$ ls /sys/fs/cgroup/
blkio  cpu  cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls
net_prio  perf_event  systemd
~~~

-------------

# Limit CPU Time

~~~ c
void main(){
    unsigned int i, end;

    end = 1024 * 1024 * 1024;
    for(i = 0; i < end; )
        i ++;
}
~~~

未限制 CPU 使用率前，上述代码的执行时间为：

~~~ bash
$ time ./a.out

real	0m3.317s
user	0m3.312s
sys  	0m0.000s
~~~

我们在 /sys/fs/cgroup 下新建一个名为 cpu_limit 的 cgroup，并设置该 cgroup 下的进程只能占用单个 CPU 10% 的使用率。 

~~~ bash
# cfs_period_us 表示 CPU 总时间片段，cfs_quota_us 表示分配给该 cgroup 的时间片段。
# 10000/100000 = 10%

$ mkdir /sys/fs/cgroup/cpu_limit
$ echo 100000 > /sys/fs/cgroup/cpu_limit/cpu.cfs_period_us
$ echo 10000 > /sys/fs/cgroup/cpu_limit/cpu.cfs_quota_us
~~~

限制后上述代码的执行时间如下，约为前者的 10 倍：

~~~ bash
$ time cgexec -g cpu:cpu_limit  ./a.out

real	0m31.904s
user	0m3.192s
sys 	0m0.000s
~~~

某个时间点 top 的输出为：

~~~ bash
$ top
......
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
28280 root      20   0    4196    668    592 R  10.0  0.0   0:01.28 a.out
~~~

-------

# Limit Memory

首先在 /sys/fs/cgroup/memory 下新建一个名为 limit_memory 的 cgroup：

~~~ bash
$ mkdir /sys/fs/cgroup/memory/limit_memory
~~~

限制该 cgroup 的内存使用量为 300 MB：

~~~ bash
# 物理内存 + SWAP <= 300 MB
$ echo 314572800 > /sys/fs/cgroup/memory/limit_memory/memory.limit_in_bytes
$ echo 0 > /sys/fs/cgroup/memory/limit_memory/memory.swappiness
~~~

下面是测试代码，它分十次申请内存，每次申请 100 MB：

~~~ c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>

#define CHUNK_SIZE 1024 * 1024 * 100

void main(){
    char *p;
    int i;

    for(i = 0; i < 10; i ++)
    {
        p = malloc(sizeof(char) * CHUNK_SIZE);
        if(p == NULL){
            printf("fail to malloc!");
            return ;
        }
        memset(p, 0, CHUNK_SIZE);
        printf("malloc memory %d MB\n", (i + 1) * 100);
    }
}
~~~

执行结果如下，当进程占用的内存超过限制时，将被 kill。

~~~ bash
$ cgexec -g memory:limit_memory ./a.out
malloc memory 100 MB
malloc memory 200 MB
Killed
~~~

-------

# Limit Block IO

我们采用 blkio 限制进程访问块设备的速率，以磁盘为例，未限制前，其读的带宽为：

~~~ bash
$ dd if=in.file of=/dev/null count=1000 bs=1M
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB) copied, 1.4419 s, 727 MB/s
~~~

采用以下方式配置 cgroup，限制磁盘的读取速率为 10MB/s

~~~ bash
# 获取所读文件所在的磁盘编号，本文的编号为 252:0
$ df  -m
Filesystem                  1M-blocks  Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-root     17755  6288     10543  38% /
...
$ ls -l /dev/mapper/ubuntu--vg-root
lrwxrwxrwx 1 root root 7 Jul 10 21:20 /dev/mapper/ubuntu--vg-root -> ../dm-0
$ ls -l /dev/dm-0
brw-rw---- 1 root disk 252, 0 Jul 10 21:20 /dev/dm-0

# 在 /sys/fs/cgroup 目录下新建一个 cgroup，名为 limit_blkio
$ mkdir /sys/fs/cgroup/limit_blkio

# 设置读速率为 10MB/s，其中 252:0 表示所读文件在的磁盘
$ echo "252:0 10485760" > /sys/fs/cgroup/blkio/limit_blkio/blkio.throttle.read_bps_device
~~~

再次执行 dd，其平均读速率为 10.5MB/s。

~~~ bash
# 清楚内存的缓存数据
$ echo 3 > /proc/sys/vm/drop_caches

$ cgexec -g blkio:limit_blkio dd if=in.file of=/dev/null count=1000 bs=1M
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB) copied, 100.03 s, 10.5 MB/s
~~~

其中某个时刻 iotop 的输出如下：

~~~ bash
Total DISK READ :       9.80 M/s | Total DISK WRITE :       0.00 B/s
Actual DISK READ:       9.80 M/s | Actual DISK WRITE:       0.00 B/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
19987 be/4 root        9.80 M/s    0.00 B/s  0.00 % 96.99 % dd if=in.file of=/dev/null count=1000 bs=1M
~~~

test\_limit 目录下有多个 blkio 相关的文件，较为常用的是以下四个：

- blkio.throttle.read_bps_device：读取块设备的带宽
- blkio.throttle.read_iops_device：读取块设备的 IOPS
- blkio.throttle.write_bps_device：写块设备的带宽
- blkio.throttle.write_iops_device：写块设备的 IOPS

---------------

# Limit Network IO

未限速时，采用 scp 测试的网络速度为：

~~~ bash
$ scp test.file root@10.10.1.180:~/
in.file                                            100% 1000MB  71.4MB/s   00:14
~~~

我们用 net_cls 标记某个 cgroup 下的包，借助 [tc](http://linux.die.net/man/8/tc) 来限制被标记的包的量，从而限制网络带宽：

~~~ bash
$ mkdir /sys/fs/cgroup/net_cls/net_limit
$ echo 0x001000001 > net_cls.classid

# 采用 tc 限制 classid 为 10:1 网络带宽为 40Mbit/s
$ tc qdisc add dev eth0 root handle 10: htb
$ tc class add dev eth0 parent 10: classid 10:1 htb rate 40mbit
$ tc filter add dev eth0 parent 10: protocol ip prio 10 handle 1: cgroup
~~~

限速后，采用 scp 测试的网络速度为 3.6 MB/s，注意到 3.6 MB/s 和 40 Mbit/s(5MB/s) 有较大差距，而 IP 和 TCP 头部额外的开销(共 40 字节头部，每个包的平均大小为 1448 字节)不可能造成如此大的差距，所以本人也、对此深感疑惑，但未能查明原因。

~~~ bash
$ scp test.file root@10.10.1.180:~/
in.file                                         100% 1000MB   3.6MB/s   04:39
~~~
