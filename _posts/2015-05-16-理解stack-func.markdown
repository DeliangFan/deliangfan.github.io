---
layout: post
title:  "理解 stack --- 函数是如何调用的"
categories: Linux
---

# 理解 Stack

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

上图是 Linux 进程的地址空间，笼统的说，进程的空间可以分为两个大类：指令和数据，指令即是上图的 code 段，其它的均属于数据。根据数据在生命周期以及作用范围的特点，又可分为 data, heap 和 stack，data 通常用于存放全局变量，它的生命周期伴随着整个程序，可被所有的线程共享，动态分配的数据通常存放在 heap，可被所有的线程共享，stack 存放的是函数以及函数的局部变量，它们同样是动态的分配和释放，但是不被其它线程共享。

在数据结构领域，stack 是一种后进先出的结构，设想某个函数被调用时，我们需要为该函数开辟空间，当调用结束后，空间将被释放，所以 stack 这种组织结构非常适用函数的调用。本文主要介绍 stack 以及函数是如何调用的。


-------

# Stack Frame

要实现函数调用，必须解决以下几个问题：

- 参数的传递
- 返回值的传递
- stack 的维护

操作系统使用[栈帧(stack frame)](https://en.wikipedia.org/wiki/Call_stack)实现函数的调用，它的结构如下：

~~~
  high address
                                 +---------------+
                                 |               |
                                 |      Old      |
                                 |     Stack     |
                                 |     Frame     |
                                 |               |
                --------------   +---------------+
                      ^     ^    |   Parameters  |
                      |   caller +---------------+
                      |     v    |  Return addr  |
                           ---   +---------------+
                Stack Frame ^    |    Old  ebp   |   <------- ebp  Frame Pointer
                            |    +---------------+
                      |  callee  |   Registers   |
                      |     |    +---------------+
                      v     v    |   Local vars  |
                --------------   +---------------+   <------- esp  Stack Pointer

low address  
~~~

首先介绍 x86 下的两个寄存器：

- esp：栈寄存器，永远指向程序的栈顶。
- ebp：栈帧寄存器，指向当前的帧

我们把主动调用函数的一方称为 caller，被调用的一方称为 callee，当函数被调用时，caller 首先把参数压栈，然后压栈返回地址，之后跳到相应的函数执行。当跳到相应的函数时，callee 需保存上个函数的帧，即把上个函数的帧地址压入栈中，并更新当前的帧寄存器，之后需压栈保存某些寄存器的值，最后才是为 callee 函数体内的局部变量开辟空间。当函数调用结束后，当前栈帧所占用的空间已经没有用处，所以需要释放这片空间，即把当前栈帧的所有数据全部弹出。

函数的调用规则有多种，这些规则主要表现在某些细节的差异，比如参数的压栈顺序，参数的出栈。[Cdecl](http://linux.die.net/man/1/cdecl) 是 C 语言默认的规则，它规定参数从右到左压栈，调用方负责参数的出栈。

----------

# Example

以下是个简单的 c 样例：

~~~ c 
int add_1(int a){
    return a + 1;
}

void main(){
    int a = 1;
    a = add_1(a);
}
~~~

编译所得的汇编如下：

~~~ s
main:
    ......
    movl    $1, -4(%ebp)    # set a = 1
    leal    -4(%ebp), %eax  # set eax = a
    movl    %eax, 4(%esp)   # store on stack
    call    add_1           # call add_1
    ......

add_1:
    pushq   %ebp            # save old ebp
    movq    %esp, %ebp      # update the ebp
    movl    -4(%ebp), %eax  # set eax = a
    addl    $1, %eax        # eax = eax + 1
    popq    %ebp            # restore ebp
    ret                     # return
~~~