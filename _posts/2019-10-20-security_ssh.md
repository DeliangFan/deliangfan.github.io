---
layout: post
title:  "K8S 下容器的登陆、安全等演进"
categories: Kubernetes
---


早期的 K8S 认证和授权功能较弱，通过在 K8S 之上增加一层权限管理以适配公司的安全体系。为了增强容器的安全性，还把运行在虚拟机上的安全审计 agent 移植到容器中，为了支持用户登录，容器中还运行 SSHD 进程。这种方式较好的兼容现有安全体系，但是不符合 PaaS 的原生玩法，期间还引入不少问题，增加维护成本，故在较新的版本对安全进行原生改造。本文结合经验重点谈谈在保证原生玩法的前提下，如何衔接好公司已有的权限管理系统。


## SSH 登陆

首先讨论一个问题：“用户需不要登陆容器？”。持反对意见者认为 SSH 带来权限和管理的复杂性，应当引导用户基于日志定位问题。从实际来看，日志能定位大部分问题，但是涉及性能和运行时问题需要登陆容器获取更多的信息，所以我认为 PaaS 平台需要支持用户登录容器。

虚拟机的 SSH 登陆认证或通过堡垒机，或由 SSHD 对接如 LDAP 等认证系统完成。在单进程容器下，用户登录的方法只有通过 kubectl exec，这就导致认证的功能势必转移到 K8S 的 apiserver。所幸的是 K8S 已支持集成丰富的认证策略，如 OpenID 协议，LDAP 等公司常见的认证体系，通过对接这些认证系统，进而支持登录认证。

```
     token          +----------------+              +-----------+
 kubectl exec -->   |    apiserver   | --> ... -->  | container |
                    |   authenticate |              +-----------+
                    +----------------+
                            | ^
                            v |
                  +-------------------+
                  | Identity Provider |
                  |  (OpenID/LDAP...) |
                  +-------------------+

```

K8S 还提供多种授权功能，其中 RBAC 是常见的授权策略，通过赋予用户在某个 namespace 的角色，从而实现用户的权限管理。K8S 的 Admission control 进一步扩展了处理用户请求的能力，通过拦截用户的请求，可以实现更深层次的认证和审计功能。

## 认证 & 授权 & 准入控制

Kubectl exec 本质上是 K8S 的 API，本节从 API 的角度谈谈 K8S 的认证、授权和准入控制如何和现有的权限体系对接，源自官网的下图展示用户访问 apiserver 的内部步骤，首先是认证，其次是权限校验，再次是准入控制，最后才进入业务逻辑处理。

