---
layout: post
title:  "读 Google《SRE》第六章有感 ——— 告警配置的一些原则和经验"
categories: DevOps
---


在运维中，告警的重要性不言而喻，它既能对某些潜在的问题提前预警；也能对已发生的问题提供信息，便于快速定位和及时处理问题。

## 告警的维度

我们把告警分为系统告警和业务告警。

很多监控和告警系统默认都支持系统的告警，如：

- 硬件异常告警：磁盘，内存，网卡
- 系统指标维度告警：CPU／内存／磁盘／网络流量／链接数等
- 系统日志告警
- ......

业务告警需要根据自身业务需求进行配置，甚至需要做定制化，如：

- 业务进程存活告警
- 业务日志告警
- 访问时延告警
- 服务异常告警
- ......

系统告警往往很直观，现象很明显，容易定位出原因；业务告警的现象往往有有种因素引起，一般需要深入定位原因。

此外，系统异常往往会诱发业务异常，二者存在因果关系。每当收到系统告警时，我们应该能初步的判断系统异常对业务层面的影响；每当收到业务告警，我们可以结合是否有系统告警，以便快速定位问题。

## 告警配置的原则

每当告警发生时，值班同学需要暂停手头工作，查看告警。这种中断非常影响工作效率，增加研发成本，特别对正在开发调试的同学，影响很严重。所以，每当我们收到告警时，我们希望它能真实的反映出异常，即告警不能误报(对正常状态报警)；每当有异常产生时，报警应该及时发出来，即告警不能错报(错过报警)。但是误报和错报总是一对矛盾的指标。

- 从监控范围出发：为了避免误报，就必须增加更多的监控指标，但是增加监控指标后，又带来更多误报的可能性。
- 从阈值角度出发：对于阈值类监控，阈值过高，误报概率低，但是错报概率高；反之，阈值过低，误报概率低，但是错报概率高；

配置告警之初，我们应尽可能扩大监控告警覆盖面，选取保守的阈值，如此是尽可能避免错报。后续定期对告警进行统计分析，对误报的告警，该屏蔽就屏蔽，该简化就简化，这是一个相对长期的过程。结合项目经验和《Google SRE》观点，推荐如下告警设置原则：

- 真实性：告警必须反馈某个真实存在的现象，展示你的服务正在出现的问题或即将出现的问题。
- 表述详细：从内容上，告警要近可能详细的描述现象，比如服务器在某个时间点具体发生了什么异常。
- 可操作性：每当收到告警时，一般需要做出某些操作，对于某些无须做出操作的告警，最好取消。比如磁盘 IO 量瞬间很大，CPU 使用率瞬间飙高，我们往往不会做出操作，对某些业务而言，这类告警意义就不大了。可操作性原则尤为重要，遵循这条原则往往能避免很多误报。

## 虚拟化告警配置的一些经验

虚拟化提供云计算相关虚拟机和容器产品，集群规模近千个物理节点，集群有数十种管理服务。对我们而言，关心如下告警：

- 所有节点的硬件状况
- 所有节点某些系统指标，比如内存，磁盘
- 所有节点进程存活状况
- 所有节点某些特定日志
- 中心节点日志
- 服务可用性

为了保障业务稳定，我们尽可能的把监控覆盖到最大范围，但是也带来了许多误报，平均每天几十条告警非常影响值班同学的效率。因此我们定期统计和分析告警频率，对于告警频率高的应用，是问题的则把问题解决，误报的告警或优化，或取消。具体如下：

- 优化告警阈值：适当提高 内存／CPU／网络 IO 告警阈值。
- 优化日志级别：优化不合理的日志级别，把部分 ERROR 级别的日志调整为 WARNING。
- 屏蔽某些日志：对难以调整日志级别的应用，根据关键字屏蔽某些频繁的日志告警。
- 预警增强：对于某些影响业务方的操作，提供预警。
- 增强紧急预警：有些硬件故障会出现反应在 /var/log/messages 中，根据关键字匹配硬件类告警，以便及时处理。
