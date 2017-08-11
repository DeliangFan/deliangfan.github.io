---
layout: post
title:  "Cinder QoS 简介"
categories: OpenStack
---

本文主要介绍如何基于 Cinder 设置数据盘的 QoS，它支持如下参数

- total\_bytes\_sec: the total allowed bandwidth for the guest per second
- read\_bytes\_sec: sequential read limitation
- write\_bytes\_sec: sequential write limitation
- total\_iops\_sec: the total allowed IOPS for the guest per second
- read\_iops\_sec: random read limitation
- write\_iops\_sec: random write limitation


Cinder 支持 front-end 端和 back-end 端设置 QoS，其中 front-end 表示 hypervisor 端，即在宿主机上设置虚拟机的 QoS，通常使用 cgroup 或者 qemu-iothrottling；back-end 端指在存储设备上设置 QoS，该功能需要存储设备的支持。


Ceph RBD 不支持 QoS，故数据盘的 QoS 需要采用 qemu io throttling 在 front-end 端设置。以 Ceph-RBD 的存储为例，设置数据盘的 QoS 步骤如下： 

1 . 创建一个 volume type，名为 cep-ssd

```
$ cinder type-create ceph-ssd --is-public true
+--------------------------------------+----------+-------------+-----------+
|                  ID                  |   Name   | Description | Is_Public |
+--------------------------------------+----------+-------------+-----------+
| a7635028-3188-4c49-99c3-3620eae97ecb | ceph-ssd |      -      |    True   |
+--------------------------------------+----------+-------------+-----------+
```

2 . 创建 QoS spec，consumer 的合法值为 front-end、back-end、both。front-end 表示使用前端控制（hypervisor 控制，会在 libvirt xml 文件中定义）, 而 back-end 表示使用后端控制(cinder drivers，需要driver 支持)，both 表示前后端同时进行 QoS 控制。因为 Ceph RBD 不支持 QoS，所以需要在 hypervisor 端控制，故 consumer 应为 front-end。

```
$ cinder qos-create ceph-ssd-qos consumer=front-end read_bytes_sec=10000000 write_bytes_sec=10000000 read_iops_sec=100 write_iops_sec=100
+--------------------------------------+--------------+-----------+------------------------------------------------------------------------------------------------------------------------+
|                  ID                  |     Name     |  Consumer |                                                         specs                                                          |
+--------------------------------------+--------------+-----------+------------------------------------------------------------------------------------------------------------------------+
| fe3fbed7-faf3-4469-aef2-26f50af93a3f | ceph-ssd-qos | front-end | {u'read_bytes_sec': u'10000000', u'write_iops_sec': u'100', u'write_bytes_sec': u'10000000', u'read_iops_sec': u'100'} |
+--------------------------------------+--------------+-----------+------------------------------------------------------------------------------------------------------------------------+
```

3 . 关联 QoS spec 和 volume type

```
$ cinder qos-associate fe3fbed7-faf3-4469-aef2-26f50af93a3f a7635028-3188-4c49-99c3-3620eae97ecb
```

4 . 之后，凡是指定 volume type 为 ceph-ssd 创建的卷都支持 QoS。

```
$ cinder  create 50 --name test --volume-type ceph-ssd
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|              attachments              |                  []                  |
|           availability_zone           |                 nova                 |
|               created_at              |      2017-08-03T06:50:58.000000      |
|                   id                  | f77b6a6e-1b56-4cd2-849d-7f8e6763dcf2 |
|                 status                |               creating               |
|              volume_type              |               ceph-ssd               |
......
+---------------------------------------+--------------------------------------+
```

5 . 把该卷挂在到一个虚拟机上。

```
$ nova volume-attach vm f77b6a6e-1b56-4cd2-849d-7f8e6763dcf2
+----------+--------------------------------------+
| Property | Value                                |
+----------+--------------------------------------+
| device   | /dev/vdb                             |
| id       | f77b6a6e-1b56-4cd2-849d-7f8e6763dcf2 |
| serverId | 3c67bedc-e0af-429b-9d45-d65779c52e15 |
| volumeId | f77b6a6e-1b56-4cd2-849d-7f8e6763dcf2 |
+----------+--------------------------------------+
```

6 . 查看虚拟机相应的 xml，数据盘已设置上 QoS。 

```
$ virsh dumpxml 13
......
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='none'/>
      <auth username='compute'>
        <secret type='ceph' uuid='5b67401f-xxxx-xxxx-xxxx-xxxxxxxxxx'/>
      </auth>
      <source protocol='rbd' name='volumes/volume-f77b6a6e-1b56-4cd2-849d-7f8e6763dcf2'>
        <host name='10.17.32.10' port='6789'/>
        <host name='10.17.32.11' port='6789'/>
        <host name='10.17.32.12' port='6789'/>
      </source>
      <backingStore/>
      <target dev='vdb' bus='virtio'/>
      <iotune>
        <read_bytes_sec>10000000</read_bytes_sec>
        <write_bytes_sec>10000000</write_bytes_sec>
        <read_iops_sec>100</read_iops_sec>
        <write_iops_sec>100</write_iops_sec>
      </iotune>
      <serial>f77b6a6e-1b56-4cd2-849d-7f8e6763dcf2</serial>
      <alias name='virtio-disk1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </disk>
......
```
