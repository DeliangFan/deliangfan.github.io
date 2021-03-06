---
layout: post
title:  "如何编写相对标准的后端项目（二）设计 Restful API"
categories: Architecture
---



本文主要介绍 Restful 风格，如何设计 HTTP Restful API，以及在设计过程中的一点经验之谈。文章有点枯燥，敬请谅解。最后建议在设计 HTTP API 时，遵循 Restful API 风格。

## 理解 Restful Style

在 web 飞速发展的环境下，Roy Fielding 于 2000 年在博士论文 [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) 提出 **R****E**presentational **S**tate **T**ransfer ([REST](https://en.wikipedia.org/wiki/Representational_state_transfer)) 概念，它倡导一种新的 web 架构风格，具有面向资源、松耦合、无状态、易扩展等特点，如今被广泛的应用。


那么什么是 REST 风格呢，论文的第五章节 [Representational State Transfer (REST)](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) 做了详细说明，其中 uniform interface 是 REST 架构的最主要特征。

> The central feature that distinguishes the REST architectural style from other network-based styles is its emphasis on a uniform interface between components.......
>
> In order to obtain a uniform interface, multiple architectural constraints are needed to guide the behavior of components. REST is defined by four interface constraints: identification of resources; manipulation of resources through representations; self-descriptive messages; and, hypermedia as the engine of application state.

注意前两个约束：identification of resources 和 manipulation of resources through representations，它们表明 uniform interface(即 RESTful API) 的主体是 resource，即 REST 是面向 resource 的。那么什么是 resource 呢？任何能够被命名的事物都可称为 resource，比如某个文本、图像，甚至某种服务。在 web 中，我们采用 URI 来指代某个 resource。顾名思义，representation 就是资源的表现形式，比如图像资源的表现形式可为 JPEG image，文本资源的表现可为 text。

那么第三个约束 self-descriptive messages 表示什么呢？论文是这样解释的，即采用 standard methods(如 HTTP Method) 和 media types(如 HTTP Content type) 来表示操作类型，举例来说，我们可以用 GET 方法表示查询某个资源，用 DELETE 方法表示删除某个资源。

第四个约束 hypermedia as the engine of application state 表示 application 的状态由 request 决定，即客户端通过发送 request 来改变 application 的状态。以 HTTP POST 为例，客户端可以新建一个 resource，因而改变了 application 的状态。

总结而言，uniform interface 的约束条件定义了 web application API 的风格，即 RESTful API，这种 API 是面向 resource 的，利用(HTTP)标准方法来描述操作，客户端通过 representation 来操作 resource，从而转化 resource 的状态。

### 推荐阅读

-  [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)


## 如何设计 Restful API

上节简单的介绍了 Restful 的核心特征，本文从实践角度出发，围绕 URI，HTTP Method，Response Code，序列化和安全等，并结合一个具体的案例，谈谈 Http Restful API 设计的经验。

比如某个账号系统（http://example.com），通常需要增删改查用户，我们就以增删改查用户(user)为例，详细的介绍Restful API 的设计。

### URI 设计

作为 Restful 的核心特性，uniform interface 是面向资源的，我们用 URI 来指代某个资源。比如，我们用如下 URI 代表账号系统用户 john：

```
http://example.com/users/john
为了简便，后续例子可能会省略域名，用 /users/john 表达相同含义。
```

查询用户 john 的请求如下。

```
GET /users/john
```

更新用户 john 的请求如下。

```
PUT /users/john
```

删除用户 john 的请求如下。

```
DELETE /users/john
```

从上可以看出，不论是查询还是删除用户 john，我们都用相同的 URI 来表指代这个用户(资源)，采用不同的 Http Method 来表示不同的操作类型。既然 /users/john 指代的是用户 john，顾名思义，/users 指代的是所有的用户，所以可以用如下请求查询所有的用户信息。

```
GET /users
```

因为 URI 代表的是某个资源，而资源本身是个名词，所以在设计 URI 时，也应该用名词，切记不要使用动词等词语。比如，如下 URI 是面向动作的，不满足 Restful 风格约束。

```
/users/getuser
```

帐户系统在不同阶段可能会有不同的版本，比如 v1，v2，我们通常把版本号放在前面，比如：

```
/v1/users/john
/v2/users/john
```

如果需要查询某些符合要求的资源，可在 URI 的 paras 模块设置查询参数，如查询所有性别为女的用户

```
GET /users?gender=female
```

如果返回结果过多，可以加上分页功能，如查询自 john 起的 10 位用户。

```
GET /users?limit=10&marker=john
```

此外，在设计 uri 时，还应该注意如下要点：

- 尽量用小写：比如用 john，最好不要用 John。
- 用中杠 -，不用下杠 _



### HTTP Method

HTTP 协议定义了 8 个 Method，在 Restful API 设计中，我们常用 POST，DELETE，PUT，GET 四方法来操作资源。其中 POST 表示创建资源，DELETE 表示删除某个资源，PUT 表示更新某个资源，GET 表示查询资源。例如，当我们要创建用户 andy 时，其 API 如下：

```
POST /uesrs
Body: {"user": "andy", ...}
```

在设计 Restful API 时，应该遵循 HTTP 关于 “安全性” 和 “幂等性” 的要求。

- 安全性：无论请求多少次，都不会改变资源的状态。比如 GET 操作，无论执行多少遍，都不会改变资源的状态。所以对于 GET 类 API，在编写应用端代码时，切记要尽可能避免出现删除或者更新数据的逻辑。
- 幂等性：无论是执行一次，还是执行多次，效果是等价的，比如 DELETE，PUT 操作。以 DELETE 操作为例，删一次和删多次，效果都是该资源不存在了。

在 HTTP 协议，不同 Method 在 “是否有 request body” 和 “是否有 response body” 有所差异。总结如下表：

| Method | 安全性 | 幂等性 | 是否有 request body| 是否有 response body
---- | --- | ---- | --- | --- |
POST | 否 | 否 | 是 | 是 
DELETE|否 | 是 | 否 | 最好无
PUT | 否 | 是 | 是 | Option
GET | 是 | 是 | 否 | 是


### Response Code

之所以着重谈谈 Response Code，是因为看到部分同学在设计 API 的时候，常常自己定义 Response Code。例如，查询某个不存在的用户 tom 时，其 Response 为：

```
GET /users/tom

Response Code: 200 OK
Body: {"error": {"message": "User tom not found.", "code": 100}}
```
在上述查询某个不存在的资源的例子中，其 Response Code 为 200，此外，还在 Body 中定义了代号为 100 的返回码。事实上，这种做法非常容易给用户造成困惑：首先，200 表示查询成功，而本例的查询结果却是失败的；再次，自定义的 100 的含义是什么？如果没有相关说明文档，用户是不得而知的。所以，清晰明了的返回值应该设计为如下：

```
Response Code: 404 Not Found
Body: {"error": {"message": "User tom not found."}}
```

设计 API 时，请先深入理解 HTTP Response Code，充分利用它们代表的含义，除非特别需要，尽量不要去自己定义额外的返回码，以免引起误解。让我们一起回顾下常见 HTTP Response Code 的含义。

请求成功：

- 200 OK: 请求已成功，Body 有返回内容。多用作 GET Method 的 API 的返回码。
- 201 Created: 请求已经被实现，资源被创建。多用作 POST Method 的同步类型 API 的返回码。
- 202 Accepted: 服务器已接受请求，但尚未处理。多用作 POST Method 异步类型 API 的返回码。
- 204 No Content: 服务器成功处理了请求，没有返回任何内容。用多于 DELETE／PUT Method 的 API 的返回码。

因客户端原因导致请求失败：

- 400 Bad Request: 如参数错误，格式错误
- 401 Unauthorized: 用户未被认证，如用密码错误，证书错误
- 403 Forbidden: 用户权限不够
- 404 Not Found: 服务端无此资源。通常为 URL 不存在，或者某个 Method 不存在 
- 409 Conflict: 请求存在冲突无法处理该请求

因服务端原因导致请求失败：

- 500 Internal Server Error: 服务端错误消息，服务器遇到了一个未曾预料的状况。这是最常用的服务端错误代码
- 501 Not Implemented: 服务器不支持当前请求所需要的某个功能
- 503 Service Unavailable: 如服务器维护或者过载等

### 编码

ASCII 编码计算机最早采用的字符编码，随着计算机在多个国家普及，几乎每个语言都有一套或者多套自己的编码 ，如汉字的 GBK 编码，日语的 Shift_JIS 编码。这些编码方案各不相同，非常容易造成兼容性问题。Unicode 的诞生很好的解决了上述问题，它使计算机能够跨语言、跨平台进行文本转换和处理。Unicode 字符集包含了世界上所有文字和符号，它有 utf-8，utf-16 等编码方案，其中 utf-8 是最为常用的编码方案。从个人经验而言，除非有特殊要求：

```
从 API 入口，到业务逻辑处理，再到存储，一律使用 utf-8 编码！
从 API 入口，到业务逻辑处理，再到存储，一律使用 utf-8 编码！
从 API 入口，到业务逻辑处理，再到存储，一律使用 utf-8 编码！

如果要支持 emoji 表情，还需额外的处理。
```

在 HTTP 协议中，Accept-Charset／Content-Type 头部可以指定字符编码方案。

- Requst 头部: Accept-Charset: utf-8
- Response 头部: Content-Type: charset=utf-8

### 序列化


在通信过程中，我们常常需要把应用层的对象转换成一段连续的二进制串，或者反过来，把二进制串转换成应用层的对象——这两个功能就是序列化和反序列化。XML／SOAP／JSON 等都是序列化和反序列化协议，其中 XML 多用于 html 类型的应用，而 JSON 成为 html 应用以外的主流序列化和反序列化协议。从个人经验而言，除非特殊需要，建议：

```
采用 JSON 作为序列化和反序列化协议，对于 html 类型应用，使用 XML。 
```

在 HTTP 协议中，Accept／Content-Type 头部可以指定序列化和反序列化协议。

- Requst 头部: Accept: application/json
- Response 头部: Content-Type: application/json


### HTTPS

出于安全的因素，此处普及下 HTTPS。HTTPS 由网景公司创建，基于 SSL(Secure Socket Layer) 或 TLS(Transport Layer Security)，它作用有两点：一是建立一个信息安全通道，保障报文在各类信道中安全的传输；二是保证网站的真实性。

HTTPS 默认的端口是 443，如果采用默认端口，可以省略 443。例如：

```
https://example.com/users/john
```

一般来说，常见的 HTTP Server 都支持 HTTPS 功能，经过简单的配置即可支持 HTTPS。对开发者来说，你需要：

- 选择支持 HTTPS 协议的 Web Server
- 知道如何配置证书。

机构颁发的证书一般需要万把块钱购买；你也可以使用 openssl 自己制作证书。如果你对 HTTPS 原理感兴趣，建议了解下 RSA／DH 算法，对称加密／非对称加密，数字签名，还有证书的含义和原理。

### 推荐阅读

- [HTTP Method](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.7/)
- [跟着 Github 学习 Restful HTTP API 设计](http://cizixs.com/2016/12/12/restful-api-design-guide)
- [Learn REST: A RESTful Tutorial](http://www.restapitutorial.com/)
- HTTP 权威指南
