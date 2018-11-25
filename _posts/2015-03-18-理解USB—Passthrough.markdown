---
layout: post
title:  "理解 USB Passthrough"
categories: OpenStack
---

----------------

虽然社区曾有人提议 [USB passthrough](https://wiki.openstack.org/wiki/Nova/proposal_about_usb_passthrough) blueprint，因不是常用功能，且会影响其它的 API，所以被社区否决。但是对于某些私有云，特别是 ERP  系统，的确有 USB passthrough 需求，遂把 USB 知识做较为全面的整理，以供看官们参考。

-----------------

# USB Device

什么是 [USB device](http://www.gentoo.org/doc/zh_cn/usb-guide.xml), 下图机智的小萌狗就是 USB device，你的鼠标是 USB device，键盘是 USB device，U 盘更是典型的 USB device。说了这么多例子，还是得用一个专业的名词一语概之，所谓 USB，即是 Universal Serial Bus(通用串行总线)，它是用来连接 USB device 和计算机，从而实现 USB device 和计算机之间的通讯。

![USB Device](http://wsfdl.oss-cn-qingdao.aliyuncs.com/USB%20device.jpg)

当把 U 盘插入到计算机的 USB 口时，我们便能在操作系统中找到该 USB device 并使用之。下图解释了 USB device 如何与计算机系统交互，USB device 不能直接和操作系统通信，它需要 USB controller interface 这座桥梁才能在硬件上接入到计算机上。HCD(Host Controller Device)则是则是硬件商向程序员提供的开发接口。

![Usb passthrough](http://wsfdl.oss-cn-qingdao.aliyuncs.com/usb_passthrough.jpg)

USB device 通过 USB controller 和计算机交互得遵守一套标准，这套标准我们称之为 USB 协议。常用的协议为 USB 1.1 和 USB 2.0，比如 U 盘一般为 USB 2.0，由于 USB 2.0 是向下兼容的，所以 USB 2.0 的设备也支持 USB 1.1。

~~~ bash
+-------------+------------+--------------+
| USB Version |  Max speed | Speed Version|　
+-------------+------------+--------------+
|   USB1.0    |  1.5Mbps   |  Low-Speed   |
|   USB1.1    |  12Mbps    |  Full-Speed  |
|   USB2.0    |  480Mbps   |  High-Speed  |
|   USB3.0    |  5-10Gbps  |  Super-Speed |
+-------------+------------+--------------+
~~~

注：[部分 USB 2.0 Device](http://support.hp.com/th-en/document/c00022531) 的速率为全速，比如很多 USB Key 便是全速的 USB 2.0 device。
   
----------------------

# USB Controller

不仅 USB device 得遵守 USB 相关协议，USB controller 也必须遵守该协议。也就是说，针对 USB 1.1 device，需要有对应的 USB controller (uhci) 来支持，对于 USB 2.0 device，也需要 USB controller(ehci) 来支持。USB controller 分为以下四种类型：

- OHCI (Open Host Controller Interface)，它是开放的支持 USB 1.1 和 Firewire(Apple) 的标准。和 UCHI 相比，OHCI 的硬件完成大部分工作，因此硬件设计复杂，而软件驱动设计简单。OHCI的另外一个缺点是它仅支持 32 bit，对于 64 bit 的操作系统，它需要 IOMMU 支持。
- UHCI (Universal Host Controller Interface)，是 Intel 主导的对 USB 1.x 的接口标准，与OHCI不兼容，支持全速和慢速设备。和 OHCI 相反，UHCI 硬件设计简单，因此需要相对复杂的软件驱动。UHCI 同样仅支持 32 bit，对于 64 bit 的操作系统，需要 IOMMU 支持。UHCI 的软件驱动的任务重，需要做得比较复杂，但可以使用较便宜、较简单的硬件的 USB 控制器。Intel 和 VIA 使用 UHCI，而其余的硬件提供商使用 OHCI。
- EHCI (Enhanced Host Controller Interface)，是 Intel 主导的专门用于支持 USB 2.0 的高速接口标准，对于 USB 2.0 的全速设备，EHCI 无法支持。因此，一般的 PC 机都带有两个 USB 控制器，EHCI 和 UCHI。
- xHCI (eXtensible Host Controller Interface) 是最新的专门用于支持 USB 3.0 的接口标准，它在速度，节能和虚拟化方面均有很大的提升，它支持 USB 各种速度标准。

--------------------------

# Linux 相关命令

1 . lsusb 查询所有的 USB device：

~~~ bash
$ lsusb
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd 
~~~

2 . lsusb -t 查询所有 USB device 层次信息，lsusb -v 可查询所有 USB device 详情。

~~~ bash
$ lsusb -t 
--Bus# 1 
  +---Dev# 1 Vendor 0x1d6b Product 0x0001 
  +----Dev# 2 Vendor 0x0627 Product 0x0001
~~~

3 . lspci 查询 USB controller。

~~~ bash
$ lspci |grep USB 
00:01.2 USB controller: Intel Corporation 82371SB PIIX3 USB [Natoma/Triton II] (rev 01)
~~~

4 . 更为详细的信息可以通过[查询 USB 文件](http://blog.csdn.net/gaojinshan/article/details/9787005)。

~~~ bash
$ cat  /proc/bus/usb/devices
T:  Bus=02 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=12   MxCh= 3
B:  Alloc=  0/900 us ( 0%), #Int=  0, #Iso=  0
D:  Ver= 1.10 Cls=09(hub  ) Sub=00 Prot=00 MxPS=64 #Cfgs=  1
P:  Vendor=1d6b ProdID=0001 Rev= 3.00
S:  Manufacturer=Linux 3.0.15 ohci_hcd
S:  Product=s5p OHCI
S:  SerialNumber=s5p-ohci
C:* #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=  0mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   2 Ivl=255ms

以上信息的含义如下[5]，参见：kernel\Documentation\usb\proc_usb_info.txt:
T = 总线拓扑（Topology）结构（Lev, Prnt, Port, Cnt, 等），是指USB设备和主机之间的连接方式
B = 带宽（Bandwidth）（仅用于USB主控制器）
D = 设备（Device）描述信息
P = 产品（Product）标识信息
S = 字符串（String）描述符
C = 配置（Config）描述信息 (* 表示活动配置)
I = 接口（Interface）描述信息
E = 端点（Endpoint）描述信息
一般格式：
d = 十进制数
x = 十六进制数
s = 字符串

拓扑信息
T:  Bus=dd Lev=dd Prnt=dd Port=dd Cnt=dd Dev#=ddd Spd=ddd MxCh=dd
|   |      |      |       |       |      |        |       |__最大子设备
|   |      |      |       |       |      |        |__设备速度（Mbps）
|   |      |      |       |       |      |__设备编号
|   |      |      |       |       |__这层的设备数
|   |      |      |       |__此设备的父连接器/端口
|   |      |      |__父设备号
|   |      |__此总线在拓扑结构中的层次
|   |__总线编号
|__拓扑信息标志

带宽信息
B:   Alloc=ddd/ddd us (xx%), #Int=ddd, #Iso=ddd
|    |                       |         |__同步请求编号
|    |                       |__中断请求号
|    |__分配给此总线的总带宽
|__带宽信息标志

设备描述信息和产品标识信息

D:   Ver=x.xx Cls=xx(sssss) Sub=xx Prot=xx MxPS=dd #Cfgs=dd
|    |        |             |      |       |       |__配置编号
|    |        |             |      |       |______缺省终端点的最大包尺寸
|    |        |             |      |__设备协议
|    |        |             |__设备子类型
|    |        |__设备类型
|    |__设备USB版本
|__设备信息标志编号#1

P:   Vendor=xxxx ProdID=xxxx Rev=xx.xx
|    |           |           |__产品修订号
|    |           |__产品标识编码
|    |__制造商标识编码
|__设备信息标志编号#2

串描述信息
S:   Manufacturer=ssss
|    |__设备上读出的制造商信息
|__串描述信息
S:   Product=ssss
|    |__设备上读出的产品描述信息，对于USB主控制器此字段为"USB *HCI Root Hub"
|__串描述信息
S:   SerialNumber=ssss
|    |__设备上读出的序列号，对于USB主控制器它是一个生成的字符串，表示设备标识
|__串描述信息

配置描述信息
C:   #Ifs=dd Cfg#=dd Atr=xx MPwr=dddmA
|    |       |       |      |__最大电流（mA）
|    |       |       |__属性
|    |       |__配置编号
|    |__接口数
|__配置信息标志

接口描述信息(可为多个)
I:   If#=dd Alt=dd #EPs=dd Cls=xx(sssss) Sub=xx Prot=xx Driver=ssss
|    |      |      |       |             |      |       |__驱动名
|    |      |      |       |             |      |__接口协议
|    |      |      |       |             |__接口子类
|    |      |      |       |__接口类
|    |      |      |__端点数
|    |      |__可变设置编号
|    |__接口编号
|__接口信息标志

端点描述信息
E:   Ad=xx(s) Atr=xx(ssss) MxPS=dddd Ivl=dddms
|    |        |            |         |__间隔
|    |        |            |__终端点最大包尺寸
|    |        |__属性(终端点类型)
|    |__终端点地址(I=In,O=Out)
|__终端点信息标志
~~~

-----------

# Python 相关库

在 pip 中找到两个和 usb 相关 package，分别为 pyUSB, usbid。

- usbid: 实现方式 too young, too simple，仅仅通过读取 USB 设备文件来获取 USB device 信息。
- pyUSB: 通过调用 libusb1.0.so 获取 usb device 设备信息，但是无法获取速度相关信息。查阅 - - - libusb 代码发现 libusb 支持获取速度信息，于是补充了 pyUSB 获取 usb device 速度信息 [patch](https://github.com/DeliangFan/pyusb)。

-----------

# Libvirt USB passthrough

## Libvirt USB Device

在 nova/virt/libvirt/config.py 定义 USB Device 如下：

~~~ python
class LibvirtConfigGuestHostdevUSB(LibvirtConfigGuestHostdev):
    def __init__(self, **kwargs):
        super(LibvirtConfigGuestHostdevUSB, self).\
                __init__(mode='subsystem', type='usb',
                         **kwargs)
        self.bus = None
        self.device = None
        self.startup_policy = 'optional'
        self.specify_usb_controller = False
        self.usb_controller_type = 'usb'
        self.usb_controller_bus = '0'
        self.usb_controller_port = None

    def format_dom(self):
        dev = super(LibvirtConfigGuestHostdevUSB, self).format_dom()

        source_address = etree.Element('address')
        source_address.set('bus', self.bus)
        source_address.set('device', self.device)

        source = etree.Element('source')
        source.set('startupPolicy', self.startup_policy)
        source.append(source_address)
        dev.append(source)

        if self.specify_usb_controller:
            address = etree.Element('address')
            address.set('type', self.usb_controller_type)
            if self.usb_controller_bus:
                address.set('bus', self.usb_controller_bus)
            if self.usb_controller_port:
                address.set('port', self.usb_controller_port)
            dev.append(address)

        return dev

    def parse_dom(self, xmldoc):
        childrens = super(LibvirtConfigGuestHostdevUSB,
                          self).parse_dom(xmldoc)
        for children in childrens:
            if children.tag == 'address':
                self.specify_usb_controller = True
                self.usb_controller_type = children.get('type')
                self.usb_controller_bus = children.get('bus')
                self.usb_controller_port = children.get('port')

            if children.tag == 'source':
                for sub in children.getchildren():
                    if sub.tag == 'address':
                        self.bus = sub.get('bus')
                        self.device = sub.get('device')
~~~

## Libvirt USB Controller

Libvirt 支持多种 [USB controller](http://libvirt.org/formatdomain.html#elementsControllers) 设备，下述xml 描述了两个 USB controller，分别为 ehci 和 uhci。

~~~ xml
<controller type='usb' index='0' model='ehci'/>
<controller type='usb' index='1' model='piix3-uhci'>
~~~

在 nova/virt/libvirt/config.py 定义 USB Controller 如下：

~~~ python
class LibvirtConfigGuestUSBController(LibvirtConfigGuestDevice):

    def __init__(self, **kwargs):
        super(LibvirtConfigGuestUSBController, self).__init__(
            root_name="controller",
            **kwargs)

        self.type = "usb"
        # Set to uhci controller by default.
        self.model = "piix3-uhci"
        # For uhci usb controller, default set its index to 0.
        self.index = '0'
        self.ports = None
        self.vectors = None

    def format_dom(self):
        dev = super(LibvirtConfigGuestUSBController, self).format_dom()

        dev.set("type", self.type)
        if self.model:
            dev.set("model", self.model)
        if self.index:
            dev.set("index", self.index)
        if self.ports:
            dev.set("ports", self.ports)
        if self.vectors:
            dev.set("vectors", self.vectors)

        return dev

    def parse_dom(self, xmldoc):
        super(LibvirtConfigGuestUSBController, self).parse_dom(xmldoc)
        self.type = xmldoc.get('type')
        self.index = xmldoc.get('index')
~~~


## Attach USB Device to VM

把 USB device [attach](http://libvirt.org/formatdomain.html#elementsHostDevSubsys) 到 ehci controler, port 对应 usb controler 的 index。

~~~ bash
virsh attach-device domain_id usb_ehci.xml

python
domain.attachDeviceFlags(xml, flags) 
~~~

usb_echi.xml 内容如下：

~~~ xml
<hostdev mode='subsystem' type='usb' managed='yes'>
   <source>
       <vendor id='0x0763'/>
       <product id='0x202a'/>
       <address bus='1' device='3' />
   </source>
   <address type='usb' bus='0' port='1'/>
</hostdev>
~~~

把 USB device attach 到 uhci controler：

~~~ bash
virsh attach-device domain_id usb_uhci.xml
~~~

usb_echi.xml 内容如下：

~~~ xml
<hostdev mode='subsystem' type='usb' managed='yes'>
   <source>
       <vendor id='0x0763'/>
       <product id='0x202a'/>
       <address bus='1' device='3' />
   </source>
   <address type='usb' bus='0' port='0'/>
</hostdev>
~~~

## Detach USB Device from VM

卸载 usb device：

~~~ bash
virsh detach-device domain_id usb_ehci.xml

python
domain.detachDeviceFlags(xml, flags)
~~~
