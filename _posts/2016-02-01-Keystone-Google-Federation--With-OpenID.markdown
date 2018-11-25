---
layout: post
title:  "Keystone Google Federation With OpenID"
categories: OpenStack
---


本文是 Keystone federation identity 系列的第二篇，主要介基于 OpenID Connect 协议搭建 Keystone 和 Google 单点登录的 demo。Federation Identity 的概念、原理以及基于 SAML2 协议的文章请见[第一篇](http://wsfdl.com/openstack/2016/01/14/Keystone-Federation-Identity-with-SAML2.html)

环境：

- Linux: Ubuntu 14.04 LTS
- OpenStack: Kilo
- Identity Provider: Google
- Federation Protocol： OpenID Connect

-------

# OpenID Connect Protocol

[OpenID](http://openid.net/connect/) 的官网是这样定义 OpenID Connect：

> OpenID Connect 1.0 is a simple identity layer on top of the OAuth 2.0 protocol. It allows Clients to verify the identity of the End-User based on the authentication performed by an Authorization Server, as well as to obtain basic profile information about the End-User in an interoperable and REST-like manner.

以上话语可简单的概括成如下几点：

- 用于认证 (Authtication) 用户，认证成功后返回用户的基本信息
- 基于 OAuth 2.0 协议，其中 OAuth 是授权(Authorization)相关协议
- 提供了友好的 API，支持 Rest-like 风格
- 支持 web 和 app 等

即：

OpenID Connect = OAuth + Identity + Authentication 

-------

# Setup Google

首先申请一个域名，并将域名指向 Keystone server，本文在阿里云申请如下域名：

> keystonegoogle.com

Keystone server 的 IP 为 209.9.108.221，设置如下 DNS 解析：

![dns](http://wsfdl.oss-cn-qingdao.aliyuncs.com/keystonegoogledns.png?imageView2/1/w/360/h/80/q/100)

参考[此文](https://www.youtube.com/watch?v=Rfxy0FKOfgw)在 [console.developers.google.com](https://console.developers.google.com/) 注册一个 application，获取 application 的 Client ID 和 Client Secret 信息，并在 Authorized redirect URIs 中填入重定向 URI，以本文为例：

- Client ID：388517667150-adc2etk5ohfif5bluber4ho2150pqb3k.apps.googleusercontent.com
- Client secret：2CnpJ5mm8mfqfoN_6aqd-72A
- Authorized redirect URIs：www.keystonegoogle.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect

![google-openid-client](http://wsfdl.oss-cn-qingdao.aliyuncs.com/googleclientsetup.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;

-------

# Setup Keystone

在某公有云厂商创建一台境外服务器，以便能访问 Google。参考官网文档 [OpenStack Kilo Installation Guide](http://docs.openstack.org/kilo/install-guide/install/apt/content/) 安装至第六章 network 即可，因 ubuntu 对 horizon 做了较大改动，所以参考[文档](http://docs.openstack.org/developer/horizon/topics/install.html)采用源码安装 horizon。

## Configure Keystone

下面的安装和配置主要参考 [Identity, Authentication & Access Management in OpenStack](http://shop.oreilly.com/product/0636920045960.do)，首先安装 libapache2-mode-auth-openidc：

~~~ bash
$ apt-get install libjansson4 libhiredis0.10 libcurl3
$ wget https://github.com/pingidentity/mod_auth_openidc/releases/download/v1.8.6/libapache2-mod-auth-openidc_1.8.6-1_amd64.deb
$ dpkg -i libapache2-mod-auth-openidc_1.8.6-1_amd64.deb
$ cp /opt/stack/keystone/etc/sso_callback_template.html /etc/keystone
~~~

更新 /etc/keystone/keystone.conf 的如下配置：

~~~
[auth]
methods = external,password,token,oauth1,oidc
oidc = keystone.auth.plugins.mapped.Mapped

[oidc]
remote_id_attribute = HTTP_OIDC_ISS

[federation]
remote_id_attribute = HTTP_OIDC_ISS
trusted_dashboard = http://keystonegoogle.com/auth/websso/
~~~

更新 /etc/apache2/sites-available/wsgi-keystone.conf 配置如下，注意到 OIDCClientID，OIDCClientSecret 和 OIDCRedirectURI 依次对应上文 Google Client 的参数：

~~~ bash
$ cat /etc/apache2/sites-available/wsgi-keystone.conf
Listen 5000
Listen 35357

LoadModule auth_openidc_module /usr/lib/apache2/modules/mod_auth_openidc.so

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /var/www/cgi-bin/keystone/main
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    LogLevel debug
    ErrorLog /var/log/apache2/keystone-error.log
    CustomLog /var/log/apache2/keystone-access.log combined

    OIDCClaimPrefix "OIDC-"
    OIDCResponseType "id_token"
    OIDCScope "openid email profile"
    OIDCProviderMetadataURL "https://accounts.google.com/.well-known/openid-configuration"
    OIDCClientID "388517667150-adc2etk5ohfif5bluber4ho2150pqb3k.apps.googleusercontent.com"
    OIDCClientSecret 2CnpJ5mm8mfqfoN_6aqd-72A
    OIDCCryptoPassphrase openstack
    OIDCRedirectURI "http://keystonegoogle.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect"
    <Location ~ "/v3/auth/OS-FEDERATION/websso/oidc">
      AuthType openid-connect
      Require valid-user
      LogLevel debug
    </Location>
</VirtualHost>

<VirtualHost *:35357>
......
~~~

## Configure Horizon

Horizon 默认不启用 websso，所以需要更新 local_settings.py 以下配置：

~~~ python
OPENSTACK_KEYSTONE_URL = "http://keystonegoogle.com:5000/v3”
OPENSTACK_API_VERSIONS = {
         "identity": 3}
WEBSSO_ENABLED = True
WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("oidc", _("Google Login")))
WEBSSO_INITIAL_CHOICE = "credentials"
~~~ 

重启 Keystone 和 Horizon：

~~~ bash
$ service apache2 restart
~~~

## Create User etc

创建 Google 的 Group 和 Project：

~~~ bash
$ openstack group create google_group
$ openstack project create google_project
$ openstack role add admin --group google_group --project google_project
~~~

创建 Google Identity Provider：

~~~ bash
$ curl -g -X POST http://keystonegoogle.com:35357/v3/OS-FEDERATION/identity_providers/google_idp
 -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token: $token" -d '{"identity_provider": {"enabled": true, "description": null, "remote_ids": ["https://accounts.google.com"]}}'
~~~

创建 mapping 等相关信息：

~~~ shell
$ cat google_mapping.json
[
  {
    "local": [
      {
        "group": {
          "id": "a52d06a163f049e29416e20d0e8a12ea"
          }
        }
      ],
    "remote": [
        {
          "type": "HTTP_OIDC_ISS",
          "any_one_of": [
            "https://accounts.google.com"
            ]
          }
        ]
  }
]

$ openstack mapping create google_mapping --rules google_mapping.json
$ openstack federation protocol create oidc --identity-provider google_idp --mapping google_mapping
~~~

----------

# Test

登录 keystonegoogle.com，进入 Login 页面。

![login](http://wsfdl.oss-cn-qingdao.aliyuncs.com/Login.png)

选择 Google Login，输入 Google 账户密码即可登录。

![dashboard](http://wsfdl.oss-cn-qingdao.aliyuncs.com/keystonegooglegoogleinstance.png)
