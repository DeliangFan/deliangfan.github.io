---
layout: post
title:  "LVM 扩展逻辑卷"
categories: DevOps
---

随着时间推移，Linux 磁盘积累了越来越多的文件，使用率已达到 83%，所以需要扩容。

~~~ shell
root@ubuntu:~# df -T
Filesystem                  Type     1K-blocks     Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-root ext4      18180876 14265152   2969140  83% /
none                        tmpfs            4        0         4   0% /sys/fs/cgroup
...
~~~

该 Linux 机器的存储由 LVM 管理，仅有一块磁盘 sda，该磁盘属于唯一一个名为 ubuntu-vg 的 volume group(简称  vg)，该 vg 包含两个 logic volume(简称 lv)，分别是挂载在根目录的 ubuntu-vg-root 和用作 swap 分区的 ubuntu-vg-swap_1。该 vg 已无剩余空间，如果要扩展根目录的空间，就必须先增加磁盘，再扩展 vg，最后扩展 lv 的大小。

~~~
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

首先增加一块磁盘 sdb，利用 fdisk 在该磁盘创建一个名为 sdb1 的扩展分区，大小和磁盘相当。

~~~
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

利用 pvgroup 在分区 sdb1 上创建 physical volume：

~~~
root@ubuntu:~# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created
~~~ 

把刚刚创建的 physical volume /dev/sdb1 加入到 ubuntu-vg volume group 中。

~~~
root@ubuntu:~# vgextend ubuntu-vg /dev/sdb1
 Volume group "ubuntu-vg" successfully extended
~~~

扩展后，ubuntu-vg 总容量为 39.74 GB，可用空间为 20 GB。

~~~
root@ubuntu:~# vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  VG Size               39.75 GiB
  PE Size               4.00 MiB
  Total PE              15296
  Alloc PE / Size       5053 / 19.74 GiB
  Free  PE / Size       5124 / 20.02 GiB
~~~

接下来扩展 logical volume /dev/ubuntu-vg/root，本次增加 10GB 的空间。

~~~
root@ubuntu:~# lvextend -L+10G /dev/ubuntu-vg/root
  Extending logical volume root to 27.74 GiB
  Logical volume root successfully resized
~~~

扩展了逻辑卷后，还需要更新文件系统，因为 /dev/ubuntu-vg/root 上的文件系统类型为 ext4

~~~
root@ubuntu:~# resize2fs /dev/mapper/ubuntu--vg-root
~~~

扩展后效果如下：

~~~
root@ubuntu:~# df -T
Filesystem                  Type     1K-blocks     Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-root ext4      28502156 14265156  12871324  53% /
none                        tmpfs            4        0         4   0% /sys/fs/cgroup
~~~