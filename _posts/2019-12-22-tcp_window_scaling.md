---
layout: post
title:  "话说 TCP 参数 net.ipv4.tcp_window_scaling"
categories: Linux
---

今天想谈谈一个 TCP 的系统参数，主要是基于两个背景，第一个是最近十年网络带宽基本保持指数级别增长，万兆网卡已是标准配置；第二是同城多机房是容灾的常见手段，当前跨机房的时延 RTT 通常在 1-10ms 左右；这个系统参数叫做 net.ipv4.tcp\_window\_scaling，linux man page 介绍如下：


> tcp\_window\_scaling (Boolean; default: enabled; since Linux 2.2)

>     Enable RFC 1323 TCP window scaling.  This feature allows the
      use of a large window (> 64 kB) on a TCP connection, should
      the other end support it.  Normally, the 16 bit window length
      field in the TCP header limits the window size to less than
      64 kB.  If larger windows are desired, applications can
      increase the size of their socket buffers and the window
      scaling option will be employed.  If tcp_window_scaling is
      disabled, TCP will not negotiate the use of window scaling
      with the other end during connection setup.


简要而言，该参数为 0 时，TCP 滑动窗口最大值为 (2^16)64KB；将其值设置为 1 时，滑动窗口的最大值可达 (2^30)1GB，大大提升[长肥管道(Long Fat Networks)](https://en.wikipedia.org/wiki/Bandwidth-delay_product)下的 TCP 传输速度。

首先介绍下什么是长肥管道，长肥体现在带宽(bandwidth)和延时(RTT)乘积很大，
[TCP Extensions for Long-Delay Paths, RFC 1072](https://tools.ietf.org/html/rfc1072) 定义 Capacity 大于 10^5 bit 即为长肥管道。用公式表示为：

```
capacity = bandwidth * RTT
```

在长肥管道下，如果 net.ipv4.tcp\_window\_scaling 设置为 0，意味着任意时刻该管道内最多只能容纳 64KB 的数据，单条 TCP 链接中带宽的最大利用率为：

```
util = 64KB / capacity
```

故单条 TCP 实际最大的传输速度为：

```
# 网络带宽通常用 bit 表示，乘以 8 表示将 byte 转化为 bit。
rate = util * bandwidth = 64KB / RTT * 8
```

回到开篇所述的二个背景，根据经验，公有云服务商在同城一般会分可用区，各个可用区分别代表不同的机房，同城机房之间的 RTT 通常在 0.1-2 ms 左右；对于自建的机房，通过专线连接的 RTT 一般在 1-3 ms 左右；而同城不同云服务商之间的专线连接的 RTT 通常在 3-10 ms 左右。如果 net.ipv4.tcp\_window\_scaling 设置为 0，根据上述公司可以估算出单条 TCP 连接的最大传输速度为：

- 公有云同城跨机房：250 Mb/s - 5000 Mb/s
- 自建同城跨机房：170Mb/s - 512 Mb/s
- 不同公有云同城跨机房：51Mb/s - 170Mb/s

在网络诞生之初，其传输速率为 KB 级别，随着技术的发展，网络的性能越来越好，IETF 也认识到这个问题，于是在 1992 年在 [TCP Extensions for High Performance, RFC 1323 ](https://tools.ietf.org/html/rfc1323#page-8) 提议增加该 TCP 参数。截至 2019 年，万兆网卡(10Gb/s)已是服务器的标配，对于机器学习和大数据等高性能机器更是可达 100Gb/s，如果该参数设置不合理，将大大的影响 RTT 较高的 TCP 传输速度，最终影响网络性能。很遗憾的是，公司不同部门的服务器，该参数设置不一；同样的现象也出现在不同的云服务商，并且因该参数多次引起性能问题，遂做此文。


 

