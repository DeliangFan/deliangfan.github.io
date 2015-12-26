
---
layout: post
title: "理解Keystone的四种Token"
categories: OpenStack
---

---------

#Token 是什么

通俗的讲，token 是用户的一种凭证，需拿正确的用户名/密码向 keystone 申请才能得到。如果用户每次都采用用户名/密码访问 OpenStack API，容易泄露用户信息，带来安全隐患。所以 OpenStack 要求用户访问其 API 前，必须先获取 token，然后用 token 作为用户凭据访问 OpenStack API。
![P1](http://7xp2eu.com1.z0.glb.clouddn.com/uuid.png)




#四种 Token 的由来
D 版本时，仅有 UUID 类型的 Token，UUID token 简单易用，却容易给 Keystone 带来性能问题，从图一的步骤 4 可看出，每当 OpenStack API 收到用户请求，都需要向 Keystone 验证该 token 是否有效。随着集群规模的扩大，Keystone 需处理大量验证 token 的请求，在高并发下容易出现性能问题。

于是，PKI([Public Key Infrastructrue](https://wiki.openstack.org/wiki/PKI)) token 在 G 版本运用而生，和 UUID 相比，PKI token 携带更多用户信息的同时还附上了数字签名，以支持本地认证，从而避免了步骤 4。因为 PKI token 携带了更多的信息，这些信息就包括 service catalog，随着 OpenStack 的 Region 数增多，service catalog 携带的 endpoint 数量越多，PKI token 也相应增大，很容易超出 HTTP Server 允许的最大 HTTP Header(默认为 8 KB)，导致 HTTP 请求失败。

顾名思义，[PKIZ token](https://blueprints.launchpad.net/keystone/+spec/compress-tokens) 就是 PKI token 的压缩版，但压缩效果有限，无法良好的处理 token size 过大问题。 

前三种 token 都会持久性存于数据库，大量与日俱增的 token 引起数据库性能下降，所以用户需经常清理数据库的 token。为了避免该问题，社区提出了 Fernet token，它携带了少量的用户信息，大小约为 255 Byte，采用了对称加密，无需存于数据库中。 

#UUID
UUID token 是长度固定为 32 Byte 的随机字符串，由 uuid.uuid4().hex 生成。

    def _get_token_id(self, token_data):
        return uuid.uuid4().hex

但是因 UUID token 不携带其它信息，OpenStack API 收到该 token 后，既不能判断该 token 是否有效，更无法得知该 token 携带的用户信息，所以需经图一步骤 4 向 Keystone 校验 token，并获用户相关的信息。其样例如下：

	144d8a99a42447379ac37f78bf0ef608

UUID token 简单美观，不携带其它信息，因此 Keystone 必须实现 token 的存储和认证，随着集群的规模增大，Keystone 将很有可能成为性能瓶颈。
 
#PKI
![P2](http://7xp2eu.com1.z0.glb.clouddn.com/pki.png)
在阐述 PKI（Public Key Infrastruction） token 前，让我们简单的回顾[公开密钥加密(public-key cryptography)](https://zh.wikipedia.org/wiki/%E5%85%AC%E5%BC%80%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86)和[数字签名](http://www.youdzone.com/signature.html)。公开密钥加密，也称为非对称加密(asymmetric cryptography，加密密钥和解密密钥不相同)，在这种密码学方法中，需要一对密钥，分别为公钥(Public Key)和私钥(Private Key)，公钥是公开的，私钥是非公开的，需用户妥善保管。如果把加密和解密的流程当做函数 C(x) 和 D(x)，P 和 S 分别代表公钥和私钥，对明文 A 和密文 B 而言，数学的角度上有以下公式：
B = C(A, S)
A = D(B, P)
其中加密函数 C(x), 解密函数 D(x) 以及公钥 P 均是公开的。上面两个公式的意思是公钥加密的密文只能用私钥解密，采用私钥加密的密文只能用公钥解密。非对称加密广泛运用在安全领域，诸如常见的 HTTPS，SSH 登录等。

数字签名又称为公钥数字签名，首先采用 Hash 函数对消息生成摘要，摘要经私钥加密后称为数字签名。接收方用公钥解密该数字签名，并与接收消息生成的摘要做对比，如果二者一致，便可以确认该消息的完整性和真实性。

PKI 的本质就是基于数字签名，keystone 用私钥对 token 进行数字签名，各个 API server 用公钥在本地验证该 token。相关代码简化如下：

    def _get_token_id(self, token_data):
        try:
            token_json = jsonutils.dumps(token_data, cls=utils.PKIEncoder)
            token_id = str(cms.cms_sign_token(token_json,
                                              CONF.signing.certfile,
                                              CONF.signing.keyfile))
            return token_id

其中 cms.cms_sign_token 调用 openssl cms --sign 对 token_data 进行签名，token_data 的样式如下：

	{
      "token": {
        "methods": [
          "password"
        ],
        "roles": [
          {
            "id": "5642056d336b4c2a894882425ce22a86",
            "name": "admin"
          }
        ],
        "expires_at": "2015-12-25T09:57:28.404275Z",
        "project": {
          "domain": {
            "id": "default",
            "name": "Default"
          },
          "id": "144d8a99a42447379ac37f78bf0ef608",
          "name": "admin"
        },
        "catalog": [
          {
            "endpoints": [
              {
                "region_id": "RegionOne",
                "url": "http://controller:5000/v2.0",
                "region": "RegionOne",
                "interface": "public",
                "id": "3837de623efd4af799e050d4d8d1f307"
              },
              {
                "region_id": "RegionOne",
                "url": "http://controller:35357/v2.0",
                "region": "RegionOne",
                "interface": "admin",
                "id": "3ffc46344293437ca2919107980daee4"
              },
              {
                "region_id": "RegionOne",
                "url": "http://controller:5000/v2.0",
                "region": "RegionOne",
                "interface": "internal",
                "id": "c138238caeb04bb391ea0957ac020ccc"
              }
            ],
            "type": "identity",
            "id": "888accf6f1364001af0b829f51d905c3",
            "name": "keystone"
          }
        ],
        "extras": {
        },
        "user": {
          "domain": {
            "id": "default",
            "name": "Default"
          },
          "id": "1552d60a042e4a2caa07ea7ae6aa2f09",
          "name": "admin"
        },
        "audit_ids": [
          "ZCvZW2TtTgiaAsVA8qmc3A"
        ],
        "issued_at": "2015-12-25T08:57:28.404304Z"
      }
    }

	
token_data 经 cms.cms_sign_token 签名生成的 token_id 如下，共 1932 Byte：

	MIIKoZIhvcNAQcCoIIFljCCBZICAQExDTALBglghkgBZQMEAgEwggPzBgkqhkiG9w0BBwGgggPkBIID4HsidG9rZW4iOnsibWV0aG9kcyI6Wy......+aRbfbxLTTz14fLX53QrHgcfuAl27dw+Vo4GzZ-D2Hrhr0acV3bMKzmqvViHf-fPVnLDMJajOWSuhimqfLZHRdr+ck0WVQosB6+M6iAvrEF7v



#PKIZ
![P3](http://7xp2eu.com1.z0.glb.clouddn.com/pkiz.png)
PKIZ 在 PKI 的基础上做了压缩处理，但是压缩的效果极其有限，一般情况下，压缩后的大小为 PKI token 的 90 % 左右，所以 PKIZ 不能友好的解决 token size 太大问题。

    def _get_token_id(self, token_data):
        try:
            token_json = jsonutils.dumps(token_data, cls=utils.PKIEncoder)
            token_id = str(cms.pkiz_sign(token_json,
                                         CONF.signing.certfile,
                                         CONF.signing.keyfile))
            return token_id

其中 cms.pkiz_sign() 比 cms.pki_sign() 多了如下一行代码，由 zlib 对签名后的消息进行压缩级别为 6 的压缩。

	compressed = zlib.compress(token_id, compression_level=6)

PKIZ token 样例如下，共 1645 Byte，比 PKI token 减小 14.86 %：

	PKIZ_eJytVcuOozgU3fMVs49aTXhUN0vAQEHFJiRg8IVHgn5OnA149JVaunNS3NYjoSUri2r885957Lly_iZzmej_6y4XZSJB33eHiAZvRZX9xyZU......W4fRaxrbNtinB2ze3PXjv3sMqqvzunYxFa_fO1tyb0jteB0czPKKt97rojheVICXYrEk0oPX6TSnP71IYj2e3nm4MLy7S84PtIPDz4_03IsOb2Q=


#Fernet
![P4](http://7xp2eu.com1.z0.glb.clouddn.com/fernet.png)
用户可能会碰上这么一个问题，当集群运行较长一段时间后，访问其 API 会变得奇慢无比，究其原因在于 Keystone 数据库存储了大量的 token 导致性能太差，解决的办法是经常清理 token。为了避免上述问题，社区提出了[Fernet token](https://github.com/openstack/keystone-specs/blob/master/specs/kilo/klwt.rst)，它采用 [cryptography](http://cryptography.readthedocs.org/en/latest/fernet/) 对称加密库(symmetric cryptography，加密密钥和解密密钥相同) 加密 token，具体由 AES-CBC 加密和散列函数 SHA256 签名。[Fernet](http://cryptography.readthedocs.org/en/latest/fernet/)
是专为 API token 设计的一种轻量级安全消息格式，不需要存储于数据库，减少了磁盘的 IO，带来了一定的[性能提升](http://dolphm.com/benchmarking-openstack-keystone-token-formats/)。为了提高安全性，需要采用 [Key Rotation](http://lbragstad.com/fernet-tokens-and-key-rotation/) 更换密钥。

    def create_token(self, user_id, expires_at, audit_ids, methods=None,
                     domain_id=None, project_id=None, trust_id=None,
                     federated_info=None):
        """Given a set of payload attributes, generate a Fernet token."""
        
        if trust_id:
            version = TrustScopedPayload.version
            payload = TrustScopedPayload.assemble(
                user_id,
                methods,
                project_id,
                expires_at,
                audit_ids,
                trust_id)
        
        ...
        
        versioned_payload = (version,) + payload
        serialized_payload = msgpack.packb(versioned_payload)
        token = self.pack(serialized_payload)

        return token

以上代码表明，token 包含了 user_id，project_id，domain_id，methods，expires_at 等信息，重要的是，它没有 service_catalog，所以 region 的数量并不影响它的大小。self.pack() 最终调用如下代码对上述信息加密：

    def crypto(self):
        keys = utils.load_keys()

        if not keys:
            raise exception.KeysNotFound()

        fernet_instances = [fernet.Fernet(key) for key in utils.load_keys()]
        return fernet.MultiFernet(fernet_instances)

该 token 的大小一般在 200 多 Byte 左右，本例样式如下，大小为 186 Byte：
 	
	gAAAAABWfX8riU57aj0tkWdoIL6UdbViV-632pv0rw4zk9igCZXgC-sKwhVuVb-wyMVC9e5TFc7uPfKwNlT6cnzLalb3Hj0K3bc1X9ZXhde9C2ghsSfVuudMhfR8rThNBnh55RzOB8YTyBnl9MoQXBO5UIFvC7wLTh_2klihb6hKuUqB6Sj3i_8


#如何选择 Token


 Token 类型  |UUID   |PKI|PKIZ|Fernet
------------|-------|---|----|------
大小         |32 Byte|KB 级别| KB 级别| 约 255 Byte
支持本地认证  |不支持|支持|支持|不支持
Keystone 负载|大 |小|小|大
存储于数据库  |是|是|是|否
携带信息     |无|User, Project, Role, Domain, Catalog 等|User, Project, Role, Domain, Catalog 等|User, Project 等
涉及加密方式     |无|非对称加密|非对称加密|对称加密(AES)
是否压缩     |否|否|是|否
版本支持| D|G|J|K


Token 类型的选择涉及多个因素，包括 Keystone server 的负载、region 数量、安全因素、维护成本以及 token 本身的成熟度。region 的数量影响 PKI/PKIZ token 的大小，从安全的角度上看，UUID 无需维护密钥，PKI 需要妥善保管 Keystone server 上的私钥，Fernet 需要周期性的更换密钥，因此从安全、维护成本和成熟度上看，UUID > PKI/PKIZ > Fernet 如果：

- Keystone server 负载低，region 少于 3 个，采用 UUID token。
- Keystone server 负载高，region 少于 3 个，采用 PKI/PKIZ token。
- Keystone server 负载低，region 大与或等于 3 个，采用 UUID token。
- Keystone server 负载高，region 大于或等于 3 个，K 版本及以上可考虑采用 Fernet token。
