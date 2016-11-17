---
layout: post
title:  "Yield 和 Coroutine"
categories: Python 
---

博文 [Iterator, Generator 与 Yield](http://wsfdl.com/python/2016/10/18/python_yield.html) 介绍了 iterator，generator 和 yield，并阐述三者之间的关系。本文将进一步介绍 yield 和 coroutine，并阐述如何通过 yield 实现一个简单的 coroutine。

[Coroutine(协程)](https://en.wikipedia.org/wiki/Coroutine) 最早于 1963 年提出，在之后的三四十年里并没有受到广泛关注，但在近些年受到热捧。通俗而言：协程相当于用户态线程。这句话包含了两层意思：

- 协程具有类似线程的功能，它能提供并发。
- 协程是用户态的，即操作系统对协程不感知，也不负责调度；应用程序负责管理协程的生命周期和调度协程，由于协程的切换是函数级别的切换，故切换的开销远远小于线程/进程。

为什么协程在近些年越来越火？原因得从并发谈起，随着互联网爆炸式增长，服务端对并发能力的需求越来越大。最初工程师采用多进程提供并发，每服务端当收到一个请求，就 fork 一个进程处理请求，Apache 是最典型的例子。但是进程很重，占用了的大量 CPU 和内存资源，和进程相比，线程占用更少的资源，所以多线程的并发模型更受欢迎，每当服务端收到一个请求，就创建一个线程处理请求，如 Nginx。每个线程维护私有的 stack，Linux 下 stack 的默认大小为 8MB，所以 8G 内存的 Linux 服务器最多能创建 1000 个线程；此外线程的调度由内核负责，调度和切换的开销也不容小觑，以上两个因素限制了多线程模型的并发能力。

为了提升服务端的并发能力，我们需要一种并发模型，这种模型具有以下特点：

- 占用更少的资源，如内存和 CPU 周期
- 避免内核调度带来的额外开销，协程仅在必要的时候才被调度切换，如 IO 操作。

而协程作为用户态线程，恰好满足上述要求，首先协程的 stack 比线程的 stack 更小，占用更少的内存空间(KB 级别)，无需通过系统调用来创建和销毁，故消耗更少的 CPU 周期；另外，协程的调度由应用程序负责，仅在必要的时候才切换，和线程相比，减少了切换的次数和每次切换的开销。

回顾 Yield，我们用它编写一个如下的 Generator。

~~~ python
def task1():
    while True:
        # Do something
        yield "This is task1"
~~~

运行：

~~~ bash
>>> t1 = task1()              # Initiate a task
>>> t1.next()                 # Run task
>>> This is task1             # Task suspend when meeting next yield
>>> t1.Send(None)             # Run task
>>> This is task1             # Task suspend when meeting next yield
>>> t1.close()                # Delete task
~~~

当我们调用一个包含 yield 的函数时，相当于初始化了一个协程，我们可以调用 next() 或者 send() 让协程运行，每当 task 遇到 yield 则自动暂停挂起，我们也可以调用 close() 结束一个协程。以下是进程，线程和上例协程生命周期管理的部分函数。

~~~
Method          Process         Thread           Coroutine
----------------------------------------------------------------------
Create/Run      fork            pthread_create   task1()/next/send
Delete          exit            pthread_exit     close
~~~

上例的协程具备基本的生命周期管理的方法，我们现在往其加入调度功能，调度算法为 FIFO。

~~~ python
class Scheduler(object):
    def __init__(self):
        self.queue = []
        self.task_num = 0

    def new(self, task):
        self.queue.append(task)
        self.task_num += 1

    def loop(self):
        while self.task_num:
            task = self.queue.pop(0)
            task.next()
            self.queue.append(task)


def task2():
    while True:
        # Do something
        yield "This is task2"
~~~

运行结果：

~~~ bash
>>> scheduler = Scheduler()
>>> scheduler.new(task1())
>>> scheduler.new(task2())
>>> scheduler.loop()
This is task1
This is task2
This is task1
This is task2
......
~~~

流程图如下，每当 Main Loop 调用 next()/send()，执行权交给相应协程，每当协程遇到 yield，则交出执行权给 Main Loop。该流程图和 Linux 的进程调度非常类似，如果把 Main Loop 比作内核，协程如同进程/线程，yield 如同系统调用、硬件中断等。任何时刻，一个 Python 进程内只有一个协程在执行，即协程是伪并发的。

~~~
           Run       Run      Run      Run      Run
Main Loop  --->      -->      -->      -->      --->
               |    |   |    |   |    |   |    |
               |Run |   |    |   |Run |   |    |     ......
Task1           --->    |    |    --->    |    |
                        |Run |            |Run |
Task2                    --->              --->
~~~

本文借用 yield 实现了一个非常简单的协程，它的调度功能非常简陋，不支持同步功能，没有处理阻塞的 IO，无法处理复杂事务，另推荐 [A Curious Course on Coroutines and Concurrency](http://www.dabeaz.com/coroutines/) 和 Python 协程库 [eventlet](http://eventlet.net/)，[gevent](http://www.gevent.org/) 做进一步的学习。