---
layout: post
title:  "如何编写相对标准的后端项目 （一）组织与运行"
categories: Architecture
---

本人接触过数个 Open Source 项目，如 OpenStack／Kubernetes 等，深感这些优秀的开源项目存在着一些共性，如：美观的代码，完整的测试，设计理念，框架和架构等等。一般来说，遵循这些优良原则的项目在易读性，可维护性，特别是（功能和规模的）可扩展性会更强些。

本文探讨性的梳理部分开源项目某些共同的优秀之处，但是软件工程领域，不同业务／语言的项目千差万别，故“标准”二字也是相对的，更适合于 golang/python 语言编写的 LAM（Linux／Apache／Mysql）后端项目，争议之处请多包涵。

## 目录组织与命名

### 目录组织

不同语言下的目录组织可能会有很大的差异，但是都遵循以下共识：

- 语义贴切：文件／目录的命名要尽可能表达其功能／用途
- 集中原则：功能相近的文件／目录应集中存放

例如：

```
your_project /
|-- contrib /        # 存放一些如 rpm.spec，脚本等等
|-- doc /            # 文档相关
|-- examples /       # 样例说明
|-- etc(or conf) /   # 配置文件
|-- src /
    |-- api /               # api 相关代码
    |-- cmd(or cli) /       # command 相关代码，
    |-- db /                # database 相关代码
    |-- rpc /               # rpc 相关代码
    |-- tests /             # 测试文件目录
    |-- utils(or common) /  # 一些公共的方法
    |-- ...                 # 其它目录/文件
|-- README           # 项目说明
|-- .gitigore        # 忽略的 git 文件和目录
|-- ...              # 其它目录／文件
```

以下对部分目录做详细说明

- api：存放 api 相关代码，比如 route，filter，middleware，serialize/de-serialize 等。
- contrib：存放一些编包的文件，git 的 patch 文件，systemd 的配置文件，以及脚本等等。
- db：存放 database(如：sql/nosql) 相关代码，包括 model, connection, session，orm/sql。是代码层面访问 db 的唯一入口，建议所有访问 database 的操作都必须在代码层面经过 db 目录下的代码。
- utils：一般存放一些公共的方法，如 time／date，encoding／decoding，exec，regex，uuid，string／json 等等。

推荐阅读：

