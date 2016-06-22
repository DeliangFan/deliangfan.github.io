---
layout: post
title:  "理解 Python 关键字 With"
categories: Python 
---

# With Statement

和 if, for, while 等一样，with 也是 python 的关键字，最早于 2.5 版本引入。它封装了 try...except...finally 编码范式，提高了易用性。以读取文件为例，采用 try...except...finally 的代码如下：

~~~ python
try:
    f = open('a.txt')
    lines = f.readlines()
finally:
    f.close()
~~~

采用 with 的代码如下：

~~~ python
with open('a.txt') as f:
    lines = f.readlines()
~~~

当 with 语句结束时，它自动调用 f.close() 关闭文件，故其代码更为简洁，除了文件 IO 的操作以外，with 也非常适合锁和信号量的获取和释放场景。但是并非所有的对象都支持 with，我们把支持 with 的对象称为 [context manager](https://docs.python.org/2/reference/datamodel.html#context-managers)。


-------------

# Context Manager

Context manager 是实现了 [context manager 协议](https://docs.python.org/2/library/stdtypes.html#typecontextmanager)的对象，该对象要实现以下两个方法：

- object.\_\_enter\_\_(self)

该方法在 with 语句开始的时候执行，返回一个对象，这个对象多为 context manager 对象自身，也可以为其它类型的对象，并赋值给 as 后面的参数。以 open('a.txt') 为例，with 在执行该语句时，调用了 open('a.txt').\_\_enter\_\_() 方法，该方法返回对应的文件描述符，并赋值给 as 后面的参数 f。

- object.\_\_exit\_\_(self, exc\_type, exc\_value, traceback)

With 语句在结束时会调用该方法，该方法一般多用于清理现场，比如关闭文件，释放锁等。exc\_type, exc\_value 和 traceback 记录执行过程中的异常，如果没有异常产生，这三个值均为 None；如果有异常产生，我们可以根据需要在该函数中处理异常，该方法可以充当 try...except...finally 的角色。

一个对象只要实现上述两种方法，该对象就可以被 with 执行，例如：

~~~ python
class ContextManager(object):
    def __enter__(self):
        print "Enter"
        return self

    def __exit__(self, exc_type, exc_value, exc_traceback):
        print "Exit"


with ContextManager() as cm:
    print "Body"
~~~

输出如下：

~~~
Enter
Body
Exit
~~~

当执行过程中出现异常时，我们可以在 \_\_exit\_\_() 处理异常，如果不希望把异常往上抛，则把返回值置为 True。

~~~ python
class ContextManager(object):
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, exc_traceback):
        print exc_type
        print exc_value
        print exc_traceback
        return True

with ContextManager() as cm:
    exc = 1 / 0
print "This line code is still excuted because no exception has been raised."
~~~

运行结果如下：

~~~ python
<type 'exceptions.ZeroDivisionError'>
integer division or modulo by zero
<traceback object at 0x10358e5f0>
This line code is still excuted because no exception has been raised.
~~~

