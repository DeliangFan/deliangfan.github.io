---
layout: post
title:  "缩减镜像 virtual size 记"
categories: OpenStack
---

公司有两个私有镜像，分别为 CentOS6 和 CentOS7，这两个镜像源远流长，缺乏完整的变更记录，源镜像也寻而不得。两个镜像目前的 virtual size 均被设置为 80G，当要创建磁盘为 40G 的虚拟机时，因 virtual size 大于虚拟机实际规格，虚拟机肯定是创建失败。所以需要把这两个镜像的 virtual size 调整为 40G。

```
$ qemu-img info centos6.qcow
image: centos6.qcow
file format: qcow2
virtual size: 80G (85899345920 bytes)
disk size: 2.0G
.....

$ qemu-img info centos7.qcow2
image: centos7.qcow2
file format: qcow2
virtual size: 80G (85899345920 bytes)
disk size: 1.2G
......
```


在调整 virtual size 之前，需要缩减镜像内部的文件系统为 40G，然后才能缩减镜像的 virtaul size，否则会坏内部数据。CentOS6 镜像内部的文件系统为 ext4，CentOS7 内部的文件系统为 xfs，ext4 文件支持缩减功能，而 xfs 不支持缩减功能，故二者处理的方式不同，本文介绍这两个镜像的缩减过程。


```
$ virt-filesystems -a centos6.qcow2 --all --long
Name       Type        VFS   Label  MBR  Size         Parent
/dev/sda1  filesystem  ext4  -      -    85898297344  -
/dev/sda1  partition   -     -      83   85898297344  /dev/sda
/dev/sda   device      -     -      -    85899345920  -

$ virt-filesystems -a centos7.qcow2 --all --long
Name       Type        VFS  Label  MBR  Size         Parent
/dev/sda1  filesystem  xfs  -      -    85898297344  -
/dev/sda1  partition   -    -      83   85898297344  /dev/sda
/dev/sda   device      -    -      -    85899345920  -
```

## CentOS 6

安装必要工具 libguestfs-tools。

```
$ yum install libguestfs-tools
```

执行如下命令进入 rescue 模式。

```
virt-rescue -a centos6.base
```

登陆 rescue 后，先后执行 e2fsck，resize2fs 等命令，其中 e2fsck 检查文件系统是否有损坏，resize2fs 则缩减文件系统大小，注意将文件系统设置为略小于 40G。

```
><rescue> e2fsck -f /dev/sda1
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
......
/dev/sda1: 56324/5242880 files (0.2% non-contiguous), 730507/20971264 blocks

><rescue> resize2fs /dev/sda1 39G
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/sda1 to 10223616 (4k) blocks.

><rescue> sync
><rescue> exit
```

创建一个 40G 大小的新镜像文件。

```
$ truncate -s 40G output.img
```

执行 virt-resize 命令制作新镜像。 

```
$ virt-resize centos6.qcow2 output.img --shrink /dev/sda1
Examining centos6.qcow2 ...
......
/dev/sda1: This partition will be resized from 80.0G to 40.0G.
......
Copying /dev/sda1 ...
 100% ⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧ 00:00
Expanding /dev/sda1 using the 'resize2fs' method ...

Resize operation completed with no errors......
```

最后在 output.img 中进入 rescue 模式，执行 resize2fs 将文件系统填满整个磁盘。

```
  virt-rescue -a output.img
  ><rescue> resize2fs /dev/sda1
  ><rescue> sync
  ><rescue> exit
```

制作完成后，效果如下

```
$ qemu-img info output.img
image: output.img
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 1.9G
......

$ virt-filesystems --long -h --all -a output.img
Name       Type        VFS   Label  MBR  Size  Parent
/dev/sda1  filesystem  ext4  -      -    40G   -
/dev/sda1  partition   -     -      83   40G   /dev/sda
/dev/sda   device      -     -      -    40G   -

```

## CentOS 7

CentOS7 的 xfs 无法缩减，我们可以创建一台虚拟机，然后把原镜像和新镜像以磁盘的形式挂载到该虚拟机上，分别为 vdb 和 vdc。把 vdc 磁盘格式化为 xfs，然后将 vdb 中所有的文件拷贝到 vdc。之后为 vdc 安装 grub，配置启动参数即完成新镜像的制作，具体步骤如下。

安装必要工具 virt-install。

```
$ yum install virt-install
```

将原有的 qcow2 镜像转化为 raw 格式的镜像，后面会用到。

```
qemu-img convert -f qcow2 -O raw centos7.qcow2 centos7.raw
```

创建一个 40G 大小的新镜像文件。

```
$ qemu-img create -f raw output.img 40G
```

启动一台 Linux 虚拟机。

```
$ virt-install --virt-type kvm --name build_image --vcpus 8 \
--ram 8096 --disk /opt/images/buildimage.qcow2,format=qcow2 \
--network network=default --graphics vnc,listen=0.0.0.0 \
--noautoconsole --os-type=linux --os-variant=rhel6 --import
```

将原镜像和新镜像文件以磁盘的形式挂载到虚拟机上。

```
$ virsh attach-disk build_image  --source /opt/images/centos7.raw --target  vdb --cache  none
$ virsh attach-disk build_image  --source /opt/images/output.img --target  vdc --cache  none
```

登陆到新建的虚拟机上，采用 fdisk 为 vdc 制作分区，然后格式化为 xfs 文件系统。分别将 vdb1 和 vdc1 挂在到相应目录，然后将 vdb 的所有文件拷贝到 vdc 上：

```
$ mount -t xfs /dev/vdb1 /mnt/vdb
$ mount -t xfs /dev/vdc1 /mnt/vdc
$ cp -rf /mnt/vdb/* /mnt/vdc/
```

为 vdc 安装 grub2。

```
$ grub2-install --boot-directory=/mnt/vdc/boot /dev/vdc
```

查看 vdc1 的 uuid，然后修改 vdc 中 /mnt/vdc/etc/fstab 和 /mnt/vdc/etc/grub2.cfg 文件中的设备 uuid 为 vdc1 的 uuid。

```
$ blkid /dev/vdc1
/dev/vdc1: UUID="fd7dac25-78a4-4177-bd71-149c1ad79e5f" TYPE="xfs"
```

之后 umount vdb 和 vdc。

```
$ umount /dev/vdb1
$ umount /dev/vdc1
```

在宿主机上把磁盘卸载，然后关闭虚拟机。

```
$ virsh detach-disk build_image --target vdb
$ virsh detach-disk build_image --target vdc
$ virsh destroy build_image
$ virsh undefine build_image
```

最后将 output.img 转换为 qcow2 格式，至此新镜像制作完成。

```
$ qemu-img convert -f raw -O qcow2 output.img output.qcow2
$ qemu-img info output.qcow2
image: output.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 1.3G
......

$ virt-filesystems --long -h --all -a output.qcow2
Name       Type        VFS  Label  MBR  Size  Parent
/dev/sda1  filesystem  xfs  -      -    40G   -
/dev/sda1  partition   -    -      83   40G   /dev/sda
/dev/sda   device      -    -      -    40G   -

```




