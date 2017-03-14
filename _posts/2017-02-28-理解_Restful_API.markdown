---
layout: post
title:  "理解 REST 风格"
categories: Architecture
---

# Introduction

**R****E**presentational **S**tate **T**ransfer ([REST](https://en.wikipedia.org/wiki/Representational_state_transfer)) 概念由 Roy Fielding 于 2000 年在博士论文 [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) 提出，它倡导一种新的 web 架构风格，具有面向资源、松耦合、无状态、易扩展等特点，如今被广泛地应用在各类 web 服务中。  

回顾下[万维网](https://zh.wikipedia.org/zh-hans/%E4%B8%87%E7%BB%B4%E7%BD%91)的发展历史，在上世纪 90 年代左右，[Tim Berners-Lee](https://zh.wikipedia.org/wiki/%E8%92%82%E5%A7%86%C2%B7%E4%BC%AF%E7%BA%B3%E6%96%AF-%E6%9D%8E) 提出关于维网的建议，1993 年诞生的 [Mosaic](https://zh.wikipedia.org/wiki/Mosaic) 浏览器极大地促进万维网的发展，各类 web 网站如雨后春笋般冒出来。在此之前，多数软件以单机版的形式存在，随着 web 的发展，越来越多的软件通过网络提供给用户，即 network-based application software。因此很有必要为基于网络的软件定义一种架构风格，从而得到一个功能强大、高性能的通信架构，正如 Field 在其论文所说：

> Software architecture research investigates methods for determining how best to partition a system, how components identify and communicate with each other, how information is communicated, how elements of a system can evolve independently, and how all of the above can be described using formal and informal notations. My work is motivated by the desire to understand and evaluate the architectural design of network-based application software through principled use of architectural constraints, thereby obtaining the functional, performance, and social properties desired of an architecture. An architectural style is a named, coordinated set of architectural constraints.


# What is REST


那么什么是 REST 风格呢，Fielding 在论文第五章节 [Representational State Transfer (REST)](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) 做了详细说明，一个 REST 风格的 web 架构需要满足以下条件：

- Client-Server: 客户端和服务端之间解耦，目前主流的 web 框架都是 client-server 形式，以获得更好的 portability, scalability。
- Stateless: 客户端和服务端之间的通信无状态，服务端不会记录客户端的状态。每当客户端访问服务端时，都要在请求中提供充足的信息；如果需要记录状态，必须是客户端保存会话，从而获得更好的 visibility, reliability, scalability。
- Cache: 客户端和服务端之间可接入 cache，减轻服务端负载，以获得更好的 efficiency，scalability。
- Uniform Interface: 各个组件(如：client, proxy, gateway, server) 之间采用统一接口，这是 REST 的最大特点(详细介绍请见下节)，以获得更好的 simplicity, visibility，。
- Layered System: 引入代理、负载均衡等实现层级架构，以获得更好的 scalability.
- Code-On-Demand(optional): 允许客户端从服务端下载脚本以扩展功能。

一图蔽之，REST 风格的 web 架构抽象如下：

![Restful instance](http://7xp2eu.com1.z0.glb.clouddn.com/Restful%20insance.png)

下图是一个基于 REST 风格具体的 web 架构，不难发现，当今很多 web 架构都如下图所示。

![Restful](http://7xp2eu.com1.z0.glb.clouddn.com/RESTful.png)

# Resource & Representation & RESTful API

正如论文中所言，uniform interface 是 REST 架构的最主要特征。

> The central feature that distinguishes the REST architectural style from other network-based styles is its emphasis on a uniform interface between components.......
> 
> In order to obtain a uniform interface, multiple architectural constraints are needed to guide the behavior of components. REST is defined by four interface constraints: identification of resources; manipulation of resources through representations; self-descriptive messages; and, hypermedia as the engine of application state. 

注意前两个约束：identification of resources 和 manipulation of resources through representations，它们表明 uniform interface(即 RESTful API) 的主体是 resource，即 REST 是面向 resource 的。那么什么是 resource 呢？任何能够被命名的事物都可称为 resource，比如某个文本、图像，甚至某种服务。在 web 中，我们采用 URI 来指代某个 resource。顾名思义，representation 就是资源的表现形式，比如图像资源的表现形式可为 JPEG image，文本资源的表现可为 text。

那么第三个约束 self-descriptive messages 表示什么呢？论文是这样解释的，即采用 standard methods(如 HTTP Method) 和 media types(如 HTTP Content type) 来表示操作类型，举例来说，我们可以用 GET 方法表示查询某个资源，用 DELETE 方法表示删除某个资源。

> self-descriptive: interaction is stateless between requests, standard methods and media types are used to indicate semantics and exchange information, and responses explicitly indicate cacheability.

第四个约束 hypermedia as the engine of application state 表示 application 的状态由 request 决定，即客户端通过发送 request 来改变 application 的状态。以 HTTP POST 为例，客户端可以新建一个 resource，因而改变了 application 的状态；但是并非所有的请求都会改变 application 的状态，比如 HTTP GET 查询资源并不会改变 application 的状态。

> An application's state is therefore defined by its pending requests, ...... The application state is controlled and stored by the user agent and can be composed of representations from multiple servers. 

总结而言，uniform interface 的约束条件定义了 web application API 的风格，即 RESTful API，这种 API 是面向 resource 的，利用(HTTP)标准方法来描述操作，客户端通过 representation 来操作 resource，从而转化 resource 的状态。