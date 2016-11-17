---
layout: post
title:  "Iterator, Generator 与 Yield"
categories: Python 
---

# Iterator

Iterator(迭代器) 是一种常见的对象，它允许调用者方便的遍历该对象中的元素，广泛的运用于多种编程语言，如 Python, Java 和 Ruby 等。以 Python 为例，Iterator 需要实现两种方法：

- \_\_iter\_\_(): 返回一个迭代器对象。

- next(): 返回对象中的下一个元素，如果没有，则抛出 StopIteration 异常（Python 3 为 \_\_next\_\_()）。

Java 还要求实现 hasNext() 方法：

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
 
所以 Generator 也必须实现 \_\_iter\_\_() 和 next() 这两种方法。那么 Generator 到底是什么呢？Generator 就是某个函数的返回值，但是这个函数必须包含一个或者多个 Yield 关键字。例如：
 
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
>>> a.next()
1
>>> a.next()
2
>>> b = a.__iter__()
>>> b.next()
3
~~~

------------

# Yield

[Yield](https://www.python.org/dev/peps/pep-0342/) 是 Python 的关键字之一，于 [Python 2.5](https://docs.python.org/3/whatsnew/2.5.html) 版本引入。通过 yield 表达式，用户可以快速的实现一个迭代器，无需自己实现 \_\_iter\_\_() 和 next() 等方法。当一个函数包含一个或者多个 yield 时，如果调用该函数，则返回一个生成器对象。当该对象第一次调用 next 方法时，生成器才开始执行函数直到遇到 yield 暂停挂起并保留此时状态，并把 yield 后面的参数作(如果 yield 后面没有参数，返回值为 None)为返回值返回给调用者。之后每次调用 next() 方法，生成器从上次挂起出执行，直到遇上下一个 yield，然后挂起并返回 yield 后面的参数。当调用 next() 方法至函数结束也没有遇上 yield 时，此次调用跑出 StopIteration 异常。

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
>>> dir(a)
[..., '__iter__', 'close', 'gi_code', 'gi_frame', 'gi_running', 'next', 'send', 'throw']
~~~

除 next() 方法外，Generator 还实现了 close(), throw(), send() 等方法。

- close(): 结束迭代，generator 调用 close() 后如再调用 next() 会报错。

~~~
>>> a = func()
>>> a.next()
1
>>> a.close()
>>> a.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
~~~

- throw(): Generator 结束迭代并抛出一个异常。

~~~
>>> a = func()
>>> a.next()
1
>>> a.throw(Exception)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "t.py", line 2, in func
    yield 1
Exception
>>> a.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
~~~

- send()：该方法稍微难以理解，介绍该方法前，首先看看如下代码，__该代码表明 yield 不仅能返回结果，还能接收参数__！

~~~
def foo():
    while True:
        in_value = (yield "Please enter something")
        print in_value
~~~

运行结果如下：

~~~
>>> a = t.foo()
>>> a.next()
'Please enter something:'
>>> a.send("hello")
hello
'Please enter something:'
>>> a.send("yield")
yield
'Please enter something:'
~~~

注意，当初始化一个生成器对象时，首先需调用 next() 直到遇上 yield，然后调用 send() 方法传入参数，之后生成器继续执行直到遇到下一个 yield。另外 send(None) 和 next() 的功能完全一样，因此从功能的角度上看，next() 相当于 send() 的一个子集。

Yield 不仅可以方便的实现一个 Generator/Iterator，它还提供了 send(), next(), close() 等方法，这些方法允许调用者和 Generator 之间进行强大的通信，每次调用 next() 或者 send()，Genertor 在下一个 yield 处暂停挂起，保存状态！

