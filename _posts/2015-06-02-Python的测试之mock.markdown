---
layout: post
title:  "Python 的单元测试之 mock"
categories: Python
---

&nbsp;&nbsp;&nbsp;

![testtools](http://wsfdl.oss-cn-qingdao.aliyuncs.com/pythontesttools.png)

&nbsp;&nbsp;&nbsp;

---------------------

# Overview

[mock](http://www.voidspace.org.uk/python/mock/index.html) 是一个用于单元测试的 Python 库，它使用 mock 模拟系统中如 class, method 等部分，并且断言它们是如何被调用的。在编写单元测试时，mock 非常适合模拟数据库，web 服务器等依赖外部的场景。本文是 mock 的入门篇，主要介绍 mock 的基本用法。

除了 mock 外，还有许多其它的 mocking 库，[Python Mock Library Comparison](http://garybernhardt.github.io/python-mock-comparison/) 在用法上对这些库做了简单的比较，其中 OpenStack 单元测试广泛的使用了 mock 和 mox。

- [mock](http://www.voidspace.org.uk/python/mock/)
- [flexmock](https://pypi.python.org/pypi/flexmock)
- [mox](https://pypi.python.org/pypi/mox)
- [Mocker](http://niemeyer.net/mocker)
- [Dingus](https://pypi.python.org/pypi/dingus)
- [fudge](http://farmdev.com/projects/fudge/)

mock 的安装非常简便：

~~~ bash
$ pip install mock
~~~

---------

# Mock Patching Methods


当使用 mock 模拟 methods 时，mock 会替换被模拟的 methods，并且记录调用详情。

~~~ bash
>>> class Foo(object):
...     def echo(self, *args):
...         return "hello"
...
>>>
>>> foo = Foo()
>>> foo.echo = mock.MagicMock()
>>> foo.echo.return_value = "mock value"
>>>
>>> foo.echo()
'mock value'
>>> foo.echo(1, 2)
'mock value'
~~~

被模拟后，foo.echo 的类型是一个名为 mock.MagicMock 类，具有 assert\_any\_call, assert\_called\_once\_with 等方法，其中 assert 类型的方法通常用于检验 foo.echo 是否被正确调用。

~~~ bash
>>> type(foo.echo)
<class 'mock.MagicMock'>

>>> dir(foo.echo)
['assert_any_call', 'assert_called_once_with', 'assert_called_with',
'assert_has_calls', 'attach_mock', 'call_args', 'call_args_list',
'call_count', 'called', 'configure_mock', 'method_calls', 'mock_add_spec',
'mock_calls', 'reset_mock', 'return_value', 'side_effect']
~~~

断言 foo.echo 被调用的情况，其中 foo.echo 共被调用两次(见上)。

~~~ bash
>>> foo.echo.called
True
>>> foo.echo.call_count
2
>>> foo.echo.mock_calls
[call(), call(1, 2)]
>>>
>>> foo.echo.assert_called_with(1, 2)
>>> foo.echo.assert_called_with(1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python2.7/dist-packages/mock.py", line 844, in assert_called_with
    raise AssertionError(msg)
AssertionError: Expected call: mock(1)
Actual call: mock(1, 2)
~~~

mock.Mock 和 mock.MagicMock 是两个常用的类，stackoverflow 有篇帖子 [mock-vs-magicmock](http://stackoverflow.com/questions/17181687/mock-vs-magicmock) 专门讲述二者的区别：

- MagicMock 是 Mock 的之类
- MagicMock 额外实现了很多 [magic 的方法](http://www.voidspace.org.uk/python/mock/magicmock.html#mock.MagicMock)

-------------------------

# Mocking Classes

采用 mock 可方便的模拟 class，例如：

~~~ bash
>>> def some_function():
...     foo = Foo()
...     return foo.echo()
...
>>> with mock.patch('__main__.Foo') as foo_mock:
...     instance = foo_mock.return_value
...     instance.echo.return_value = "mock result"
...     result = some_function()
...     assert result == "mock result"
...
>>> print some_function()
hello
~~~

值得注意的是，[mock.patch](http://www.voidspace.org.uk/python/mock/getting-started.html#patch-decorators) 把模拟的效果限制在 with 作用域的范围内，所以 with 作用域之外的 some_function 的返回值依旧为 hello。

---------------------

# Setting Return Values and Attributes

mock 同样可方便的模拟返回值和 attributes，例如模拟一个对象的返回值，

~~~ bash
>>> value_mock = mock.Mock()
>>> value_mock.return_value = 3
>>> value_mock()
3
~~~

模拟一个方法的返回值：

~~~ bash
>>> method_value_mock = mock.Mock()
>>> method_value_mock.method.return_value = 3
>>> method_value_mock.method()
3
~~~

模拟对象的 attribute：

~~~ bash
>>> attr_mock = mock.Mock()
>>> attr_mock.x = 3
>>> attr_mock.x
3
~~~

-------------

# 参数 side_effect

side\_effect 是一个非常有用的参数，大大提高了 mock 返回值的灵活性，它可以是一个异常、函数或者可迭代对象。例如返回一个异常：

~~~ bash
>>> except_mock = mock.Mock(side_effect=Exception('Boom!'))
>>> except_mock()
Traceback (most recent call last):
  ...
Exception: Boom!
~~~

当 side\_effect 为迭代对象时，样例如下：

~~~ bash
>>> iter_mock = mock.Mock(side_effect=[1, 2, 3])
>>> iter_mock()
1
>>> iter_mock()
2
>>> iter_mock()
3
>>> iter_mock()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Library/Python/2.7/site-packages/mock/mock.py", line 1062, in __call__
    return _mock_self._mock_call(*args, **kwargs)
  File "/Library/Python/2.7/site-packages/mock/mock.py", line 1121, in _mock_call
    result = next(effect)
  File "/Library/Python/2.7/site-packages/mock/mock.py", line 127, in next
    return _next(obj)
StopIteration
~~~

单 side\_effect 为函数时，样例如下：

~~~ bash
>>> def side_effect(value):
...     return value
...
>>>
>>> side_effect_mock = mock.Mock(side_effect=side_effect)
>>> side_effect_mock(1)
1
>>> side_effect_mock("hello world!")
'hello world!'
~~~
