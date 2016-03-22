---
layout: post
title:  "Python 的单元测试之 mock"
categories: Python
---

&nbsp;&nbsp;&nbsp;

![testtools](http://7xp2eu.com1.z0.glb.clouddn.com/pythontesttools.png)

&nbsp;&nbsp;&nbsp;

---------------------

# Overview

[mock](http://www.voidspace.org.uk/python/mock/index.html) 是一个用于单元测试的 Python 库，它使用 mock 替代系统中如 class, method 等部分，模拟被替代部分的功能，并且断言它们是如何被调用的。在编写单元测试时，mock 非常适合模拟数据库，web 服务器等。本文是 mock 的入门篇，主要介绍 mock 的基本用法。

除了 mock 外，Python 还有许多其它 mocking 库，[Python Mock Library Comparison](http://garybernhardt.github.io/python-mock-comparison/) 在用法上对这些库做了简单的比较，其中 OpenStack 单元测试广泛的使用了 mock 和 mox。

- [mock](http://www.voidspace.org.uk/python/mock/)
- [flexmock](https://pypi.python.org/pypi/flexmock)
- [mox](https://pypi.python.org/pypi/mox)
- [Mocker](http://niemeyer.net/mocker)
- [Dingus](https://pypi.python.org/pypi/dingus)
- [fudge](http://farmdev.com/projects/fudge/)

mock 的安装非常简便：

~~~ bash
pip install mock
~~~

---------

# Mock Patching Methods

http://stackoverflow.com/questions/17181687/mock-vs-magicmock

当使用 mock 模拟 methods 时，mock 会替换被模拟的 methods，并且记录调用详情。

~~~ bash
>>> class Foo(object):
...     def echo(*args):
...         return "hello"
...
>>>
>>> foo = Foo()
>>> foo.echo = mock.MagicMock()
>>> foo.echo.return_value = "mock value"
>>> foo.echo()
'mock value'
>>> foo.echo(1, 2)
'mock value'
~~~

被模拟后，foo.echo 的类型是一个名为 mock.MagicMock 类，

~~~ bash
>>> type(foo.echo)
<class 'mock.MagicMock'>

>>> dir(foo.echo)
['assert_any_call', 'assert_called_once_with', 'assert_called_with', 'assert_has_calls', 'attach_mock', 'call_args', 'call_args_list', 'call_count', 'called', 'configure_mock', 'method_calls', 'mock_add_spec', 'mock_calls', 'reset_mock', 'return_value', 'side_effect']
~~~


~~~ bash
>>> foo.echo.called
True
>>> foo.echo.call_count
2
>>> foo.echo.mock_calls
[call(), call(1, 2)]
~~~~


~~~ bash
>>> foo.echo.assert_called_with(1, 2)
>>> foo.echo.assert_called_with(1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python2.7/dist-packages/mock.py", line 844, in assert_called_with
    raise AssertionError(msg)
AssertionError: Expected call: mock(1)
Actual call: mock(1, 2)
~~~