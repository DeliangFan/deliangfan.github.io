---
layout: post
title:  "从 Helion OpenStack 浅谈 OpenStack 的产品化"
categories: Algorithm
---

-------------

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

Helion OpenStack 基于 OpenStack，

-------------

# 浅谈 OpenStack 的产品化

下图从 Security, High availability, Extend service, Operation 和 HPE 的角度分析 Helion 在产品化 OpenStack 所做的工作。

![Helion1](http://7xp2eu.com1.z0.glb.clouddn.com/Helion1.png)

## Security

- SSL：所有的 Public Endpoint 必须支持 HTTPS，以保证 HTTP 通信的安全。
- Keystone：
- Network：默认采用 VLAN 某型，
- AppArmor：增强宿主机系统的安全
- ArcSight：监控日志，
- Data Encryption：敏感数据加密，诸如涉及密码的配置文件

## HPE

-
-
-
-

![Helion2](http://7xp2eu.com1.z0.glb.clouddn.com/Helion2.png)