---
layout: post
title:  "从 Helion OpenStack 浅谈 OpenStack 的产品化"
categories: OpenStack
---


# Ecosystem VS Product

![](http://7xp2eu.com1.z0.glb.clouddn.com/EcosystemandProduct.png)

## OpenStack

确切的说，OpenStack 更像一个生态系统，从硬件、软件到服务，覆盖了计算、存储、网络和安全等领域，截止到 2016 Feb，OpenStack Community 共有：

- 36008 people
- 20M+ lines of code
- 177 countries
- 563 supporting companies
- 600+ project

虽然 OpenStack 提供了丰富的功能，但是很多项目的成熟度和稳定性还有待商榷，并且其安装和运维也并非易事。

## Helion

Helion OpenStack 基于 OpenStack 并对其修枝裁叶，为企业提供自助服务平台和简化管理，并帮助开发人员快速交付应用。

- 对运维而言，简化管理，更易部署和运维
- 对开发人员，提供丰富的自助服务，支持快速获取资源
- 对业务而言，更为稳定，安全可靠
- 对企业而言，提高资源利用率，节省成本，提高效率

-------------

# 浅谈 OpenStack 的产品化

下图从 Security, High availability, Extend service, Operation 和 HPE 的角度分析 Helion 在产品化 OpenStack 所做的工作。

![Helion1](http://7xp2eu.com1.z0.glb.clouddn.com/Helion1.png)

## Security

- SSL：所有的 Public Endpoint 必须支持 HTTPS，以保证 HTTP 通信的安全。
- Keystone：
- Network：默认采用 VLAN 某型，
- AppArmor：增强宿主机系统的安全
- ArcSight：日志监控与告警
- Data Encryption：敏感数据加密，诸如涉及密码的配置文件

## HPE 差异化

- Hardware：HPE 的 x86 服务器，网络设备等
- Hlinux：HPE Linux
- Storm & Vertica：HPE 大数据分析和存储软件
- VSA & 3PAR：HPE 分布式存储
- Sherpa：HPE 云生态
- HPE Portal 

## High Availability

- Controller HA：采用 Haproxy 和 IP Cluster 保证控制节点高可靠性
- Database HA：采用 Galera Cluster 保证数据库高可靠性
- RabbitMQ HA：采用 RabbitMQ Cluster 保证消息中间件的可靠性
- Network HA：采用 Neutron DVR 避免网络节点的单点故障
- Storage HA：采用 Ceph，VSA 等分布式存储软件保证存储的高可靠性
- Freezer：支持备份宿主机的文件系统
- AZ & Region：多区域，多机房部署

## Extend Service

- ELK：提供日志的搜集、存储和检索
- Mail Alarm：提供邮件通知服务

## Operation

- Package：维护上百个安装包
- Deployment：提供自动化快速部署功能
- Automation：简化运维工作

![Helion2](http://7xp2eu.com1.z0.glb.clouddn.com/Helion2.png)
