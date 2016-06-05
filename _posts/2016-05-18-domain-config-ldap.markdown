---
layout: post
title:  "Multiple Domain and LDAP"
categories: OpenStack
---

---------------

# Theory

Keystone 从 E 版本开始支持 LDAP，为 OpenStack 进军企业迈出一大步，[《Keystone 集成 LDAP》](http://wsfdl.com/openstack/2016/01/13/Keystone%E9%9B%86%E6%88%90LDAP.html) 介绍了如何配置 Keystone 和 LDAP。从 H 版本起，domain 的概念被引入，它为用户提供了 user namespace 功能，即不同的 domain 的 user(group) 完全隔离，允许不同的 domain 下的用户重名。一个通常的做法是，云服务提供商给每家客户分配各一个 domain，这样不同客户之间的用户信息就完全隔离。事实上，绝大多数企业客户都有自身的用户管理系统，这些用户管理系统通常是 LDAP 或者 Microsolt 的 AD。出于安全等因素的考虑，企业客户使用云服务时，更希望云服务提供商能够接入已有的认证系统，Multi-Domain Config 为这种场景提供了优良的解决方案。Multi-Domain Config 是 J 版本的一个非常振奋人心的特性，它支持每个 domain 对接客户已有的 LDAP、AD 或者 SQL。

![multi-domain](http://7xp2eu.com1.z0.glb.clouddn.com/multi-domain-overview.png)

如上图，以认证为例，当 Keystone 收到 authentication 的请求时，对于 domain A 下的用户，它最终发请求到 LDAP A 做认证，同理，对于 domain B 下的用户，Keystone 最终依赖 LDAP B 认证用户。要实现此类功能，Keystone 需要维护不同 domain 的 LDAP 的信息，方式有两种：

- 配置文件
- API

顾名思义，基于配置文件的方式就是把 LDAP 信息存放在配置文件中，一个文件维护一个 domain 的 LDAP 信息，配置文件的命名规则为 keystone.{domain-name}.conf，这些配置文件通常存放在 /etc/keystone/domain 目录中。每当有配置文件添加、删除和更改时，均需要重启 Keystone 才能生效。例如：

~~~
# keystone.hpe.conf

[ldap]url = ldap://hpe.ldap.test:10389query_scope = subuser_tree_dn = ou=users,ou=test,ou=customers,dc=ecs,dc=hpe,dc=comuser_objectclass = MyOrgPersonuser_id_attribute = cnuser_name_attribute = mailuser_mail_attribute = mailuser_pass_attribute = userPassworduser_enabled_attribute = enabledgroup_tree_dn = ou=groups,ou=test,ou=customers,dc=ecs,dc=hpe,dc=comgroup_objectclass = groupOfUniqueNamesgroup_id_attribute = cngroup_name_attribute = cngroup_member_attribute = uniquemember

user_allow_create = true
group_allow_create = true
~~~

使用配置文件有两个缺点：增加运维复杂度，重启会造成服务中断。K 版本引入了用于管理 domain 的 LDAP 信息的 [Domain Config API](http://developer.openstack.org/api-ref-identity-v3.html#domains-config-v3)，用户可以方便的使用这些 API 来维护 LDAP 的信息，这是本文的重点：

~~~
PATH: /v3/domains/​{domain_id}​/config
Method: PATCH/GET/DELETE
~~~

例如，为 domain 创建 LDAP 信息：

~~~
$ curl -s -X PUT -H "X-Auth-Token: $OS_TOKEN" -H "Content-Type: application/json" - d '
{    "config": {        "identity": {            "driver": "keystone.identity.backends.ldap.Identity"        },        "ldap": {
            "url": "ldap://hpe.ldap.test:10389",            "query_scope": "sub",            "user_tree_dn": "ou=users,ou=test,ou=customers,dc=ecs,dc=hpe,dc=com",            "user_objectclass": "MyOrgPerson",            "user_id_attribute": "cn",            "user_name_attribute": "mail",            "user_mail_attribute": "mail",            "user_pass_attribute": "userPassword",            "user_enabled_attribute": "enabled",            "group_tree_dn": "ou=groups,ou=test,ou=customers,dc=ecs,dc=hpe,dc=com",            "group_objectclass": "groupOfUniqueNames",            "group_id_attribute": "cn",            "group_name_attribute": "cn",            "group_member_attribute": "uniquemember",
            "user_allow_create": "true",
            "group_allow_create": "true"        }    }}' http://localhost:5000/v3/domains/a52d06a163f049e29416e20d0e8a12ef/config
~~~

-----------

# Practice

本节展示 Domain Config API 的使用，使用前需要设置以下配置项：

~~~
[identity]

domain_specific_drivers_enabled = true
domain_configurations_from_database = true
~~~

1.Create A new Domain

~~~
$ openstack domain create hpe+---------+----------------------------------+| Field   | Value                            |+---------+----------------------------------+| enabled | True                             || id      | a52d06a163f049e29416e20d0e8a12ef || name    | ibm                              |+---------+----------------------------------+
~~~

2.Create domain configuration options for hpe

~~~
$ curl -s -X PUT -H "X-Auth-Token: $OS_TOKEN" -H "Content-Type: application/json" - d '
{    "config": {        "identity": {            "driver": "keystone.identity.backends.ldap.Identity"        },        "ldap": {
            "url": "ldap://hpe.ldap.test:10389",            "query_scope": "sub",            "user_tree_dn": "ou=users,ou=test,ou=customers,dc=ecs,dc=hpe,dc=com",            "user_objectclass": "MyOrgPerson",            "user_id_attribute": "cn",            "user_name_attribute": "mail",            "user_mail_attribute": "mail",            "user_pass_attribute": "userPassword",            "user_enabled_attribute": "enabled",            "group_tree_dn": "ou=groups,ou=test,ou=customers,dc=ecs,dc=hpe,dc=com",            "group_objectclass": "groupOfUniqueNames",            "group_id_attribute": "cn",            "group_name_attribute": "cn",            "group_member_attribute": "uniquemember",
            "user_allow_create": "true",
            "group_allow_create": "true"        }    }}' http://localhost:5000/v3/domains/a52d06a163f049e29416e20d0e8a12ef/config
~~~

3.Show a user in LDAP

~~~
$ openstack user show testusera@hpe.com --domain hpe+-----------+------------------------------------------------------------------+| Field     | Value                                                            |+-----------+------------------------------------------------------------------+| domain_id | a52d06a163f049e29416e20d0e8a12ef                                 || email     | testusera@hpe.com                                                || id        | 169906f82ac911e69d6bb8e8562f95ea349e8a1c2ac911e68142b8e8562f95ea || name      | testusera@hpe.com                                                |+-----------+------------------------------------------------------------------+
~~~

---------

# Some tips

- Domain 和 Domain Config 都只被 V3 的 API 支持，但是为了兼容 V2 的 API，社区构建了一个特殊的 domain，它的名称和 id 都是 default，即 V2 的 API 只能访问 default domain 下的 user，project 等等。Multi-Domain Config identity 支持 LDAP(AD) 和 SQL 两种后端，但它有一个很大的限制：仅仅允许存在一个 SQL！为了达到最佳的兼容性，我们通常把 keystone.conf 吧 identity 的 driver 配置为 sql，default 和某些不需要 LDAP 的 domain 就可以使用 SQL 作为存储的后端，对于某些需要配置 LDAP 的 domain，则调用 domain config API 来维护相关信息。
- 配置文件和 API 的方式二者只能选其一。
- 由于存在并发同步的问题，Domain Config API 在 K、L 版本均被注明为 [experimental](https://bugs.launchpad.net/keystone/+bug/1429557)，本人认为这些问题的影响不大。


