---
layout: post
title:  "Python 的单元测试之 tox"
categories: Python
---

# Overview

[Tox](https://tox.readthedocs.org/en/latest/) 是个标准的virtualenv 管理器和命令行测试工具，提供了标准化的测试。它创造了一个隔离的 Python 沙箱环境，根据配置下载安装依赖包，然后执行测试用例。

- 在不同的 Python 版本环境下检验 package 能否正确安装
- 在不同的 Python 版本环境下运行测试用例，检查编码规范，测试覆盖率等
- 和 Jenkins 等 CI server 集成，支持多个 Pypi Server


