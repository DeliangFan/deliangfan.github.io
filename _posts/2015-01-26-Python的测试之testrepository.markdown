---
layout: post
title:  "Python 的 testing ———— Tox"
categories: Python
---


Tox is a generic virtualenv management and test command line tool you can use for:

- checking your package installs correctly with different Python versions and interpreters
- running your tests in each of the environments, configuring your test tool of choice
- acting as a frontend to Continuous Integration servers, greatly reducing boilerplate and merging CI and shell-based testing.

------------------------

Current features¶

- automation of tedious Python related test activities
- test your Python package against many interpreter and dependency configs
- - automatic customizable (re)creation of virtualenv test environments
- - installs your setup.py based project into each virtual environment
- - test-tool agnostic: runs py.test, nose or unittests in a uniform manner
- (new in 2.0) plugin system to modify tox execution with simple hooks.
- uses pip and setuptools by default. Support for configuring the installer command through install_command=ARGV.
- cross-Python compatible: CPython-2.6, 2.7, 3.2 and higher, Jython and pypy.
- cross-platform: Windows and Unix style environments
- integrates with continuous integration servers like Jenkins (formerly known as Hudson) and helps you to avoid boilerplatish and platform-specific build-step hacks.
- full interoperability with devpi: is integrated with and is used for testing in the devpi system, a versatile pypi index server and release managing tool.
- driven by a simple ini-style config file
- documented examples and configuration
- concise reporting about tool invocations and configuration errors
- professionally supported
- supports using different / multiple PyPI index server