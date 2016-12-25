---
layout: post
title:  "LVM 扩展逻辑卷"
categories: DevOps
---

# Overview

对于早期 Linux 用户，如何准确地评估分区的大小以及分配合适的空间一直是安装系统时的难题，因为在普通磁盘分区的管理下，分区划分后无法在不影响数据和系统运行的条件下改变其大小，当某个分区的存储空间消耗殆尽时，往往需要更换磁盘并迁移数据，如此可能会影响业务的正常运行。

[LVM(Logical Volume Management)](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) 的出现完美的解决上述问题，它是建立在硬盘和文件系统之间的一个逻辑层，支持动态的调整文件系统的大小，增添硬盘等，提高了存储空间管理的灵活性。LVM 大有先破后立之感，它把多块硬盘组成一个资源池，再按需从资源池划分虚拟分区，并且可以动态调整虚拟分区的大小和数量，它有以下三个重要概念。

- [Physical Volume(PV)](https://www.centos.org/docs/5/html/Cluster_Logical_Volume_Manager/physical_volumes.html)：建立在物理分区之上，每个分区可以创建一个 PV。
- [Volume Group(VG)](https://www.centos.org/docs/5/html/Cluster_Logical_Volume_Manager/volume_group_overview.html)：建立在 PV 之上，由一个或者多个 PV 组成，每个 PV 只能加入到一个 VG 中。
- [Logical Volume(LV)](https://www.centos.org/docs/5/html/Cluster_Logical_Volume_Manager/lv_overview.html)：它是从 VG 划分的一块虚拟分区，可动态调整大小，一个 VG 可划分出多个 LV，这些 LV 可被格式化成文件系统并挂载在相关目录下。

~~~
                   +-----------+ +------------+ +------+
File System        |  / (ext4) | | /var (xfs) | |  ... |
                   +-----------+ +------------+ +------+
                -------------------------------------------
                   +-----------+ +------------+ +------+
Logical Volume     |  lv-root  | |   lv-var   | |  ... |
                   +-----------+ +------------+ +------+
                -------------------------------------------
                   +-----------------------------------+
Volume Group       |                 vg                |
                   +-----------------------------------+
                ------------------------------------------- 
                   +---------------+   +---------------+
Physical Volume    |      pvsda1   |   |    pvsdb1     |
                   +---------------+   +---------------+
                -------------------------------------------
                   +---------------+   +---------------+
Disk partitions    |      sda1     |   |      sdb1     |
                   +---------------+   +---------------+
                -------------------------------------------
                   +---------------+   +---------------+
Disk               |      sda      |   |      sdb      |
                   +---------------+   +---------------+
~~~             

-------------------------

# Guide

以本人 Linux 机器为例，根目录的空间使用率已达到 83%，扩容步骤如下。

~~~ shell
root@ubuntu:~# df -T
Filesystem                  Type     1K-blocks     Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-root ext4      18180876 14265152   2969140  83% /
none                        tmpfs            4        0         4   0% /sys/fs/cgroup
...
~~~

它仅有一块磁盘 sda，分区 sda5 是最主要的分区，占据绝大多数存储空间，该分区属于一个名为 ubuntu-vg 的 VG，该 VG 包含两个 LV，分别是挂载在根目录的 ubuntu-vg-root 和用作 swap 的 ubuntu-vg-swap_1。该 VG 已无剩余空间，如果要扩展根目录的空间，就必须先增加磁盘，再扩展 ubuntu-vg VG，最后扩展 ubuntu-vg-root LV。

~~~ shell
root@ubuntu:~# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda5
  VG Name               ubuntu-vg
  PV Size               19.76 GiB / not usable 2.00 MiB
  
root@ubuntu:~# vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  ...
  Alloc PE / Size       5053 / 19.74 GiB
  Free  PE / Size       5 / 20.00 MiB

root@ubuntu:~# lvdisplay
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/root
  VG Name                ubuntu-vg
  LV Size                17.74 GiB
  ...

  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/swap_1
  VG Name                ubuntu-vg
  LV Size                2.00 GiB
  ...
~~~

首先增加一块磁盘 sdb，利用 [fdisk](https://linux.die.net/man/8/fdisk) 在该磁盘创建一个名为 sdb1 的扩展分区，分区的大小和磁盘相当。

~~~ shell
root@ubuntu:~# fdisk /dev/sdb
...
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-41943039, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039):
Using default value 41943039

Command (m for help): t
Selected partition 1
Hex code (type L to list codes): 8e
Changed system type of partition 1 to 8e (Linux LVM)

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
~~~

利用 [pvcreate](https://linux.die.net/man/8/pvcreate) 在分区 sdb1 上创建一个名为 sdb1 的 PV：

~~~ shell
root@ubuntu:~# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created
~~~ 

把刚刚创建的 sdb1 PV 加入到 VG 中：

~~~ shell
root@ubuntu:~# vgextend ubuntu-vg /dev/sdb1
 Volume group "ubuntu-vg" successfully extended
~~~

这时 ubuntu-vg 总容量为 39.74 GB，可用空间为 20 GB：

~~~ shell
root@ubuntu:~# vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  VG Size               39.75 GiB
  PE Size               4.00 MiB
  Total PE              15296
  Alloc PE / Size       5053 / 19.74 GiB
  Free  PE / Size       5124 / 20.02 GiB
~~~

接下来扩展 /dev/ubuntu-vg/root LV，本次增加 10GB 的空间。

~~~ shell
root@ubuntu:~# lvextend -L+10G /dev/ubuntu-vg/root
  Extending logical volume root to 27.74 GiB
  Logical volume root successfully resized
~~~

扩展了逻辑卷后，还需要更新文件系统，因为 /dev/ubuntu-vg/root 上的文件系统类型为 ext4，所以 resize2fs 命令扩展。

~~~ shell
root@ubuntu:~# resize2fs /dev/mapper/ubuntu--vg-root
~~~

扩展后效果如下：

~~~ shell
root@ubuntu:~# df -T
Filesystem                  Type     1K-blocks     Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-root ext4      28502156 14265156  12871324  53% /
none                        tmpfs            4        0         4   0% /sys/fs/cgroup
~~~