---
layout: post
title:  "K8S 日志搜集演进"
categories: Kubernetes
---

两年前选择胖容器建设 PaaS 平台，毫无疑问的把搜集日志的 agent 毫无疑注入到容器中，这种妥协的方案遵循了用户习惯，降低容器化难度。随着时间的推移，暴露出越来越多的问题：

- 每次升级日志 agent，需要花费巨大的代价推动所有的业务方重新制作镜像。
- 对于外部镜像，需要安装 agent 并重新制作镜像，一旦出现依赖库版本问题，将大大增加制作成本。
- 每当业务更新日志配置，需要重新制作镜像。
- 自研的 agent 占用较大的资源，某些异常甚至影响业务。

所以将收集日志的 agent 和业务容器解耦，将极大的提升灵活性、效率和稳定性。在解耦的过程中主要面临三个核心问题，笔者将结合实际经验探索这三个问题答案：

- 从功能和灵活性的角度：agent 的部署方式选择，daemonset VS sidecar?
- 从功能和成本的角度：如何选择合适的 agent，filebeat VS logstash ......？
- 从效率的角度：K8S 的 yaml 通常比较复杂，如何保障用户体验，如何保障用户的日志排查效率？

## 容器环境下的日志搜集简介

针对容器化应用，虽然 Docker 推荐的记录方式是将日志写入标准输出和标准错误流，但是从实际的场景和业内普遍的状况来看，文件日志依然是主流的日志形态，特别在多份日志场景下，统一写到标准输出容易混淆日志，增加整理日志的工作量。因此 K8S 的日志平台应当充分考虑文件日志的搜集。

容器频繁的动态变化给问题定位带来了更大的挑战，因此所收集的日志应当尽量的详细和准确，从而提升用户的排查效率：

- IP 信息：日志应当包含 IP 信息。
- Pod 信息：日志应当包含 Pod 名字信息。
- APP 信息：日志应当包含应用名字信息，应用可以是 Statefulset 或者 Deployment 等等。
- 原始文件名称：日志应当包含原始日志文件名称。
- 准确：收集后的日志必须保证顺序。

K8S 没有为日志数据提供原生的存储方案，需要集成现有的日志解决方案到 Kubernetes 集群中。就日志系统而言，一般分为日志收集、日志存储和日志展示三个模块，其中容器环境下，主要是日志收集模块(即 agent)需要和 K8S 适配，而日志存储和日志展示模块不关心应用是否运行在容器环境中。

## Agent 的部署方式 daemonset VS sidecar

Daemonset 和 sidecar 是两种常见的日志搜集原生方案，二者各有优劣，不少开源的日志组件同时支持两种方案。

Daemonset 方式下日志收集 agent 以 daemonset 部署在每一个宿主机上。Docker 默认把所有容器的标准输出和标准错误均写入到 /var/lib/docker/containers 对应的文件中，通过把宿主机的 /var/lib/docker/containers 目录挂载到 agent 容器里，实现了日志的收集。该方式完全解耦业务和日志收集 agent，应用只需要确保日志写入标准输出中，无需感知日志收集 agent。由于每台宿主机只部署了一个 agent，所以占用很少的资源。这种方式虽然简单，但是局限性也很明显，首先日志必须写到标准输出，其次需要解决多日志场景下日志混淆的问题，最后收集的日志中通常缺少容器 IP 等重要定位信息。

Sidecar 是容器中常见的模式，service mesh 就是以这种部署方式实现容器的服务治理。Sidercar 为每个业务容器分配一个日志收集容器中，通过挂载相同的日志卷来收集日志。该方式一定程度上解耦了业务和日志收集的 agent，而且具备很强的灵活性，它支持用户设置 agent 配置文件从而实现强大的日志收集功能，比如支持收集多个日志文件，支持给日志添加 IP 等信息。这种方式主要有两个缺点，一是用户需要感知日志收集的相关配置；二是每个业务容器都需要一个日志收集容器，相对 daemonset 更耗费资源。

由上可以得出，由于云原生应用通常遵循容器的规范，日志多写在标准输出，如果 K8S 中存在大量的云原生的应用，最好选择 daemonset；如果存在大量写日志到文件的应用，sidecar 更符合业务需求。如今公有云上的 serverless K8S 正走向公众的视野，但它不支持 daemonset，所以 serverless 下最好采用 sidecar 的部署方式进行搜集。公司存在大量写日志到文件的业务，同时考虑向 serverless 的演进，故选择 sidecar 部署模式。

## Agent 的选择 filebeat VS logstash VS ...

在日志搜集领域，喜欢造轮子的工程师们毫无例外地造出大大小小的轮子，从功能强大的老牌 logstash，再到小而轻的 filebeat，还有 fluentd，logagent，flume，graylog.....。这些工具的基本功能都是收集日志并发送到下游的消息队列或者 Elasticsearch，主要差异体现在日志的解析能力、插件的丰富程度、协议的丰富程度和资源要求。

关于这些工具，网上有非常丰富的介绍，例如 [filebeat vs logstash](https://logz.io/blog/filebeat-vs-logstash/)，[fluentd vs logstash](https://logz.io/blog/fluentd-logstash/) 等等，本文不再累述。从基于 sidecar 部署角度来看，日志收集工具首先要轻，占用很少的内存和 CPU 资源；从实际业务角度来看，绝大部分应用采用主流的日志库，保证了日志格式比较标准，因而在收集时无需复杂的解析；从用户体验的角度而言，搜集工具的配置应该简单明了，上手成本低，故综合考虑下选择 filebeat 作为日志收集工具。Filebeat 采用 golang 编写，安装和配置非常简单，在给定模板下，用户只需要配置收集的路径即可。经过优化配置，其运行时通常占用几十兆的内存，非常符合实际需求。

## Filebeat 的配置

Filebeat 给每条收集的日志添加一个递增的 offset 字段，这个字段确保日志在检索时不会乱序。POD\_NAME 和原始文件名称默认被 filebeat 搜集，为让日志携带 POD\_IP 和 APP 等信息，建议 POD\_IP 和 NAMESPACE 可以通过 [Downward API](https://kubernetes.io/zh/docs/tasks/inject-data-application/environment-variable-expose-pod-information/) 获取，APP 信息配置在环境变量中。

```
env:
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: APP
    value: {your_app_name}
```

filebeat flields 模块支持读取环境变量：

```
fields:
  ip: ${POD_IP}
  namespace: ${POD_NAMESPACE}
  node: ${NODE_NAME}
  app: ${APP}
```

通过把上述的配置模板化，用户只需要关心挂载的日志路径，从而提升用户的体验和效率，[具体的 yml 样例建议参考 app-log-collection](https://jimmysong.io/kubernetes-handbook/practice/app-log-collection.html)，本文不再累述。为了确保 filebeat 不会占用太多资源，可优化如下配置：

- GOMAXPROCS：filebeat 容器的 GOMAXPROCS 设置为 1。
- queue.mem.events：消息队列大小，调整为 256，避免日志带来内存爆增。
- max\_message\_bytes：单条消息大小，调整为 100000，避免日志带来内存爆增。

对 1.18 之前的 K8S 版本，如果业务容器正常退出，sidecar 容器依旧在运行，空占资源。从 1.18 版本起，只要所有的业务容器退出，就会给 sidecar 容器发送 SIGTERM 信号结束 sidecar 容器，保证资源的及时回收。