---
layout: post
title:  "K8S 的调度(二) 提升资源利用率"
categories: Kubernetes
---

高效的资源利用率是 K8S 的一大优势，从成本角度，节点资源应当得到充分利用，避免闲置，具体而言：

- 避免碎片。
- 各类资源得到全面的利用，避免单种类型资源限制。
- 避免大规格容器调度失败。
- 离在线混部。

回顾以往，我们曾顾虑容器相对薄弱的隔离性会导致业务之间互相影响，因此通过 label 划分大大小小的专属资源池；我们曾受 IaaS 资源规格的影响，同时顾虑碎片导致的闲置，所以固定容器规格为 4C8G, 8C16G 等；我们顾虑大规格的实例容易调度失败，故专门为大规格的实例预留资源池。后面实践证明，这些顾虑，几乎都是伪需求，却极大增加 K8S 管理的复杂度，极大的影响资源利用率。

通过实践，我们收获了一些经验，部分和 [《Large-scale cluster management at Google with Borg》](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43438.pdf) 不谋而合，显著的降低成本。本文重点分享相这些经验，它们不依赖高深的技术，绝大部分通过优化配置和简单投入即可实现。


## 经验一. 合并资源池

最早建设 PaaS 的时候，非常顾虑不同业务之间互相影响，因此通过 node label 的方式给每个业务划分了独立的资源池，这些大大小小的资源池带来三大痛点：

- 高昂的维护成本。
- 每个资源池都需要预留资源，造成大量资源闲置。
- 多数资源池容易出现 CPU 或者内存资源得不到充分利用。比如 Redis 资源池 CPU 利用率通常极其低下。

但是实践证明，docker 的隔离和稳定性满足大部分业务要求，通过逐步合并各个资源池，上述三大痛点迎刃而解。比如 hulk 属于计算密集型业务，elasticsearch 对磁盘空间要求较高，而 kvstore 对内存要求高，这些不同类型的业务混部在一起，往往能够全方位充分的利用物理机资源，通过合并资源池措施，整体资源利用率提升约 15 %。

当然，对于极少数对磁盘 IO 有很高性能要求的业务，在没有 PVC 的情况下建议为其分配独立的资源池。

## 经验二. 按需分配

受历史 IT 资源形态的影响，业务方在为容器设置规格的时候，通常会选择 2C4G,4C8G 等极具 IaaS 特色的规格。有些业务方出于友好目的，甚至将容器 CPU 和内存配比绝地的设置成和物理机一致。

但是在 PaaS 时代，容器实例的规格应该按需分配。对于监控指标完善的 PaaS 平台而言，可以根据监控数据决定资源的合理分配。其中 CPU 资源影响的是业务的性能，可以根据的性能要求来分配和调整。内存资源多少决定业务是否能够正常运行起来，由于内存的评估比较难，所以很多业务方都倾向申请更大的内存，这就容易造成内存的浪费，可以通过如下命令获取容器创建以来的内存峰值，从而督促业务方优化容器的内存配置。

```
$ cat /sys/fs/cgroup/memory/memory.max_usage_in_bytes
7032881152
```

## 经验三. 大规格实例调度

K8S 提供了丰富的调度策略，有些策略的效果是截然相反。默认的 LeastRequested 策略优先选择最空闲的节点。当大大小小规格的实例长时间运行后，小实例均衡的打散在各个节点上，容易造成大规格资源调度失败的问题。虽然可以粗暴的为大实例提供单独资源池，但是造成复杂的维护和资源闲置。

和 LeastRequested 相反的 MostRequested 调度策略优先把实例调度到符合要求但是剩余资源最小的节点上。为了避免同一个业务的实例调度到相同节点，影响高可靠性，可以为相同业务实例设置反亲和性，并赋予一个非常高的权重值，如此既提升服务的高可靠性，又解决了大规格实例调度失败的问题。

K8S 支持为 kube-scheduler 配置调度策略，设置如下参数：

- use-legacy-policy-config=true
- policy-config-file=/etc/kubernetes/scheduler-policy-config.json

其中 scheduler-policy-config.json 的部分配置如下：

```
{
    "kind": "Policy",
    "apiVersion": "v1",
    "priorities": [
        {
            "name": "SelectorSpreadPriority",
            "weight": 1
        }, {
            "name": "InterPodAffinityPriority",
            "weight": 10
        }, {
            "name": "MostRequestedPriority",
            "weight": 3
        }, {
            "name": "BalancedResourceAllocation",
            "weight": 1
        }, {
            "name": "NodePreferAvoidPodsPriority",
            "weight": 10000
        }, {
            "name": "NodeAffinityPriority",
            "weight": 1
        }, {
            "name": "TaintTolerationPriority",
            "weight": 1
        }
    ],
    ......
}
```

亲和性相关的配置文档请见官网 [Inter-pod affinity and anti-affinity
](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)，此处不再累述。

## 经验四. 离在线混部

离在线混部一直是热门的话题，避免离线业务影响在线业务是做好混部的前提，K8S 为此做了许多努力，在 K8S 中，在线业务通常为 Guarantee Pod，离线业务通常为 BestEffort 或者 Burstable Pod。

从 CPU 角度出发，[CPU Manager
](https://kubernetes.io/blog/2018/07/24/feature-highlight-cpu-manager/) 特性的合入完美的解决在线业务和离线业务 CPU 竞争的问题，通过 Cgroup CPUSet 为在线业务绑定独占的核，离线业务之间采用 Cgroup CPUShare 共享其它的核，解决了离在线业务互相影响 CPU 资源的问题。

从内存角度出发，当内存资源不足时，K8S 通过为离线业务设置较高的 oom\_score\_adj 机制优先选择杀死离线业务，优先保障在线业务的可靠性。

## 感悟

总而言之，在提升资源利用率方面，应该实事求是的为容器分配资源，然后根据业务特点、资源状况等选择合适的调度策略，让 K8S 得到全局相对优良的解。特别在云上，还可以调整底层的资源配比，让资源层适配上层的业务资源需求。
