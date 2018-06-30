---
layout: post
title:  "K8S 的调度(一) —— 抽象优雅的 Affinity"
categories: Kubernetes
---

无论是 IaaS 还是 PaaS，在调度方面会收到非常多类似的需求，比如基于节点类型的调度，实例之间亲和性调度等等。数年前做 OpenStack 时，那时 OpenStack 的调度功能很基础，所以笔者做了不少开发，特别是亲和性调度，因对全局缺乏充分的认知，导致代码的复用性很差，用户体验也不佳。直到接触 K8S，深入了解后愈发感叹 K8S 在调度方面的抽象之优雅，实现之精美，从实际情况上看，K8S 满足我们对调度的基本需求，几乎无需做额外的开发。

Kubernetes Scheduler 根据调度算法将 pod 调度到最优的节点上，和 OpenStack 和 Mesos 等非常类似，kube-scheduler 首先过滤不符合要求的节点，然后从符合要求的节点中根据权重选出最优节点，这两个步骤在 K8S 中分别被称为 predicates 和 priorities。

Predicates 主要有以下类型：

- PodFitsResources：节点 CPU，内存资源是否充足。
- 节点是否压力大：节点 CPU，内存，磁盘资源是否存在压力。
- Volume 相关调度：节点是否支持相应的云厂商，卷的数量是否超出上限等。
- MatchNodeSelector：节点亲和性调度，即 node affinity。
- MatchInterPodAffinity：容器之间的亲和性调度，即 pod affinity。
- 其它类型性

本文主要介绍 MatchNodeSelector，MatchInterPodAffinity 这两个调度模块，即 node affinity 和 pod affinity。

## NodeAffinty

### 需求来源

源于硬件和软件层多样性，我们需要将某个 pod 调度到某些特定的节点上，例如指定机房，存储类型，网络类型等等：

- 指定机房调度：某些业务希望部署在指定的机房中。
- 专属节点资源：某些机器属于某个业务独享，只有该业务方的容器才能调度到这些节点上运行。
- 磁盘类型调度：计算节点的磁盘类型包含 ssd 和 sata，其中 sata 盘的 IO 性能较差。对于 IO 密集型 Pod，我们希望将其调度到磁盘类型为 ssd 的服务器上，对于非 IO 密集型 Pod，将其调度到磁盘类型为 sata 盘的服务器上。
- 存储类型调度：不同节点可能支持不同的持久化存储模式，例如：local/ceph/gluster 等。我们需要根据 pod 要求的存储类型将其调度到相应的节点上。
- 网络调度：根据网络信息调度到支持该网络的节点。

K8S 将这些依赖节点是否满足特定条件的调度做良好的抽象和实现，使得我们仅需要给这些节点打上相应的 label 即可完成 pod 调度，无需再做额外的代码开发。

### K8S 实现

#### nodeSelector

K8S 早期采用 nodeSelector 将 pod 调度到具有特定 label 的节点上。它的匹配规则简单，功能也相对简单，但是具有直观易用的特点。以磁盘类型调度为例，它采用 label 标记节点的磁盘类型:

```
$ kubectl lable nodes node01 disktype=ssd
```
创建 Pod 时在 nodeSelector 注明对磁盘类型 disk_type，如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: scheduler-to-ssd-node
spec:
  containers:
  - name: nginx
    image: registry.cn-beijing.aliyuncs.com/opendcp/nginx
  nodeSelector:
    disk_type: ssd

```

#### Node affinity

鉴于 nodeSelector 的功能过于简单，K8S 于 1.2 版本引入了 node affinity 功能，在 1.11 版本处于 beta 阶段。node affinity 同样采用 label 标记节点，创建 pod 时在 affinity 字段注明匹配规则，它的匹配规则丰富，使用灵活，但是用法复杂。基于官网的介绍，可以如下语句贴切的介绍 node affintiy：

>this pod should (or, in the case of anti-affinity, should not) run in the node if the node meet rule Y.
 
> Y is expressed as a LabelSelector

Y 表示 LabelSelector，它支持丰富的匹配符号，如：In, NotIn, Exists, DoesNotExist, Gt, Lt 等。Node affinity 支持两种调度模式：

- requiredDuringSchedulingIgnoredDuringExecution：一定要存在满足条件 Y 的节点，如果不存在，则 pod 创建失败，熟称 hard 模式。
- preferredDuringSchedulingIgnoredDuringExecution：优先选择满足条件 Y 的节点，如果不存在，则在其它节点中择优创建 pod，熟称 soft 模式。

以磁盘类型调度为例，它采用 label 标记节点的磁盘类型:

```
$ kubectl lable nodes node01 disktype=ssd
```
创建 Pod 时在 affinity 注明对磁盘类型 disk_type，如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: scheduler-to-ssd
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: scheduler-to-ssd
    image: registry.cn-beijing.aliyuncs.com/opendcp/nginx
```