![官网](https://d33wubrfki0l68.cloudfront.net/673dbafd771491a080c02c6de3fdd41b09623c90/50100/images/docs/admin/access-control-overview.svg)

### 认证(Authentication)

K8S 中的用户分为两种，Service Account 和 User Account，其中 Service Account 用于 Pod 内部 Process 访问 apiserver，由 K8S 存储和管理。User Account 是本文重点关注的用户类型，由外部认证系统存储和管理。K8S 内置如下数种简单的[认证策略](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)，这些策略或用于内部服务，或用于非生产环境。

- X509 Client Certs
- Static Token File
- Bootstrap Tokens
- Static Password File
- Service Account Tokens

生产环境下的 K8S 需要和现有的认证系统打通，K8S 支持对接满足 [OpenID](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) 协议的认证系统，官网已做详细介绍，此处不再累述。K8S 无法直接对接 LDAP，它需要相关插件(如 Keycloak, Dex 等)完成对接工作。[Dex](https://github.com/dexidp/dex#connectors) 是开源支持 OpenID 协议的认证适配器，K8S 通过 OpenID 协议和 Dex 对接，而 Dex 适配常见的 OpenID, LDAP, SAML 等协议的认证系统。如下是基于 JWT 的 token 认证方式，用户首先从认证系统获取 token，然后 kubectl 携带此 token 访问 apiserver，由于 apiserver 配置了签发 JWT token 的公钥，所以无需向认证系统验证 token 的有效性，如果 token 符合要求，则认证成功。

![OpenID](https://d33wubrfki0l68.cloudfront.net/d65bee40cabcf886c89d1015334555540d38f12e/c6a46/images/docs/admin/k8s_oidc_login.svg)

### 授权(Authorization)

所谓的授权，是指赋予用户在某个 namespace  对某种资源具有某种操作权限。K8S 支持 role 和 rolebinding 资源，意味着它具备授权功能，由于 namespace，资源均为 K8S 内部概念，和外部权限管理系统无关，所以我认为授权功能完全由 K8S 完成，几乎不需要依赖外部的权限管理系统。目前支持的授权模式主要有以下几种：

- Node Authorization
- ABAC Authorization
- RBAC Authorization
- Webhook Authorization

[RBAC](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/) 是业务主流的权限管理系统，而且可以对用户的权限进行灵活的配置，所以建议采用 RBAC 策略，该策略下有如下四个重要概念：

- Role：指某个 namespace 下的角色
- ClusterRole：全局的角色
- RoleBinding：授予用户操作某个 namespace 中某种资源的权限
- ClusterRoleBinding：授予用户操作所有 namespace 中某种资源的权限

公司的技术体系是面向应用，每个应用关联机器、DB 和域名等资源，开发人员和应用挂钩。通过为每个应用分配不同的 namespace，并赋予相应的应用负责人操作权限，从而实现授权功能。如此用户只能增删改查属于自己 namespace 下的资源，包括登陆容器。K8S 内置了非常多种 role，但是在实际场景中，只需要创建少数几种 role，授予普通用户增删改查 pod，deployment，statefulset，job，configmap 等普通资源的权限即可。

### 准入控制(Admission Controller)

当 API 请求通过认证和授权模块后，便进入准入控制模块。准入控制大大的扩展处理 API 请求的灵活性，比如根据某些条件拒绝请求，修改请求中的参数等等。K8S 内置了[一些准入控制插件](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#%E6%AF%8F%E4%B8%AA%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6%E5%99%A8%E7%9A%84%E4%BD%9C%E7%94%A8%E6%98%AF%E4%BB%80%E4%B9%88)，如大家耳熟能详的 [LimitRanger](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#limitranger)，[NamespaceLifecycle](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#namespacelifecycle)，[ResourceQuota](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#resourcequota)，此外它还支持以 [Webhook](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/) 的方式扩展能力。从类型角度准入控制器 webhook 可以分为 MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook 两类，前者通常用来修改请求的某些字段，后者用于验证请求的合法性。

Admission Controller 的 webhook 模块解耦了 K8S 和业务处理模块，大大扩展处理请求的能力。比如对 API 统计功能可以放置到 webhook 中，比如仅允许从特定的仓库中拉取镜像，比如为 Pod 添加挂载默认一些卷，添加 sidecar 等等。

## 其它模块

PaaS 安全相关涉及多个模块，从容器隔离性，privilege 权限管理，镜像安全，端口的管理，HTTPS 和证书，乃至 K8S 和 Docker 的漏洞缺陷。安全是一个长久的话题，没有最好，只有更好，但是也要兼顾投入和产出比，以及用户体验和功能。

容器较弱的隔离性基本能为私有云接受，在公有云上通过独占宿主机的方式规避。由于不少业务依赖 privilege 权限运行，比如 service mesh 的 sidecar，通常需要开启 privilege 权限，同时建议把容器镜像的危险命令，如 reboot, shutdown 等等清除。[33 个 Kubernetes 安全工具](https://blog.fleeto.us/post/33-kubernetes-security-tools/) 系统的介绍相关工具，比如采用 Anchore, [Clair](https://coreos.com/clair/docs/latest/) 构建时扫描镜像文件，Kube-Hunter 查找集群中的安全弱点。
