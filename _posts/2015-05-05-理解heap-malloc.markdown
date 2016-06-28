---
layout: post
title:  "理解 heap --- 实现一个简单的 malloc"
categories: Linux
---

# 理解 Heap

~~~
  high address   +---------------+
                 |               |
                 |     Stack     |
                 |               |
                 +---------------+
                 |       |       |
                 |       v       |
                 |               |
                 |               |
                 +---------------+
                 |      Mmap     |
                 +---------------+
                 |               |
                 |               |
                 |       ^       |
                 |       |       |
                 +---------------+
                 |               |
                 |     Heap      |
                 |               |
                 +---------------+
                 |     Data      | 
                 +---------------+
                 |     Code      |
  low address    +---------------+
~~~

上图是 Linux 进程的[地址空间](https://en.wikipedia.org/wiki/Address_space)，从低位到高位地址分别为：

- Code Segment: 程序的代码，CPU 执行的指令部分，共享只读。
- Data Segment: 可细分为初始化数据段和未初始化数据段，常用于存储全局变量等。
- Stack: 函数以及自动变量(未加 static 的自动变量又称为局部变量)。
- Mmap: mmap 调用分配的地址空间。
- Heap: 动态分配内存，如 malloc() 分配的内存。

本文主要讲解 heap，从上图可知，进程的堆是一段连续的空间，它分为三个区域:

- Mapped region: 该区域的空间已经在物理地址上分配，可以直接被程序使用。
- Unmapped region: 该区域的空间未在物理地址上分配，需分配后才可以使用。
- Unusable region: 不可使用的地址空间，超出 rlimit 的空间都是不可使用的，不同的硬件和操作系统下，rlimit 可能各不相同。

其中 break 和 rlimit 是三个区域的分界线：

- break: mapped region 和 unmapped region 的分界线，可调用 sbrk(0) 返回当前的 break。
- rlimit: 堆能分配的最大地址空间，getrlimit(2) 可获取 rlimit。
- start of heap: 堆的起始地址，当堆上从未开辟空间时，sbrk(0) 返回的就是堆的起始地址，也可在 /proc/{proc\_id}/mapping 获取堆的起始地址。

~~~
                         Heap

   high address  +-------------------+
                 |                   |
                 |  Unusable Region  |
                 |                   |
                 +-------------------+  <--- rlimit
                 |                   |
                 |                   |
                 |  Unmapped Region  |
                 |                   |
                 |                   |
                 +-------------------+  <--- break
                 |                   |
                 |   Mapped Region   |
                 |                   |
   low address   +-------------------+  <--- start of heap
~~~

根据运行的需要，程序可以向堆动态的申请和释放空间，当程序所需要的空间超过 mapped region 时，就需要向操作系统申请扩展堆的空间，即把 break 移动到更高的地址，linux 提供了两个系统调用用于调整堆的大小。

- [brk(const void *addr)](http://man7.org/linux/man-pages/man2/sbrk.2.html): 直接把 break 移动到 addr 地址处。
- [sbrk(int incr)](http://linux.die.net/man/2/sbrk): 把 break 移动 incr 个字节。

除此以外，当程序申请较大(比如大于 128 KB)的空间时，linux 还提供了 [mmap()](http://man7.org/linux/man-pages/man2/mmap.2.html) 系统调用分配空间，严格来讲，mmap 分配的地址空间并不属于 heap，它介于 heap 和 stack 之间。

当程序不再需要某块内存时，通常使用 free() 释放它，释放这块内存时，并不会告知操作系统，它由[运行库(runtime library)](https://en.wikipedia.org/wiki/Runtime_library)管理起来，并标志为空闲状态，我们用资源池表示这些处于空闲状态的内存块的集合。当程序再次申请空间时，运行库首先搜索资源池，如果找到满足要求的内存块，则直接将这块内存供给程序使用，如果资源池没有满足要求的内存块，则调用 sbrk() 向操作系统分配内存。采用资源池的方式管理堆空间，避免每次分配和释放内存时都需要告知操作系统，减少了系统调用的开销。如何按需分配、高效的管理堆空间，这就是堆分配算法。它通常由运行库实现，malloc 和 free 等组成了堆的分配算法的一种实现。

~~~
                                                system call

   +-----------+               +--------------+             +------------+
   |           |    malloc     |   Runtime    |    sbrk     |            |
   | Programme |   -------->   |   library    |  -------->  |     OS     |
   |           |    free       |              |     brk     |            |
   +-----------+               +--------------+             +------------+
~~~

--------

# 实现一个简单的 malloc

本节主要介绍如何实现一个简单的 malloc，malloc 属于运行库的一个函数，它管理着已向操作系统申请好的堆空间。比如，它必须记录 mapped region 里有多少空间，哪些块已经被分配，哪些块处于空闲状态，以及它们的地址和大小信息。管理这些块的方法有多种，比如链表、位图等。本节采用列表的方式管理这些内存块：

~~~
                     Mapped Region

   high address  +-------------------+  <--- break
                 |                   |
                 |      Block N      |
                 |                   |
                 +-------------------+
                 |  Block N metadata |
                 +-------------------+
                 |                   |
                 |      ......       |
                 |                   |
                 +-------------------+  
                 |                   |
                 |      Block 1      |
                 |                   |
                 +-------------------+
                 |  Block 1 metadata |
   low address   +-------------------+  <--- start of heap
~~~

如上图所示，每块内存包含两部分，block 和 block metadata，其中 block 用于存放程序的数据，block metadata 用于描述该 block 的基本信息，如大小、状态等:

~~~
                     Block Metadata

                 +-------------------+ 
                 |       size        |
                 +-------------------+
                 |       free        |
                 +-------------------+ 
                 |       next        |
                 +-------------------+
                 |       prev        |
                 +-------------------+  
~~~ 

综上，一个 malloc 的样例为：

~~~ c
#include <stdlib.h>
#include <unistd.h>

#define BLOCK_SIZE sizeof(struct block_meta)

void *heap_start_addr = NULL;

struct block_meta {
    struct block_meta *prev;
    struct block_meta *next;
    int free;
    size_t size;
};

void get_heap_start_addr(){
    if(!heap_start_addr)
        heap_start_addr = sbrk(0);
}

void *get_last_block(){
    struct block_meta *blk_meta;

    blk_meta = heap_start_addr;
    while(blk_meta->next){
        blk_meta = blk_meta->next;
    }
    return blk_meta;
}

void *find_block(size_t size){
    struct block_meta *blk_meta;

    blk_meta = heap_start_addr;
    while(blk_meta){
        if(blk_meta->free)
            if(blk_meta->size >= size)
                break;
        blk_meta = blk_meta->next;
    }
    return blk_meta;
}

void *extend_heap(void *prev, size_t size){
    void *cur_brk;
    struct block_meta *cur_blk_meta, *prev_blk_meta;

    cur_brk = sbrk(size + BLOCK_SIZE);
    if(!cur_brk){
        return NULL;
    }
    else{
        cur_blk_meta = cur_brk;
        cur_blk_meta->free = 0;
        cur_blk_meta->size = size;
        cur_blk_meta->prev = prev;
        cur_blk_meta->next = NULL;

        if(prev){
            prev_blk_meta = prev;
            prev_blk_meta->next = cur_blk_meta;
        }

        return cur_brk + BLOCK_SIZE;
    }
}

void split_block(struct block_meta *blk_meta, size_t size){
    struct block_meta *free_blk_meta;

    if(size + BLOCK_SIZE < blk_meta->size){
        free_blk_meta = blk_meta + size + BLOCK_SIZE;
        free_blk_meta->prev = blk_meta;
        free_blk_meta->next = blk_meta->next;
        free_blk_meta->size = blk_meta->size - size - BLOCK_SIZE;
        free_blk_meta->free = 1;

        blk_meta->size = size;
        blk_meta->free = 0;
        blk_meta->next = free_blk_meta;
    }
    else{
        blk_meta->free = 0;
    }
}

void *malloc(size_t size) {
    void *split_blk, *cur_brk, *ptr;
    struct block_meta *last_blk;

    if(!size)
        return NULL;

    // align with 8 bytes.
    if(size & 0x7){
        size = ((size >> 3) + 1) << 3
    }

    get_heap_start_addr();
    cur_brk = sbrk(0);
    if(heap_start_addr == cur_brk){
        // First malloc, just extend the brk.
        ptr = extend_heap(NULL, size);
        if(ptr)
            return ptr;
        else
            return NULL;
    }
    else{
        split_blk = find_block(size);
        if(split_blk){
            // There is a free block, and we split it.
            split_block(split_blk, size);
            ptr = split_blk + BLOCK_SIZE;
            return ptr;
        }
        else{
            // No block is valid, extend the heap.
            last_blk = get_last_block();
            ptr = extend_heap(last_blk, size);
            if(ptr)
                return ptr;
            else
                return NULL;
        }
    }
}
~~~

类似的，free 的实现如下：

~~~ c
void free(void *ptr) {
    struct block_meta *cur_blk_meta, *prev_blk_meta, *next_blk_meta;

    if(!ptr)
        return ;

    cur_blk_meta = ptr - BLOCK_SIZE;
    cur_blk_meta->free = 1;

    next_blk_meta = cur_blk_meta->next;
    if(next_blk_meta){
        if(next_blk_meta->free){
            cur_blk_meta->next = next_blk_meta->next;
            cur_blk_meta->size += next_blk_meta->size + BLOCK_SIZE;
        }
    }

    prev_blk_meta = cur_blk_meta->prev;
    if(prev_blk_meta){
        if(prev_blk_meta->free){
            prev_blk_meta->next = cur_blk_meta->next;
            prev_blk_meta->size += cur_blk_meta->size + BLOCK_SIZE;
        }
    }
}
~~~