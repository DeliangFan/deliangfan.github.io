---
layout: post
title:  "[首发于 InfoQ]奔跑在 K8S 上的 Tensorflow (一) 起源"
categories: Kubernetes
---

近期在全力支撑机器学习业务，包括基于 K8S 搭建分布式训练平台，并深入到业务层面进行分布式改造。不论是平台的建设，还是业务的改造，以及多个维度的性能调优，都收获诸多心得，遂有此系列文章。

在建设分布式训练平台过程中，我和所有机器学习的用户，包括应用/图像/风控等部门等进行深入的沟通，调研他们的痛点。从中我们发现，算法业务方往往专注于模型和调参，而工程领域是他们相对薄弱的一个环节，建设一个强大的分布式平台，整合各个资源池，提供统一的机器学习框架，将大大加快训练速度，提升效率，带来更多的可能性，此外还有助于提升资源利用率。本文重点介绍这些痛点，并谈谈 K8S 是如何解决这些痛点。

## 痛点一：单机训练

单机训练是机器学习最大的痛点。以商品推荐为例，面对几亿的参数，近百亿的样本数量，即使采用 GPU 机器，也需要长达一个礼拜的训练时间；而图像业务拥有更多的参数和更复杂的模型，面对几 TB 的训练样本，单机往往需要长达近月的训练时间。

机器学习具有试错性非常强的特点，更快的训练速度可以带来更多的尝试，从而发掘更多的可能性。Tensorflow 从 0.8 版本开始支持分布式训练，至今为止，无论是高阶还是低阶的 API，对分布式训练已经有了完善的支持。同时，K8S 和 Hadoop 等具有完善的资源管理和调度功能，为 tensorflow 分布式训练奠定资源层的基础。

[Tensorflow On Yarn](https://github.com/Intel-bigdata/TensorFlowOnYARN) 和 [Tensorflow On Spark](https://github.com/yahoo/TensorFlowOnSpark) 是较早解决方案，奇虎的 [Xlearning](https://github.com/Qihoo360/XLearning) 也得到众人的青睐。而基于 K8S 的 [kubeflow](https://github.com/kubeflow/kubeflow) 正如冉冉升起的太阳，随着容器化的车轮将越走越远。

上述方案无一例外的将各个部门分散的机器纳入到统一的资源池，并提供资源管理和调度功能，让基于 excel 表的资源管理和人肉调度成为过去，致力用户专注于算法模型，而非基础设施。在几十 worker 下，无论是 CPU 还是 GPU 的分布式训练，训练速度都得到近乎线性的提升，将单机近月的训练时间大大缩短到一天以内，提升效率的同时也大大提升了资源利用率。

事实上，蘑菇街数月前将 CPU 类型的训练任务接入了 Xlearning，尝到了分布式训练的甜头，但也发现一些问题：1. 公司目前 yarn 不支持 GPU 资源，虽然近期版本已支持该特性，存在稳定性风险；2. 缺乏资源隔离和限制，同节点的任务容易出现资源冲突；3. 监控信息不完善。所以，后面就有了基于 K8S 的分布式训练平台。

## 痛点二：人肉部署

业务方反馈的第二大问题是人肉部署，特别在大家共享机器的情况下。每次跑训练任务前，用户手动登陆机器，下载代码，安装对应的 python 包，过程繁琐且容易出现包的冲突。

由于不同的训练任务对 python 的版本和依赖完全不同，比如有些模型使用 python 2.7，有些使用 python 3.3，有些使用 tensorflow 1.8，有些使用 tensorflow 1.11 等等，非常容易出现依赖包冲突问题。虽然沙箱能在一定程度上解决这问题，但是也带来了额外的管理负担。

容器技术的很好的解决了上述问题，借助文件系统良好的隔离性，各个容器的文件系统互不影响，将训练任务制作成镜像，避免了多次安装的繁琐。同时 K8S 支持生命周期管理的 API，用户可以基于 API 一键式完成训练任务的生命周期管理，避免人工 ssh 的各种繁琐，较大的提升了用户体验和效率。

## 痛点三：监控

监控也是分布式训练重要的一环，它是性能调优的重要依据。比如在 PS-Worker 的训练框架下，我们需要为每个 PS/Worker 配置合适的 GPU/CPU/Mem，并设置合适的 PS 和 Worker 数量。如果某个参数配置不当，往往容易造成性能瓶颈，影响整体资源的利用率。比如当 PS 的网络很大时，我们需要增加 PS 节点，并对大参数进行 partition；当 worker CPU 负载过高时，我们应该增加 worker 的核数。 

K8S 已拥有非常成熟监控方案，它能周期采集 CPU，内存，网络和 GPU 的监控数据，即每个 PS/Worker 的资源使用率都能得到详细的展示，为优化资源配置提供了依据。事实上，我们也是通过监控信息为不同的训练任务分配合适的资源配置，使得在训练速度和整体的吞吐率上达到一个较好的效果。
