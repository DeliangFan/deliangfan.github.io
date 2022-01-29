---
layout: post
title:  "进程究竟用了多少内存"
categories: Linux
---

我们所接触的内存相关概念中，既有物理内存，又有虚拟内存；既有内核内存，又有用户态内存；既有进程专属内存，又有共享内存；分配的内存中既有可回收内存，又有不可回收内存；既有 mmap 分配的，又有 brk 调用分配的；内置内存管理的语言而言，既有 Runtime 已申请内存，又有实际被使用的内存。因此绝对准确的回答某个进程所占内存是一件困难之事。

# 从一个 Golang 内存困扰现象说起

## 现象

对于采用 1.12-1.15 golang 编译器和 4.5 及以上内核的 go 应用来说，存在两个困扰的内存问题：

- 从 Linux 视角看到的进程内存远远大于 go pprof 视角看到进程 inuse 内存，有时甚至相差十倍之多。
- Linux 视角内存久居高位不下。

## 分析

事实上，golang runtime 有诸多描述进程内存指标，详情请见 [mstats.go](https://github.com/golang/go/blob/master/src/runtime/mstats.go)，举例而言：


```
# golang 内存指标非常之多，如仅例举核心指标

# 从 OS 申请的内存，等于 sum(heap/stack/mspan sys)，基本等同于 linux VIRT 指标。
Sys = 46359864776
# Golang Runtime 已分配且未释放的内存，等同 HeapAlloc。
Alloc = 8606478720

# 从操作系统申请用于 heap 的内存
HeapSys = 44083806208
# heap 上已分配且未释放的内存
HeapAlloc = 8606478720
# heap 上用于 MSpan 的内存
HeapInuse = 10107944960
# heap 调用 madvise 归还给 OS 的内存，但是这部分内存还可以再次被申请。
HeapReleased = 32709271552
# heap 未被用于 MSpan 的内存，这部分可以归还给 OS。
HeapIdle = 33975861248

# StackInuse / StackSys，二者基本相等，表示 stack 所用资源。
Stack = 6717440 / 6717440

# 构建 MSpan 结构体使用的内存(MSpanInuse/MSpanSys)
MSpan = 216646912 / 460046336

# 下次 GC 触发的阈值(HeapAlloc) 
NextGC = 16474941344
......
```

top 的指标如下：

```
VIRT: 46.458g
RSS:  0.028t                                                                                                                                   
```

故从 Golang 的视角来看：

- Alloc 基本代表进程的实际运行所需要的内存。
- Sys 表示从操作系统分配的内存，基本等同于 VIRT。
- HeapReleased 则表示一种中间状态的内存，既可以被操作系统回收，也可以被 Runtime 再次申请的内存。正是内存大户 HeapReleased 带来了上述困扰。

## Linux madvise

从 [go memory runtime](https://go.dev/src/runtime/mem_linux.go) 不难发现，golang 通过 mmap 分配内存，通过 munmap 和 madvise 释放内存。而 munmap 封装在 runtime.sysFree 中，该函数被调度的场景通常在内存分配异常过程中，详细介绍如下：

> sysFree transitions a memory region from any state to None. Therefore, it returns memory unconditionally. It is used if an out-of-memory error has been detected midway through an allocation or to carve out an aligned section of the address space. It is okay if sysFree is a no-op only if sysReserve always returns a memory region aligned to the heap allocator's alignment restrictions.

因此对于常态的内存释放，主要是通过 [madvisor syscall](https://man7.org/linux/man-pages/man2/madvise.2.html) 来完成，man 手册对 madvise 的详细介绍如下：

> int madvise(void *addr, size_t length, int advice);
> 
> The madvise() system call is used to give advice or directions to the kernel about the address range beginning at address addr and with size length bytes In most cases, the goal of such advice is to improve system or application performance.

在 4.5 内核之后 advice 开始支持 MADV\_FREE 参数，当完成 madvisor 调用后，内核会立刻回收 file pages，对于 anonymous page，内核仅在内存紧张时回收对应的内存，在实际回收之前，进程依旧可以使用这部分内存。

> The application no longer requires the pages in the range specified by addr and len.  The kernel can thus free these pages, but the freeing could be delayed until memory pressure occurs.  For each of the pages that has been marked to be freed but has not yet been freed, the free operation will be canceled if the caller writes into the page.
> 
> MADV\_FREE will unmap file pages. MADV_FREE on anonymous pages is interpreted as a signal that the application no longer needs the data in the pages, and they can be thrown away if the kernel needs the memory for something else.

golang 在 1.12 之前和 1.16 及其以后采用 MADV\_DONTNEED 选项，当完成 madvisor 调用后，内核马上回收 file page 和 anonymous pages 内存，RSS 显著下降。

> The kernel is free to delay freeing the pages until an appropriate moment. The resident set size (RSS) of the calling process will be immediately reduced however.
> 
> MADV_DONTNEED will unmap file pages and free anonymous pages。

对于 1.12-1.15 golang 在 4.5 及以上内核版本默认采用  MADV\_FREE 参数，是一种以空间换性能的做法，同时给用户造成很大的困扰；用户可以配置 GODEBUG=madvdontneed=1 强制使用 MADV\_DONTNEED。

# A Little More

事实上，上节提到的 file-backed page 和 annoymous page 通常占据着进程的大部分内存，也是 /proc/memoinfo 的重要参数；tmpfs、slab 等等也是常见的概念；它们和 brk 和 mmap 系统调用分配的内存有什么关系呢？[这篇博文](https://techtalk.intersec.com/2013/07/memory-part-1-memory-types/)做了如下条理性总结：

|    |    Private  |  Shared    |
|  ---  |  ----   |  ----  |
| anonymous  |  stack / malloc(brk/sbrk) / mmap(ANON, PRIVATE) | mmap(ANON, SHARED)  |
|  file-backed  | mmap(fd, PRIVATE) / binary / shared libraries | mmap(fd, SHARED) / shm |

正如其字面意思，Private 和 Shared 非常好理解，所谓 Private 指内存独属某一个进程，大部分场景下所用的内存均为此；Shared 则表示内存可以被一个或者多个进程共享使用，比如 tmpfs，基于 mmap 的进程间通信。

所谓的 Anonymous page，[内核](https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html#anonymous-memory)的解释如下：

> The anonymous memory or anonymous mappings represent memory that is not backed by a filesystem. Such mappings are implicitly created for program’s stack and heap or by explicit calls to mmap(2) system call. Usually, the anonymous mappings only define virtual memory areas that the program is allowed to access. The read accesses will result in creation of a page table entry that references a special physical page filled with zeroes. When the program performs a write, a regular physical page will be allocated to hold the written data. The page will be marked dirty and if the kernel decides to repurpose it, the dirty page will be swapped out.

因此 Anonymous page 包括进程的堆、栈，由于 anonymouse 没有对应的文件，仅能交换到 swap 分区上，另外 anonymous page 实际的物理内存分配是发生在进程访问所分配内存时，因此会出现 top 命令中 VIRT 大于 RSS 的现象。

对于 file-backed page，通常有硬盘的文件与之对应，包括进程的代码文件、mmap 映射的文件、内核中的 page cache 等等，当内存不足时，内核可以回收这些内存，并且将数据同步到文件中。

无论是 file-backed page 还是 annoymous page，它们均可以分为 active 和 inactive，其中 active 表示活跃的内存，一般不可被回收，内核回收内存时，优先回收 inactive 的内存。


# 参考资料

-  http://linuxperf.com/?p=142
- https://www.kernel.org/doc/gorman/html/understand/understand007.html
- https://www.sunliaodong.cn/2021/03/11/Linux-PageCache%E8%AF%A6%E8%A7%A3/
- https://techtalk.intersec.com/2013/07/memory-part-1-memory-types/
- http://linuxperf.com/?p=97
