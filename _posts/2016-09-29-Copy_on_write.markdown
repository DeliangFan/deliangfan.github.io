---
layout: post
title:  "理解 Copy On Write 技术"
categories: Algorithm
---

# Copy-On-Write Overview

随着阅历的增长，越发感慨在千差万别的软件中总能找到某些非常相通相似的思想、技术或者算法，比如 [Copy On Write(写时复制)](https://en.wikipedia.org/wiki/Copy-on-write) 便是资源管理方面的一种优化技术，广泛的应用于软件开发(如 [Java 的 Copy On Write 容器](http://ifeve.com/java-copy-on-write/))、虚拟内存管理(如进程的 [fork](https://en.wikipedia.org/wiki/Fork_(system_call))、qemu-kvm [虚拟机镜像](https://en.wikipedia.org/wiki/Qcow)乃至 Docker 的 [AUFS 文件系统](https://en.wikipedia.org/wiki/Aufs)等等。[维基百科](https://en.wikipedia.org/wiki/Copy-on-write)的定义如下：

> Copy-on-write (COW), sometimes referred to as implicit sharing or shadowing, is a resource management technique used in computer programming to efficiently implement a "duplicate" or "copy" operation on modifiable resources. If a resource is duplicated but not modified, it is not necessary to create a new resource; the resource can be shared between the copy and the original. Modifications must still create a copy, hence the technique: the copy operation is deferred to the first write. By sharing resources in this way, it is possible to significantly reduce the resource consumption of unmodified copies, while adding a small overhead to resource-modifying operations.

通俗的解释，假定多方需要使用同一个资源时，没有必要为每一方都创建该资源的一个完整的副本，反而令多方共享这个资源，当某方需要修改资源的某处时，利用引用计数，把该处复制一个副本，再把跟新的内容写入该副本中，从而节省创建多个完整副本时带来的空间和时间上的开销。

~~~
                             Resource 

                             +-------+
                             |   1   |
                             +-------+
User A   ---->               |   2   |               <---- User B
                             +-------+          
                             |   3   |
                             +-------+                             
~~~

当 A 需更新 Block 1 时，A 新建 New 1 Block，并把新的内容写入该 Block，从而避免影响 B 的 Block 1。同理，当 B 需更新 Block 3 时，B 新建 New 3 Block，并把新的内容写入该 Block，避免影响 A 的 Block 3。

~~~
                             Resource 

                +-------+    +-------+
                | New 1 |    |   1   |
                +-------+    +-------+
User A   ---->               |   2   |               <---- User B
                             +-------+    +-------+          
A see: New 1, 2, 3           |   3   |    | New 3 |        B see: 1, 2, New 3
                             +-------+    +-------+
~~~

-------------------

# Example

## C++ STL String Class

C ++ 曾在性能问题上被广泛地质疑指责过，为了提升性能，STL 的许多类采用了 Copy-On-Write 技术。Coolshell 博文 [C++ STL string 的 Copy-On-Write 技术](http://coolshell.cn/articles/12199.html) 详细的从 C++ 层面介绍改技术。例如：

~~~ c
#include <stdio.h>
#include <string>
using namespace std;
  
main()
{
    string str1 = "hello world";
    string str2 = str1;
 
    printf ("Sharing the memory:/n");
    printf ("/tstr1's address: %x/n", str1.c_str() );
    printf ("/tstr2's address: %x/n", str2.c_str() );
 
    str1[1]='q';
    str2[1]='w';
 
    printf ("After Copy-On-Write:/n");
    printf ("/tstr1's address: %x/n", str1.c_str() );
    printf ("/tstr2's address: %x/n", str2.c_str() );
}
~~~

运行结果如下：

~~~ bash
> g++ -o stringTest stringTest.cpp
> ./stringTest
Sharing the memory:
        str1's address: 343be9
        str2's address: 343be9
After Copy-On-Write:
        str1's address: 3407a9
        str2's address: 343be9
~~~

## Fork()


早期的 Unix 在实现 fork 系统调用时，并没有使用该技术，创建新进程的开销很大。出于效率考虑，Copy On Write 技术引入到进程中，fork 之后的父进程和子进程完全共享数据段、代码段、堆和栈等的完全副本，而且内核将共享的地址空间的访问权限改变为只读，如果父进程和紫金陈中的任何一个仕途修改这些区域，则内核职位修改区域的那块内存制作一个副本，通常是虚拟存储系统中的一“页”(引自 [Unix 高级环境编程](https://book.douban.com/subject/1788421/))。


## Qcow2 image

[Qcow2(QEMU Copy On Write)](http://www.linux-kvm.org/page/Qcow2) 是格式镜像是 qemu 模拟器支持的一种镜像，和普通的 Raw 格式镜像相比，具有以下优点：

- 更小的空间占用
- 支持 Copy On Write.
- 支持多级快照
- 支持压缩
- 支持 AES

经过多次优化，Qcow2 镜像的性能已经逼近 Raw 格式的镜像，所以 OpenStack 的虚拟机多采用 Qcow2 镜像。

ide 驱动时的性能对比：

~~~
cache                      off         writethrough      writeback
Old qcow2 (0.10.5)         16:52 min   28:58 min         6:02 min
New qcow2 (0.11.0-rc1)     5:44 min    9:18 min          6:11 min
raw                        5:41 min    7:24 min          6:03 min
~~~

virtio 驱动时的性能对比：

~~~
cache                      off         writeback
Old qcow2 (0.10.5)         31:09 min   8:00 min
New qcow2 (0.11.0-rc1)     18:35 min   8:41 min
raw                        8:48 min    7:51 min
~~~

除了镜像格式以外，很多文件系统都使用了 Copy-On-Write 技术，以节省空间、提升效率和更好的支持快照。
