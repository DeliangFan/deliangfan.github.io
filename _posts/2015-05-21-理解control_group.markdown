---
layout: post
title:  "Linux Control Group 简介"
categories: Linux
---

[Linux Control Group](https://en.wikipedia.org/wiki/Cgroups)(简称 cgroup)是一个用于限制、统计和隔离进程的资源的特性，它于 2006 年由 Google 的两位工程师开发，之后合入 2.6.24 版本的内核。那时 docker 在 Google 内部兴起，本人推测正是前者催生了 cgroup。

在虚拟化领域，如 qemu-kvm 和 linux container，cgroup 用常用来限制以下资源：

- CPU time：进程占用的 CPU 时间
- Memory：进程占用的物理内存
- Disk IO：进程访问磁盘的 bandwidth 或 IOPS
- Network IO：进程访问网络的 bandwidth 或 packages


[](https://sysadmincasts.com/episodes/14-introduction-to-linux-control-groups-cgroups)