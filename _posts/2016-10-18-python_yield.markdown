---
layout: post
title:  "Iterator, Generator 与 Yield"
categories: Python 
---

# Iterator

Iterator(迭代器) 是一种常见的对象，它允许调用者方便的遍历该对象的元素，广泛的运用于多种编程语言，如 Python, Java 和 Ruby 等。以 Python 为例，Iterator 需要实现两种方法：

- \_\_iter\_\_(): 返回一个迭代器对象。

- next(): 返回对象中的下一个元素，如果没有，则抛出 StopIteration 异常（Python 3 为 \_\_next\_\_()）。

Java 还要求它提供 hasNext() 方法：

- hasNext()：判断对象是否有下一个元素，如果有，返回 True，否则返回 False。

例如，采用 Count 构造的对象就是一个迭代器，它实现了 \_\_iter\_\_() 和 next() 这两个方法。

~~~
class Count(object):

    def __init__(self):
        self.count = 0

    def __iter__(self):
        return self

    def next(self):
        self.count = self.count + 1
        return self.count
~~~

运行结果：

~~~
>>> a = Count()
>>> dir(a)
[..., '__init__', '__iter__', 'count', 'next']
>>> a.next()
1
>>> a.next()
2
>>> b = a.__iter__()
>>> b.next()
3
~~~

--------------

# Generator
 
 在介绍 Generator(生成器) 前，首先需要明确一点：
 
 - 所有的 Generator 都是 Iterator！
 
 所以 Generator 只是 Iterator 的一类子集，所以 Generator 也必须实现 \_\_iter\_\_() 和 next() 这两种方法，那么 Generator 到底是什么呢？Generator 就是函数的返回值，但是这个函数必须包含一个或者多个 Yield 关键字。例如：
 
~~~
def count():
    c = 0
    while True:
        c = c + 1
        yield c
~~~

运行结果：

~~~
>>> a = count()
>>> dir(a)
[..., '__iter__', 'close', 'gi_code', 'gi_frame', 'gi_running', 'next', 'send', 'throw']
>>> a.next()
1
>>> a.next()
2
>>> b = a.__iter__()
>>> b.next()
3
~~~

上例不难看出，Generator 还实现了 close(), send(), throw() 等方法。

------------

# Yield

Yield 是 Python 的关键字之一，于 Python 2.5 版本引入。通过 yield 表达式，用户可以快速的实现一个迭代器，无需自己实现 \_\_iter\_\_() 和 next() 等方法。当一个函数包含一个或者多个 yield 时，如果调用该函数，则返回一个生成器对象。当该对象第一次调用 next 方法时，生成器才开始执行函数直到遇到 yield 暂停挂起并保留此时状态，并把 yield 后面的参数作(如果 yield 后面没有参数，返回值为 None)为返回值返回给调用者。之后每次调用 next() 方法，生成器从上次挂起出执行，直到遇上下一个 yield，然后挂起并返回 yield 后面的参数。当调用 next() 方法至函数结束也没有遇上 yield 时，此次调用跑出 StopIteration 异常。

~~~
def func():
    yield 1
    yield 2
    yield 3
~~~

运行结果：

~~~
>>> a = func()
>>> a.next()
1
>>> a.next()
2
>>> a.next()
3
>>> a.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
~~~



