---
layout: post
title:  "Python 的单元测试之 unittest"
categories: Python
---

----------------

# Overview

## Basic example

随着项目的不断扩大，单元测试在保证开发效率、可维护性和软件质量等方面越发举足轻重，是一本万例的举措。Python 常用 [unittest](https://docs.python.org/2/library/unittest.html) module 来编写单元测试，它包含四个重要概念：

- test fixture：初始化和清理测试环境，比如创建临时的数据库，文件和目录等等，其中 [setUp()](https://docs.python.org/2/library/unittest.html#unittest.TestCase.setUp) 和 [setDown()](https://docs.python.org/2/library/unittest.html#unittest.TestCase.tearDown) 是最常用的方法。
- test case：单元测试用例，[TestCase](https://docs.python.org/2/library/unittest.html#unittest.TestCase) 是编写单元测试用例最常用的类。
- test suite：单元测试用例的集合，[TestSuite](https://docs.python.org/2/library/unittest.html#unittest.TestSuite) 是最常用的类。
- test runner：执行单元测试

例如：

~~~ Python
import unittest

class TestStringMethods(unittest.TestCase):

  def test_upper(self):
      self.assertEqual('foo'.upper(), 'FOO')

  def test_isupper(self):
      self.assertTrue('FOO'.isupper())
      self.assertFalse('Foo'.isupper())

if __name__ == '__main__':
    unittest.main()
~~~

执行结果如下所示：

~~~ bash
$ python testdemo.py
..
----------------------------------------------------------------------
Ran 2 tests in 0.000s

OK
~~~

## Add fixture

[setUp()](https://docs.python.org/2/library/unittest.html#unittest.TestCase.setUp) 和 [setDown()](https://docs.python.org/2/library/unittest.html#unittest.TestCase.tearDown) 允许执行每个测试用例前分别初始化和清理测试环境，用法如下：

~~~ Python
import unittest

class TestStringMethods(unittest.TestCase):

    def setUp(self):
        # Do something to initial the test environment here.
        pass

    def tearDown(self):
        # Do something to clear the test environment here.
        pass

    def test_upper(self):
        self.assertEqual('foo'.upper(), 'FOO')

    def test_isupper(self):
        self.assertTrue('FOO'.isupper())
        self.assertFalse('Foo'.isupper())

if __name__ == '__main__':
    unittest.main()
~~~

## Igore some testcases

有时希望某些用例不被执行，unittest.skip() 提供了这种功能，用法如下：

~~~ Python
import unittest

class TestStringMethods(unittest.TestCase):

  def test_upper(self):
      self.assertEqual('foo'.upper(), 'FOO')

  @unittest.skip('skip is upper.')
  def test_isupper(self):
      self.assertTrue('FOO'.isupper())
      self.assertFalse('Foo'.isupper())

if __name__ == '__main__':
    unittest.main()
~~~

执行结果如下：

~~~ bash
$ python testdemo.py
test_isupper (testdemo.TestStringMethods) ... skipped 'skip is upper.'
test_upper (testdemo.TestStringMethods) ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.000s

OK (skipped=1)
~~~

----------------------

# Run your tests

## Command Line Interface

unittest 也提供了丰富的命令行入口，可以根据需要执行某些特定的用例。有了命令行的支持，上述例子的最后两行代码就显得冗余，应当被移除：

~~~ python
...

# 删除以下代码
if __name__ == '__main__':
    unittest.main()
~~~

执行 testdemo.py 文件的所有测试用例：

~~~ bash
$ python -m unittest testdemo
~~~

执行 testdemo.py 文件 TestStringMethods 类的所有测试用例：

~~ bash
$ python -m unittest test_demo.TestStringMethods
~~~

执行 testdemo.py 文件 TestStringMethods 的 test_upper 用例：

~~~
$ python -m unittest test_demo.TestStringMethods.test_upper
~~~


-------------------------

## Test Discovery

unittest 也提供自动匹配发现并执行测试用例的功能，随着项目代码结构的越发庞大，势必有多个测试文件，自动匹配发现并测试用例的功能在此就显得非常有用，只要满足 [load_tests protocol](https://docs.python.org/2/library/unittest.html#load-tests-protocol) 的测试代码都会被 unittest 发现并执行，测试用例文件的默认匹配规则为 test\*.py。**通过一条命令即可执行所有的测试用例，如此就很容易被 tox 等测试工具所集成**。使用如下：

~~~ bash
python -m unittest discover
~~~

参数如下：

~~~ bash
-v, --verbose
Verbose output

-s, --start-directory directory
Directory to start discovery (. default)

-p, --pattern pattern
Pattern to match test files (test*.py default)

-t, --top-level-directory directory
Top level directory of project (defaults to start directory)
~~~

假设现在要被测试的代码目录如下：

~~~ bash
$ tree demo
demo
├── testdemo.py
└── testdemo1.py
~~~

~~~ bash
$ python -m unittest discover -s demo -v
test_isupper (testdemo.TestStringMethods) ... ok
test_upper (testdemo.TestStringMethods) ... ok
test_is_not_primer (testdemo1.TestPrimerMethods) ... ok
test_is_primer (testdemo1.TestPrimerMethods) ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.001s

OK
~~~