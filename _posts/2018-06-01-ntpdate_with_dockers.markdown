---
layout: post
title:  "容器中的 ntpdate 踩坑杂记"
categories: 踩坑杂记
---

## 现象

故事发生在一个下午，业务方发现容器的系统时间异常，大幅度跳变到如 2019，2046 等各种时间。此外，部分非 K8S 的物理节点也出现数秒的时间跳变。

经查明是 NTP 服务器硬件故障，导致服务器数秒的时间跳变。首先介绍下客户端同步时间的方式，crontab 每十分钟采用 ntpdate 同步一次时间，所有物理机和容器都运行着同步时间的脚本。因容器以 privilege 模式运行，故容器可以调整时间。

那么问题来了：

- 为什么只有 K8S 计算节点出现大幅度的时间跳变？
- 为什么 K8S 计算节点日常未出现大幅度的时间跳变？

## 原因

Linux 常用同步时间的方式有两种(请注意如下“校准”和“调整”的用词之差)：

- ntpd: 以 daemon 运行，每当检测到时间差异，ntpd 通过细微的调整时钟频率来校准时间，即让时间变快或者变慢。这种调整方式柔性优雅，对应用影响小。但是时间差异较大时，调整的过程较长，特别当时间差超过 1000s，ntpd 停止校时。 
- ntpdate: 以脚本运行，每当检测到时间差异，ntpdate 直接调整时间，即让时间跃变。这种方式简单粗暴，有可能对应用造成很大影响。但它能校准任意幅度的时间差。

出于 ntpd 无法校正差异超过 1000s 的时间，公司采用 ntpdate 调正时间。ntpdate 发送 udp 报文，它通过给各个步骤打上时间戳，计算出客户端和服务端的时间差 offset，然后根据 offset 调整时间(具体原理请见参考文档)。

K8S 计算节点运行多个容器，同一节点上的容器(N 个)在相同时间点执行调时脚本，倘若此前 ntp 服务器时间跳变，这些容器都有可能发现时钟出现 offset 偏差(和脚本实际执行顺序有关)，由于每个容器都调整 offset 偏差，那么节点实际上有可能被调整了 N+1 个(N 容器 + 1 物理机) offset，结果将误差放大了 N 倍；然后下次调时，进一步将误差放大到 N * N offset，最终造成恶性循环。大致如下图：

![NTP time change](http://wsfdl.oss-cn-qingdao.aliyuncs.com/ntpdate-time-change.png)

可用如下脚本模拟多个 docker 共同调时场景，每当修改 ntp 服务器时钟，客户端有很大概率出现时间大幅度跳变。其中容器越多，跳变的概率和幅度越大。

~~~ golang
package main

import (
	"fmt"
	"os/exec"
	"time"
)

func main() {
	N := 10
	for {
		for i:=0; i<N; i++{
			go func(){
				out, err := exec.Command("/usr/sbin/ntpdate", "-u", "ntpserver").Output()
				if err == nil {
					fmt.Println(string(out))
				}
			}()
		}
		time.Sleep(5 * time.Second)
	}
}
~~~

日常调整时间时，offset 并不为 0，一般为 0.01 秒内，为什么 K8S 计算节点日常未出现大幅度的时间跳变呢？这和 ntpdate 内部机制有关，正如 update 的 man page 所说：

> Time adjustments are made by ntpdate in one of two ways.  If ntpdate determines the clock is in error more than 0.5 second it will simply step the time by calling the system settimeofday(2) routine.  If the error is less than 0.5 seconds, it will slew the time by calling the system adjtime(2) routine.  The latter technique is less disruptive and more accurate when the error is small, and works quite well when ntpdate is run by cron(8) every hour or two.

当时间误差在 0.5s 内，ntpdate 采用 adjtime() 校整时间；误差超过 0.5s 时，采用 settimeofday() 调整时间。adjtime 通过调整时钟频率来校准时间，效果和 ntpd 一样。而 settimeofday 则是直接调整系统时间，让时间跃变。即使有多个容器采用 adjtime 校准时间，从数学的角度论证，offset 最终能够收敛。

- adjtime(): If the adjustment in delta is positive, then the system clock is speeded up by some small percentage until the adjustment has been completed.  If the adjustment in delta is negative, then the clock is slowed down in a similar fashion.
- settimeofday(): The time returned by gettimeofday() is affected by discontinuous jumps in the system time

日常情况下，offset 通常在 0.01s 内，远小于 0.5s，故日常场景未出现时间跳变。如果强制 ntpdate 在任何情况下都使用 settimeofday 来调整时间，即：

```
ntpdate -b ntpserver
```

那么日常很快也会出现时间大幅度跳变。


## 教训

时钟异常对很多服务影响很大，特别是分布式服务，它们依赖时钟判断节点的存活等。

从容器的角度来说，容器最好不要开启 privilege 权限，即使要开启，一些敏感的命令应当去除，此外还需避免容器设置系统和内核等相关参数等。

从 NTP 的使用姿势来说，ntpd 可以根据多台 ntp 服务器来校准时间，单台 ntp 服务器时间异常不会影响客户端的校时的准确性。所以一个很好的解决办法是搭建多台 ntp 服务器，采用 ntpd 同步时间；此外 ntpdate 注入启动脚本，仅在系统启动的时候采用 ntpdate 同步一次时间。正如 ntpdate 所推荐的：

> The ntpdate utility can be run manually as necessary to set the host clock, or it can be run from the host startup script to set the clock at boot time. This is useful in some cases to set the clock initially before starting the NTP daemon ntpd(8).

## 参考

- [Linux System Administrators Guide: 14.6. Basic NTP configuration
](http://www.tldp.org/LDP/sag/html/basic-ntp-config.html)
- [计算机网络时间同步技术原理介绍](https://segmentfault.com/a/1190000005337116)
