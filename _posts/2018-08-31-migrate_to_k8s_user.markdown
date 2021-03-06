---
layout: post
title:  "谈谈业务容器化 (一) 选对业务方"
categories: Kubernetes
---

今年推动业务容器化，接触多种业务方，不同业务方对容器化的接纳程度存在较大差异。此外，相比虚拟机，推广容器的复杂程度和困难程度远远超过虚拟机。期间种种，特别对于业务方的选择，以及如何降低接入成本，颇有感触。本文介绍业务方的选择，即哪些业务更适合容器化，下文介绍如何降低容器化成本。

## Docker 和 K8S 带来了什么

Docker 和 K8S 作为 PaaS 核心模块，功能强大，比如 Docker 具有隔离，高性能，秒级启动等优势；K8S 具有 Service，RS，Deployment 等高级功能。但是从笔者所了解的用户需求来看，业务方更青睐如下特性：

- Docker 提供良好的隔离性
- Docker 定义了应用交付和运行的标准，为 Ops 带来更多可能性
- K8S 资源管理和资源调度
- K8S 为应用的生命周期管理提供 API 化
- 提升资源利用率
- 其它高级功能如服务发现，RC，Deployment 等高级功能......

Docker 良好的隔离性和标准化的运行使得应用的生命周期管理 API 化成为可能，业务方可以调用 PaaS 的 API 在分钟级别的时间内快速创建大量实例，极大的提升了运维效率。其次，标准化后，为 devops 带来了更多的可能性。

K8S 也是一套优秀的资源管理和资源调度平台，它统计内存，CPU，GPU 等的使用率，为调度提供良好的决策基础。K8S 提供了丰富的调度策略，比如资源调度，根据网络／端口／label 等等的调度，还有抽象非常优雅的亲和性和反亲和性调度。此外，K8S 资源管理和调度还提供了超配的功能，为提升资源利用率，离在线混部提供了良好的基础。

K8S 也提供了 Service，RC，Deployment 等资源，它们提供了服务发现，故障快速恢复，滚动升级等高级特性。实际上用户对此类的功能并非很感兴趣。以服务发现为例，如果相关需求存在，很可能已有相关人员维护类似 dubbo 的服务，替换成 K8S 的服务发现可能涉及大量改造；如果用户没有该需求，那么这个功能对用户的意义很小。

## 选对业务方

虽然容器可以支持有状态的业务，但是它的设计理念更适合无状态的业务，故建议优先考虑无状态的业务。从业务方接入的经验来看，如下三类无状态的业务接入意愿更强烈，接入的成本更低廉。

- 资源大户业务：占用大量的机器资源，具有大量实例，这些实例具有性质类似的特点。它们多为 PaaS／SaaS 平台类型服务，比如 redis，部分消息中间件，机器学习，搜索引擎等。
- Job 业务：即批处理类型业务，具有运行时间短，频繁创建删除的特点。
- 标准化业务：为了提升研发效率，公司制定应用标准，比如应用的目录组织，配置文件，启动脚本，日志文件和命名等必须满足一定规范。

资源大户业务占用大量的机器资源，以 redis 为例，当物理机达到上百台，缓存实例达到成千上万台时，依赖 excel 的人肉分配非常不现实，用户需要一个资源管理和调度的工具，很显然，用户自己实现一套资源管理和调度平台并非易事，K8S 优秀的资源管理和调度功能刚好满足这类业务的需求，特别是亲和性调度。其次，这些实例缺乏隔离性，容易互相影响，而 Docker 恰好解决隔离性问题。再次，API 化的生命周期管理简化部署流程，极大的提升了交付效率。最后，容器化保证每个实例都有一个 IP，避免了端口冲突的问题，业务方无需再维护端口信息。从和用户的交流情况来看，这类用户是非常愿意拥抱 K8S 和 Docker，虽然涉及一定的业务改造，并且要学习镜像的制作和 K8S API 的使用，但是平摊到每个容器上，改造和学习成本就非常低了。

Job 业务具有运行时间短，频繁删除的特点。对此，基于脚本化的生命周期管理显得非常繁杂，API 化的生命周期管理功能对这类业务方无疑非常有吸引力。再次，Job 业务也有资源管理和调度的需求，特别对弹性有独特的需求，而这些需求 K8S 都能满足。从资源利用率层面出发，批处理任务多为计算密集型，引入 Job 业务有利于提升资源的混部，从而提升资源利用率。故 Job 的业务方对 K8S 和 Docker 的接纳程度也很高。

标准化业务的应用结构和运行时遵循一定的标准，标准化大大的降低了容器化的成本。因为标准化业务的项目目录，配置文件，二进制和库都遵循一定的规格。所以可以根据这些标准开发自动生成 Dockerfile 并制作镜像的功能，用户几乎不用感知镜像的存在；同样，我们可以根据这些标准生成 Statefulset/ReplicationSet 参数配置，然后调用 K8S 的 API 完成容器的创建，特别对于接入发布系统的应用，几乎不感知 K8S 的存在。平心而论，从业务方的角度出发，容器化或许并不能给他们带来特别明显的优势，但是因为标准化业务的接入成本较低，绝大部分业务方还是能够接受。

此外，Docker 定义了应用交付和运行的标准，这些标准给 Ops 带来更多的可能性。传统上，CI／CD 的工具非常之多，这些工具的用法也比较复杂，比如 CI 有 jenkins 和 travis CI 等工具，以 jenkins 为例，如果有多个用户使用，那么可能存在版本和库冲突的问题，同时 jenkins 的配置也比较复杂，容器化简化了 jenkins 的配置，同时避免用户之间的互相干扰。再比如说运维工具，常见的有 ansible，puppet，chef 等等，K8S 和 Docker 的部分职责和这些工具重叠，容器化后，用户完全可以不再需要这些工具。笔者认为，容器化给 Ops 简化了流程，提升了效率，二者存在很大的相互促进空间。

## 写在最后

几年前，那时 IaaS 还如日中天，大量的业务从物理机迁移到虚拟机，除了个别业务对性能有特殊要求外，绝大部分业务都顺利的接入虚拟机，业务方完全不感知虚拟机和物理机的差异；对 IaaS 团队来说，几乎无需关注业务的特点。

今年，当接触多个业务方，接入上千个容器后，深感不同的业务方对 PaaS 看法存在非常大的差异，接入的成本也有天壤之别。在推广容器化的过程中，应当首先考虑资源大户和 Job 业务，其次是标准化的业务。鉴于 PaaS 在提升资源利用率上有着天然的优势，可以在资源利用率做出显著的成果，节省成本，得到上层的认可，再由上层推动其它业务方做容器化。另外一方面，PaaS 也应该从多个维度去降低接入成本，加快容器化的推广速率。
