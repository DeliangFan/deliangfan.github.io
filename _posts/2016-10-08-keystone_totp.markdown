---
layout: post
title:  "多因子认证之 Keystone TOTP"
categories: OpenStack
---

随着 blueprint [totp-auth](https://blueprints.launchpad.net/keystone/+spec/totp-auth) 的实施，Keystone 自 M 版本新增了一次性密码 [Time-base One-time Password(TOTP)](http://wsfdl.com/algorithm/2016/04/14/%E7%90%86%E8%A7%A3TOTP.html) 认证方式。其功能基础，操作稍有不便，距离生产环境尚有一段距离：

- 二维码的认证由 Keystone 完成，市场上已有许多成熟的 MFA 产品或解决方案，Keystone 应当像对接 LDAP 一样对接这些成熟的多因子认证产品，然后由这些产品负责校验一次性密码
- 只支持 user 级别的 enble/disable，有时需要 group 和 domain 级别的 enable/disable
- 用户端的配置复杂，用户需生成二维码并扫描之
- 用户仅选择 totp 的认证方式也可以获取 token，安全性低。当开启 TOTP 认证方式后，Keystone 应当强制要求双因子认证。

虽有诸多缺点，但不失为多因子认证的里程碑，本文主要介绍如何使用 Keystone TOTP，详情请见[官网文档](http://docs.openstack.org/developer/keystone/auth-totp.html)。

-------------

# Configuring Keystone

TOTP 默认是禁止的，使用前需要更新以下配置项从而开启它：

~~~
# keystone.conf

[auth]
methods = external,password,token,oauth1,totp
~~~

采用以下脚本生成 TOTP 的 secret key：

~~~
import base64
message = '1234567890123456'
print base64.b32encode(message).rstrip('=')
~~~

生成的 key 如下：

~~~
GEZDGNBVGY3TQOJQGEZDGNBVGY
~~~

为需要 TOTP 认证的用户配置相关信息，Keystone 在认证时利用这些信息生成一次性密码，从而验证用户提供的一次性密码的有效性：

~~~
SECRET=GEZDGNBVGY3TQOJQGEZDGNBVGY

curl -i -H "Content-Type: application/json" \
-H "X-Auth-Token: 9490cd78c39943379714198b7153fe1b"\
-d '
{
    "credential": {
        "blob": "'$SECRET'",
        "type": "totp",
        "user_id": "56117f0c793e45c5bd91888665fd8412"
    }
}' \
http://localhost:5000/v3/credentials
~~~

-------------

# Configuring User Side

当用户开启 TOTP 认证后，用户可以通过硬件或者软件的方式(如：Google Authenticator)生成一次性密码，本文使用 Google Authenticator，该软件可在 Apple APP Store 下载，安装后扫下面生成的二维码即可：

~~~
import qrcode

secret='GEZDGNBVGY3TQOJQGEZDGNBVGY'
uri = 'otpauth://totp/{name}?secret={secret}&issuer={issuer}'.format(
    name='totp_user',
    secret=secret,
    issuer='Keystone_server')

img = qrcode.make(uri)
img.save('totp.png')
~~~

![totp image](http://7xp2eu.com1.z0.glb.clouddn.com/totp.png)

扫描后如下：

![totp passcode](http://7xp2eu.com1.z0.glb.clouddn.com/totp_passcode.png)

验证如下：

~~~
PASSCODE=019863

curl -i \
  -H "Content-Type: application/json" \
  -d '
{ "auth": {
        "identity": {
            "methods": [
                "totp","password"
            ],
            "totp": {
                "user": {
                    "id": "56117f0c793e45c5bd91888665fd8412",
                    "passcode": "'$PASSCODE'"
                }
            },
            "password": {
                "user": {
                    "id": "56117f0c793e45c5bd91888665fd8412",
                    "password": "123456"
                }
            }           
        },
        "scope": {
            "project": {
                "id": "4d7651138dde46bfb31d1c1c1edacc9b"
            }
        }
    }
}' \
  http://localhost:5000/v3/auth/tokens
~~~