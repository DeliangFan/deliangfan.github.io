---
layout: post
title:  "Redis 和 Jaguar 迁入 K8S 记"
categories: Kubernetes
---

近期接入 K8S 的业务中，Redis 和 Jaguar 两个业务方分别遇上了不同的性能问题。本文主要介绍接入过程中遇上的性能问题和解决办法。

## Redis

### Redis 简介

蘑菇街使用开源的 [redis](https://redis.io) 做为 Key-Value 数据库，支撑线上多个业务。迁入 K8S 前，redis 部署在多台独占物理机上，每台物理机上部署多个 redis 实例，这些 redis 之间的隔离性很低，它们共享 CPU，内存，磁盘等资源。

### RTT 超时

接入 K8S 前，我们根据负载等因素把 redis 折算成不同规格的容器实例，比如负载比较低的，我们使用 CPU(requests 1, limits 2) RAM(requests 4G, limits 4G) 的规格实例，对于负载高的，相应提升规格。

当 redis 接入 K8S 后，部分 redis 客户端请求的超时概率[超过 10ms]明显上升，接近 0.1 %。例如下图：

![rtt](http://7xp2eu.com1.z0.glb.clouddn.com/rtttimeout.png)

从基础监控数据来看，容器实例的 CPU，内存，IO，TCP 重传率等都在合理的范围内。在全链路同时抓包发现，对于超时的请求，redis 容器内部网卡 response 和 request 报文的时间间隔达到数十 ms，进一步怀疑是由 redis 服务端导致的。

Redis 是内存型数据库，CPU 和内存是影响其性能的最主要因素。容器采用 CPU Shares 模式限制 CPU 资源，在 Shares 模式下，虽然容器可以调度到多个物理核上运行，但是在一个时间片段 cpu.cfs\_period\_us 内，Cgroup 限制了容器最多能使用的 CPU 时间片段 cpu.cfs\_quota\_us。直觉上，正是因为这种限制，在一个时间片段内，如果并发数瞬间较高(我们暂且称之为 burst 场景)，当消耗完分配的资源后，部分请求只有等到下个时间片段才被处理，最终导致数十 ms 的延时。虽然 Redis 容器平均 CPU 使用率低，但是在 busrt 场景下，CPU 的资源不足。此外，每 10ms 对比 /proc/{pid}/stat 的 jiffies 数也可佐证上述猜测。

### 解决方法

具体而言，就是增大 Redis 容器的 CPU limits，容器在一个时间片段内，分配到的 CPU 资源上限增加。由于 K8S 的调度是根据 requests 来的，调整 limits 参数并不会影响调度和降低容器部署密度，在当前场景下有利用充分利用资源，治标治本，操作简便。

另外，适当的调大 cpu.cfs\_period\_us 或许也能缓解超时问题。但是如果并发数继续提高，则超时的 RTT 变的更长，超时率也会继续上升，治标不治本，且操作不便。

当增加一倍 CPU limits 后，某个业务 RTT 的超时率如下：

![RTT better](http://7xp2eu.com1.z0.glb.clouddn.com/rttbetter.png)


## Jaguar

### Jaguar 简介

Jaguar 是蘑菇街自研的静态化服务器，提供一套静态化解决方案，用于替代 Apache Traffic Server，采用 golang 编写。主要出于复用 nginx gzip 需求，jaguar 前端部署了一个七层代理 nginx。

![jaguar](http://7xp2eu.com1.z0.glb.clouddn.com/jaguar%20deployment.png)

接入 K8S 前，jaguar 和 nginx 部署在同一台 4C8G 的虚拟机上，最高性能可达 3w qps。

### 并发之痛

采用相同的规格接入 K8S 后，压测得出其最高性能只能达到 1.5w qps。压测期间，CPU 使用率达到 100%，其中 nginx 占用了大部分 CPU 资源。我们采用 iperf 发现大量的 CPU 耗费在 nginx 向 jaguar 建立 TCP 连接过程中，主要被 raw\_spin\_lock 函数占用。

![iperf](http://7xp2eu.com1.z0.glb.clouddn.com/nginx_iperf.png) 

根据上述现象，怀疑有可能是 tcp 相关参数造成的，其中一个小伙伴对比虚拟机和容器的参数后有重大发现：

- 容器：net.ipv4.ip\_local\_port\_range = 32768 61000
- 虚拟机：net.ipv4.ip\_local\_port\_range = 10023 61000

二者可用端口数相差约 1 倍，性能也相差一倍。调整端口数后，容器和性能和虚拟机性能接近。

### 解决方法

Nginx 作为七层代理，用户访问 jaguar 时，首先需要和 nginx 建立一条 TCP 链接，然后 nginx 再向 jaguar 建立一条 TCP 链接。问题就出现在 nginx 向 jaguar 建 TCP 连接上，nginx 作为客户端，需要内核分配一个端口用于建立连接，在这条 TCP 关闭之间，该端口一直占着，当并发数过大时，导致端口耗竭，最终影响了性能。

和业务方沟通后，业务方表示 nginx 可以去除，再次压测，性能可达 10w qps。

在对比虚拟机和容器参数的过程中，吸取了一个教训：容器镜像系统参数最好和虚拟机保持一致性，以免业务方在迁入过程因参数差异造成异常。
