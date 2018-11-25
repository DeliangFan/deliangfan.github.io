---
layout: post
title:  "Python 的单元测试之 tox"
categories: Python
---

&nbsp;&nbsp;&nbsp;

![testtools](http://wsfdl.oss-cn-qingdao.aliyuncs.com/pythontesttools.png)

&nbsp;&nbsp;&nbsp;

---------------------

# Overview

[Tox](https://tox.readthedocs.org/en/latest/) 是标准的 virtualenv 管理器和命令行测试工具，它创造一个隔离的 Python 沙箱环境，根据配置下载安装依赖包，然后执行测试用例。

- 在不同的 Python 版本环境下检验 package 能否正确安装
- 在不同的 Python 版本环境下运行测试用例，检查编码规范，测试覆盖率等
- 和 Jenkins 等 CI server 集成，支持多个 Pypi Server

![pypitoxjenkins](http://wsfdl.oss-cn-qingdao.aliyuncs.com/toxwithjenkins.png)


---------------------

# Example

## tox-quickstart

Tox 依赖配置文件 tox.ini 构建测试环境，下面介绍如何编写 tox.ini，首先运行 tox-quickstart 生成一个基本的 tox.ini。

~~~ bash
$ tox-quickstart
Welcome to the Tox 2.1.1 quickstart utility.

...

What Python versions do you want to test against? Choices:
    [1] py27
    [2] py27, py33
    [3] (All versions) py26, py27, py32, py33, py34, py35, pypy, jython
    [4] Choose each one-by-one
> Enter the number of your choice [3]: 3

What command should be used to test your project -- examples:
    - py.test
    - python setup.py test
    - nosetests package.module
    - trial package.module
> Command to run to test project [{envpython} setup.py test]: python -m unittest testdemo

What extra dependencies do your tests have?
> Comma-separated list of dependencies [ ]:

Creating file tox.ini.
...
~~~

本文要执行的测试文件为 testdemo.py，生成后的 tox.ini 如下：

~~~ ini
[tox]
envlist = py26, py27, py32, py33, py34, py35, pypy, jython

[testenv]
commands = python -m unittest testdemo
deps =
~~~

可指定 python 版本执行测试用例， 如在 py27 下执行用例：

~~~
tox -epy27
~~~

## Config dependency

Tox 支持多种设置 dependency 的方法，如在 setup.py，tox.ini 和 requirements.txt 均可设置。

- setup.py 设置 dependency 的样例：

~~~ python
setuptools.setup(
    ...
    install_requires=(
        'python-ldap',
        'ldappool'
    ),
    ...
)
~~~

- 在 tox.ini 设置如下：

~~~ ini
[testenv]
deps=
    python-ldap4
    ldappool
~~~

- 在 requirement.txt 设置如下：

~~~
$ cat requirement.txt
python-ldap4
ldappool
~~~

同时需设置 tox.ini：

~~~
[testenv]
deps = -rrequirements.txt
~~~

## Config a different Pypi server

Tox 默认的 Pypi server 为 [http://pypi.python.org/simple](http://pypi.python.org/simple)，我们可以为 tox 指定某个 Pypi server:

~~~ ini
[tox]
indexserver =
    another = http://another-pypi.com
~~~

Tox 允许指定设置多个 Pypi server:

~~~ ini
[tox]
indexserver =
    default = http://default-pypi.com
    dev = http://dev-pypi.com
~~~

## Add pep8 checker

Tox 支持为项目项设置代码规范检查，样例如下：

~~~ ini
[testenv:pep8]
commands =
  flake8

[flake8]
# H405: multi line docstring summary not separated with an empty line
ignore = H405
show-source = True
exclude = .venv,.tox,dist,doc,*egg,build,
~~~

执行：

~~~
tox -epep8
~~~

## Add coverage

为项目生成测试覆盖率报表，样例如下：

~~~ ini
[testenv:cover]
commands = python setup.py testr --coverage --testr-args='{posargs}'
~~~

执行：

~~~
tox -ecover
~~~

## Config Jenkens

Tox 另外一个很 nice 的特性就是可以和 Jenkins 等 CI Server 集成，[Using Tox with the Jenkins Integration Server](https://tox.readthedocs.org/en/latest/example/jenkins.html) 详细的介绍了集成步骤。