- [What is the best project structure for a Python application?](https://stackoverflow.com/questions/193161/what-is-the-best-project-structure-for-a-python-application)
- [What is a sensible way to layout a Go project](https://stackoverflow.com/questions/14867452/what-is-a-sensible-way-to-layout-a-go-project)

### 命名

匈牙利命名法和 unix 命名法是常见的命名风格，一般来说，c 和 python 以 unix 命名法居多，golang 和 java 以匈牙利命名法居多。编程过程中，遵循一致的命名风格，会使得代码更为美观，可读性强。

编程中贴切的缩写常见单词，可使得代码更为简洁美观，但是缩写最好应遵循某些约定成俗，避免产生歧义。

一般情况下不建议采用拼音命名和拼音缩写。

推荐阅读：

- [Google Python Style Guides](https://google.github.io/styleguide/pyguide.html)
- [Google Shell Style Guide](https://google.github.io/styleguide/shell.xml)
- [一个查单词缩写的网站](http://www.abbreviations.com)


## 发布与运行

为什么应用的发布和运行也应该遵循一定的规则呢？首先，用户体验更好。比如，软件工程师习惯性在 /etc 寻找配置文件，/var/log 寻找日志文件。其次，遵循这些规则有利于被运维软件管理，如 ansible，chef 等等。最后，linux 及相关软件在设计时就已经遵循这些规则，如果你的应用不遵循，可能会产生某些冲突。 

### 版本号

发布版本时，版本号的命名需要遵循某种规则，其中 Semantic Versioning 2.0.0 定义了一套简单的规则。重点如下：
版本号的格式为 X.Y.Z(又称 Major.Minor.Patch)，递增的规则为：

- X 表示主版本号，当 API 的兼容性变化时，X 需递增。
- Y 表示次版本号，当增加功能时(不影响 API 的兼容性)，Y 需递增。
- Z 表示修订号，当做 Bug 修复时(不影响 API 的兼容性)，Z 需递增。

1. X, Y, Z 必须为非负整数，且不得包含前导零，必须按数值递增，如 1.9.0 -> 1.10.0 -> 1.11.0
2. 当 API 的兼容性变化时，X 必须递增，Y 和 Z 同时设置为 0；当新增功能(不影响 API 的兼容性)或者 API 被标记为 Deprecated 时，Y 必须递增，同时 Z 设置为 0；当进行 bug fix 时，Z 必须递增。
3. 版本一经发布，不得修改其内容，任何修改必须在新版本发布！

推荐阅读：

- [Semantic Versioning 2.0.0](https://semver.org/)

### 打包

不同语言的应用可能有不同的打包和发布规范，如 java 的 jar 包，python 的 wheel 包，它们极大的简化了应用的安装／升级／管理等等，非常方便的解决了版本和依赖问题。但是上述的打包方式在通用性上有所欠缺，只能被特定语言的工具管理，比如 jar 包需要被 maven 等管理，wheel 包需要 pip 等管理。为此 Redhat 定义了 [redhat package management]()(即 rpm 包)，所有语言的应用都可以制作成 rpm 包，然后用 rpm 和 yum 等常用工具进行管理。与此类似，ubuntu 也定义了 debian 包。

个人认为，应用最好以软件包的形式发布，避免直接拷贝源码或者二进制文件到线上。

推荐阅读：

- [How to create an RPM package](https://fedoraproject.org/wikiHow_to_create_an_RPM_package/zh-cn)
- [Python application 的打包和发布](http://wsfdl.com/python/2015/09/06/Python%E5%BA%94%E7%94%A8%E7%9A%84%E6%89%93%E5%8C%85%E5%92%8C%E5%8F%91%E5%B8%83%E4%B8%8A.html)

### 运行

一个应用可能包涵配置文件，代码库，可执行文件等等，这些文件的存放也所讲究，一般来说：

- 可执行文件应存放于 /usr/bin 或者 /usr/local/bin 目录下
- 代码或者库应放于  /usr/lib 或者 /var/lib 目录下
- 配置文件应存放于 /etc/your_project/ 目录下
- 日志文件应存放于 /var/log/your_project 目录下
- systemd 相关脚本应该放置于 /usr/lib/systemd/system/ 目录下
- logrotate 相关脚本应该放置于 /etc/logrotate.d/ 目录下

个人不建议把应用放在 /home 目录下。

## 测试

测试可分为单元测试／集成测试／性能测试。需不需要测试，需要哪些测试，测试覆盖率为多少和很多因素有关，个人认为：

- 代码越多，测试越重要
- 动态语言，测试越重要
- 开发人员越多，测试越重要

从经验上看，很多项目一般都会有集成测试，但是很少有单元测试。这里想谈谈单元测试的重要性，它很大程度上保证了新加的代码不会影响原有的逻辑。首先这种代码级别首先更为精细，能够针对性的覆盖重要功能代码。其次，单元测试的运行一般不依赖具体环境，能够随时编写随时测试。再次，单元测试能够在一定程度上维护代码的结构。最后，都说动态一时爽，重构火葬场，而完整的单元测试是避免火葬场的重要手段。

很多语言都有测试框架和工具，这些工具功能强大，如：运行测试，生成测试报告，代码风格规范检查，集成 jenkins 等等。建议根据需要选择合适的测试框架。

推荐阅读：

- [tox vision: standardize testing in Python](https://tox.readthedocs.io/en/latest/)
- [谈谈开发中单元测试的重要性](http://ufdouble.com/2016/07/%E8%B0%88%E8%B0%88%E5%BC%80%E5%8F%91%E4%B8%AD%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E7%9A%84%E9%87%8D%E8%A6%81%E6%80%A7/)
