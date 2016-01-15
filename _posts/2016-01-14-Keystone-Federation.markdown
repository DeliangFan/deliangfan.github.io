---
layout: post
title:  "Keystone Federation"
categories: OpenStack
---



Linux: Ubuntu 14.04 LTS
OpenStack: Kilo

------------------
#1. Federation 简介


思索了许久，却难以找到合适准确的中文词句定义 [federation identity](https://en.wikipedia.org/wiki/Federated_identity)，所以直接引用维基百科的定义：

> A federated identity is the means of linking a person's electronic identity and attributes, stored across multiple distinct identity management systems(IDM).

通俗而言，在多个认证管理系统(IDM)相互信任的基础上，federation identity 允许多个认证管理系统联邦认证各个系统的用户身份。它最重要的一个功能就是实现单点登录(SSO)，即用户仅需认证一次，便可访问这些相互授信的资源。假设 A, B 两个企业有独自的员工管理系统，A 公司员工需要访问 B 公司的某个服务，出于安全等因素，但不希望 B 存储 A 公司的员工信息，通过 federation identity 可为上述场景提供很好的解决方案，简单的说，它首先需要 A，B 公司互相信任，每当有 A 公司员工要访问 B 的服务时，B 依赖 A 的认证系统通过某种认证协议完成用户身份信息的确认。我们把 A 称之为 Identity Provider, B 称之为 Service Provider，员工称之为 Subject。

- Subject: 可简单的理解成用户
- Service Provider: 服务提供方，服务提供方只提供服务，依赖 Identity Provider 认证用户身份信息
- Identity Provider: 断言(assertion)方，用于审核和认证用户身份
- Assertion Protocol: 认证(断言)协议，Service Provider 和 Identity Provider 完成认证用户身份所用的协议，常用有 SAML, OpenID, Oauth 等

以 SAML 协议为例，典型的的 federation identity 的[认证流程](http://www.searchsoa.com.cn/showcontent_1604.htm)可分为 [Redirect Bindings](https://en.wikipedia.org/wiki/SAML_2.0#HTTP_Redirect_Binding) 和 [Artifact/POST Bindings](https://en.wikipedia.org/wiki/SAML_2.0#HTTP_Artifact_Binding) 两种。

![Redirect Bindings](http://7xp2eu.com1.z0.glb.clouddn.com/Redirect%20Binding.png)

![Artifact Bindings](http://7xp2eu.com1.z0.glb.clouddn.com/artifact%20binding.png)

Federation identity 具有以下优点：

- 支持 SSO 单点登录
- 避免向 Service Provider 暴露用户身份信息，提高安全性
- 避免用户注册多个账号，增加用户负担

事实上，在 cloud 领域也有诸多 federation identity 场景需求，比如 A 公司的员工需使用 AWS 的公有云服务，但又不想在 AWS 中为每一个员工创建账户信息，那么通过 federation identity 就可以打通二者之间的用户认证和授权，使得 A 公司员工只需在本公司完成身份认证即可访问 AWS 资源。可以说，federation 为 hybrid cloud 在用户管理方面提供了良好的解决方案。

--------------
# Keystone Federation 的原理

鉴于 federation identity 的诸多优点，Keystone 从 Icehouse 开始逐步增加 federation identity 的功能，Icehouse 支持 Keystone 作为 Service Provider，Juno 版本又支持 Keystone 作为 Identity Provider，支持 SAML 和 OpenID 两种认证协议。

通常情况下，OpenStack 作为云服务的解决方案，对外提供计算、存储和网络等服务，所以在 federation identity 的场景下，Keystone 更多的作为 Service Provider，对接其它的 Identity Provider，所以本节着重阐述 Service Provider 的原理。


--------------

#Keystone as a Service Provider

Keystone 的配置如下：

```
keystone.conf

[auth]
methods = external,password,token,oauth1,saml2
saml2 = keystone.auth.plugins.mapped.Mapped 
```

Apache 的配置如下：

```
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /var/www/cgi-bin/keystone/main/$1
     ......
</VirtualHost>

<VirtualHost *:35357>
    WSGIScriptAliasMatch ^(/v3/OS-FEDERATION/identity_providers/.*?/protocols/.*?/auth)$ /var/www/cgi-bin/keystone/admin/$1
      ......
</VirtualHost>

<Location /Shibboleth.sso>
    SetHandler shib
</Location>

<LocationMatch /v3/OS-FEDERATION/identity_providers/.*?/protocols/saml2/auth>
    ShibRequestSetting requireSession 1
    AuthType shibboleth
    ShibExportAssertion Off
    Require valid-user
</LocationMatch>
``` 

Shibboleth

Keystone consumes SAML assertions issued by external Identity Providers (which will be another Keystone in the case of K2K federation). This is done via a third party SP software, in this guide we will be using Shibboleth

```
apt-get install libapache2-mod-shib2
```

```
/etc/shibboleth/attribute-map.xml


<Attribute name="openstack_user" id="openstack_user"/>  
<Attribute name="openstack_roles" id="openstack_roles"/>  
<Attribute name="openstack_project" id="openstack_project"/> 
<Attribute name="openstack_user_domain" id="openstack_user_domain"/>  
<Attribute name="openstack_project_domain" id="openstack_project_domain"/>  
```

/etc/shibboleth/shibboleth2.xml

```
<SSO entityID="http://idp:5000/v3/OS-FEDERATION/saml2/idp">  
    SAML2 SAML1
</SSO>

<MetadataProvider type="XML" uri="http://idp:5000/v3/OS-FEDERATION/saml2/metadata"/>  
```

```
shib-keygen  
service apache2 restart

root@SP:~# a2enmod shib2
Module shib2 already enabled
```

----------

# Keystone as an Identity Provider

```
sudo apt-get install xmlsec1  
sudo pip install pysaml2
```

keystone.conf

```
[saml]
certfile=/etc/keystone/ssl/certs/ca.pem  
keyfile=/etc/keystone/ssl/private/cakey.pem  
idp_entity_id=http://keystone.idp/v3/OS-FEDERATION/saml2/idp  
idp_sso_endpoint=http://keystone.idp/v3/OS-FEDERATION/saml2/sso  
idp_metadata_path=/etc/keystone/keystone_idp_metadata.xml  
```

```
keystone-manage saml_idp_metadata > /etc/keystone/keystone_idp_metadata.xml
service apache2 restart
```

# Reference
1. https://developer.rackspace.com/blog/keystone-to-keystone-federation-with-openstack-ansible/
2. https://specs.openstack.org/openstack/keystone-specs/api/v3/identity-api-v3-os-federation-ext.html
3. http://blog.rodrigods.com/it-is-time-to-play-with-keystone-to-keystone-federation-in-kilo/
4. https://bigjools.wordpress.com/2015/05/22/saml-federation-with-openstack/