---
layout: post
title:  "理解 Copy On Write 技术"
categories: Algorithm
---

随着阅历的增长，越发感慨在千差万别的软件中总能找到某些非常相通相似的思想、技术或者算法，比如 [Copy On Write(写时复制)](https://en.wikipedia.org/wiki/Copy-on-write) 便是资源管理方面的一种优化技术，广泛的应用于软件开发(如 [Java 的 Copy On Write 容器](http://ifeve.com/java-copy-on-write/))、虚拟内存管理(如进程的 [fork](https://en.wikipedia.org/wiki/Fork_(system_call))、qemu-kvm [虚拟机镜像](https://en.wikipedia.org/wiki/Qcow)乃至 Docker 的 [AUFS 文件系统](https://en.wikipedia.org/wiki/Aufs)等等。[维基百科](https://en.wikipedia.org/wiki/Copy-on-write)的定义如下：

> Copy-on-write (COW), sometimes referred to as implicit sharing or shadowing, is a resource management technique used in computer programming to efficiently implement a "duplicate" or "copy" operation on modifiable resources. If a resource is duplicated but not modified, it is not necessary to create a new resource; the resource can be shared between the copy and the original. Modifications must still create a copy, hence the technique: the copy operation is deferred to the first write. By sharing resources in this way, it is possible to significantly reduce the resource consumption of unmodified copies, while adding a small overhead to resource-modifying operations.