## PodAffinity

### 需求来源

我们常常会收到亲和性／反亲和性相关的需求。亲和性多用于实现业务的就近部署，减少网络，降低延时；反亲和性多用于故障容灾，从多个维度分散实例，尽可能降低故障影响的同类业务实例数量，特别是数据存储类的业务，它们对反亲和性的需求往往很强烈。

从硬件资源拓扑角度来看，每个机架(rack)约有十多台的物理机和一(两)台接入交换机，这些接入交换机连接到汇聚交换机，汇聚交换机再连接到核心交换机。一般来说，汇聚交换机和核心交换机都会从硬件层面实现高可靠。


我们遇上了多种故障，最常见的是主机硬件故障，此外还有四子星机器电源故障，机柜电源故障，接入交换机故障等等。并非所有的机柜都做到双电源，并非所有的接入交换机都有冗余。所以机柜电源故障和接入交换机故障往往会影响整个机柜的实例。所以一般的业务方要求实例部署在不同的机器，特殊的业务方，如 DB 等等，需要将相同 DB 的实例分布在不同的机柜。此外因链路割接，故障演练等因素，某些业务还需要在有异地机房冗余。


### K8S 实现

Pod affinity 功能于 1.4 版本引入，在 1.5 版本处于 alpha 阶段，在 1.11 版本依旧处于 beta 版本。关于 pod affinity/anti-affinity，官网用如下语句恰当的表达了其功能：

>this pod should (or, in the case of anti-affinity, should not) run in an X if that X is already running one or more pods that meet rule Y

> X is a topology domain like node, rack, cloud provider zone, cloud provider region, etc. 
> 
> Y is expressed as a LabelSelector

笔者认为上述表达非常到位，pod affinity/anti-affinty 的抽象非常优雅，功能强大。K8S 把 node, rack, zone, region 等多种拓扑层次进行抽象成了 topology domain，使得我们通过简单的配置即可实现节点／机柜／可用域／地区的亲和性或者反亲和性，无需额外代码开发。K8S 默认支持如下 topology domain。

- kubernetes.io/hostname
- failure-domain.beta.kubernetes.io/zone
- failure-domain.beta.kubernetes.io/region

用户可以方便的定义自己的 topology domain，以 rack 为例，首先给所有节点打上 rack 相关的 label 信息，如 rack=Rack01；然后在 pod 的 spec 中把 topologyKey 设置为 rack 即可。LabelSelector 功能与用法和上节 node affinity 中的一样，此处不在累述。Pod affinity 支持两种调度模式：

- requiredDuringSchedulingIgnoredDuringExecution：一定要存在满足条件 Y 的节点，如果不存在，则 Pod 创建失败，熟称 hard 模式。
- preferredDuringSchedulingIgnoredDuringExecution：优先选择满足条件 Y 的节点，如果不存在，则在其它节点中择优创建 Pod，熟称 soft 模式。

如下表示 3 redis 必须分布在不同的节点。 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-server
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 3
  template:
    metadata:
      labels:
        app: redis
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - redis
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
...
```

最后谈谈 pod affinity 的性能问题，对于 1.11 之前版本，亲和性调度过程中会查询所有的 pods，所以对于上千节点的大集群，调度一个 pod 多者需要数十秒的时间，严重影响用户体验。2018 年 4 月合入如下的 patch 极大的提升了调度效率，降低计算的复杂度，最终使得在一般情况下计算的复杂度和节点数量呈现线性关系。

[Improve performance of affinity/anti-affinity predicate by 20x in large clusters](https://github.com/kubernetes/kubernetes/pull/62211)