---
layout: post
title:  "K8S 监控告警原生演进"
categories: Kubernetes
---

公司在虚拟机时代已有一套完善而强大的监控告警系统：agent 支持采集丰富的监控指标、监控日志的错误字段和进程存活并发送告警，后端提供强大的聚合方式和灵活的告警配置，前端具备强大的可视化能力。在如此优秀的监控告警系统映衬下，一切开源的方案都显得如此苍白无力，所以毫无疑问将该 agent 稍微改造即塞到胖容器中。该方案很好的适配已有的监控告警体系，但是不利于 agent 的升级维护，也不符合原生的玩法，所以对监控进行一些改造，本文主要谈谈相关经验，即 K8S 如何在原生玩法下对接已有的监控告警系统。

## 指标

首先讨论 PaaS 平台需要采集哪些指标，这些指标可以分为容器指标和宿主机指标两个层面。无论是宿主机还是容器，如下是常用的监控指标：

- CPU：usr, sys, iowait, irq, softirq, load
- GPU：显卡使用量，显存使用量
- 内存：主要为内存的真实使用
- 网络：网卡出入带宽，网卡出入包速率，丢包率等
- 磁盘：读写 iops，读写带宽，ioutil，磁盘使用率等
- 上下文切换数

### 容器监控指标

CAdvisor 内置于 Kubelet 中，默认采集该节点所有容器的一些监控指标。其中容器的 cpu 通过容器中 /sys/fs/cgroup/cpu 相关文件所得，内存指标通过容器中 /sys/fs/cgroup/memory/ 相关文件获取，磁盘指标源自 /sys/fs/cgroup/blkio/ 下相关文件，网络则是通过读取容器的 /proc/\<pid\>/net/dev 获取。具体而言，cAdvisor 支持获取如下指标：

- CPU：user, sys, load
- GPU：显卡使用量，显存使用量
- 内存：内存使用率, pagefault, pgmajfault 等
- 网络：网卡出入带宽，网卡出入包速率，丢包率，tcp 状态统计等
- 磁盘：同步和异步读写 iops，同步和异步读写带宽等

虽然 cAdvisor 缺乏 irq, iowait, 上下文切换等监控指标，但是整体比较全面，既能满足日常的监控需要，也能满足大部分问题定位场景的需求。单进程容器下只有业务进程，容器的状态代表了业务进程的状态，故业务进程的状态实质由 K8S 维护，故容器内部无需监控进程状态。此外由于把业务日志告警的功能移植到日志管理模块，所以容器中无需再运行自研的监控告警 agent。

### 宿主机监控指标

CAdvisor 还能收集宿主机基本监控指标，它们和容器的指标大同小异。但是 cAdvisor 不能监控宿主机上的进程存活状态，也不能监控宿主机上的硬件异常，不满足我们实际需求。

[node-problem-detector](https://github.com/kubernetes/node-problem-detector) 是监控宿主机异常的开源方案，它支持监控 KernelDeadlock，ABRT 等异常场景，还支持用户扩展插件，支持对接 prometheus，对于完全采用 K8S 配套开源监控体系来说，node-problem-detector 是一个很好的补充。由于自研的监控告警 agent 的功能非常强大，它能收集宿主机详细完善的指标，探测宿主机的进程存活状态，监控宿主机的硬件和内核异常事件，稳定的运行了数年，所以没有必要引入 node-problem-delector 这个轮子。

## 方案

### 开源方案

围绕 K8S 诞生了很多监控的项目，如 cAdvisor，heapster，promethues，kube-state-metrics 等，这些项目有的已经消亡，有的已成 CNFC 的成员。它们的核心功能在于数据的采集和分发，而数据的存储和展示采用的方案几乎都是监控告警领域的经典项目。

- 监控指标展示：grafana。
- 监控数据存储：influxDB, levelDB 等时间序列数据库。
- 数据采集：采用 cAdvisor 采集容器和宿主机的基础指标，node-problem-detector 监控宿主机状态，kube-state-metrics 采集 K8S 的集群资源信息。
- 数据收集聚合：heapster(已废弃), promethues 基本成本数据搜集和聚合的中心，还支持告警和展示。

下图是一种可选的基于开源的原生监控方案，可以根据自身情况决定是否需要告警模块，以及 prometheus 的存储方案。

![收集](https://wsfdl.oss-cn-qingdao.aliyuncs.com/prometheus.png)

### 落地实践

在实际的演进情况来看，考虑到现有监控体系的完备、强大和稳定，所以很大程度上沿用现有的监控体系，仅在数据的采集方面做了原生方向的演进。

首先移除了容器中的自研的监控 agent，完全依赖 cAdvisor 采集容器的监控数据。通过宿主机上自研监控 agent 周期性拉取 cAdvisor 数据，实现了容器指标数据的上报。

其次保留了宿主机上自研的监控 agent，并以 daemonset 的形式运行在各个宿主机上，便于升级和管控。该 daemonset 下的容器与宿主机共享网络，以监控宿主机的网络性；共享了 PID namespace，以便监控进程状态；还 mount 了宿主机上的日志目录，以便监控宿主机的日志异常。

![实践](https://wsfdl.oss-cn-qingdao.aliyuncs.com/monitor-k8s.png)