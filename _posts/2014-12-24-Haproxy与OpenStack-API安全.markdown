---
layout: post
title:  "Haproxy与OpenStack-API安全"
categories: DevOps
---

----------

[Haproxy](http://docs.openstack.org/ha-guide/controller-ha-haproxy.html) 作为优秀的软件负载均衡器，常常作为服务的前端，为外界提供服务的入口，同时隐藏服务端内部的细节，通过负载均衡提高服务端的并发量。随着 Haproxy 的发展，它的安全性也不断的提升。

- 作为负载均衡器，提高服务端的并发处理能力
- 保证服务高可靠性
- 增强安全性

OpenStack 社区在 [High Availability](http://docs.openstack.org/ha-guide/intro-ha-arch-keepalived.html) (高可靠) 推荐 Controller 端部署 Haproxy 和 Keepalived，前者负责转发外部 http 请求到正常的 OpenStack API 节点，后者通过 VRRP 协议管理 VIP 的漂移，如下图：

![Controller HA](http://7xp2eu.com1.z0.glb.clouddn.com/Controller%20HA.png?imageView2/1/w/800/q/100)

本文重点关注其安全特性，首先，它作为服务的唯一入口，对外隐藏了 OpenStack API 服务器的细节；其次，支持 HTTPS，保证客户端和服务端通信的安全性；再次，它配合内核 IP/TCP 协议栈，能够较好的抵抗 DDOS 攻击，还能限制单个 IP 的连接数和请求速率等，防止用户的恶意行为。

- 作为服务端唯一入口，隐藏服务端的细节
- 支持 SSL/TSL, 提供 HTTPS
- 通过配置 ACL，在一定程度上防止 DDOS 攻击 

Haproxy 配置参数多的一塌糊涂，功能丰富，灵活多样，本文抛砖引玉，在文章末尾给出 nova-api 的配置样例，更多的安全和其它功能请详见[官网手册](https://cbonte.github.io/haproxy-dconv/index.html)。

----------

#HTTPS

1.5.0 版本的 Release Notes 有下面这么一行话，意味着从该版本开始， haproxy 支持由 HTTPS 加密客户端和服务端的通信消息，避免明文传输。

	Update [2012/09/11] : native SSL support was implemented in 1.5-dev12.

![https-http](http://7xp2eu.com1.z0.glb.clouddn.com/https-http.png?imageView2/1/w/800/q/100)

[how-to-implement-ssl-termination-with-haproxy](https://www.digitalocean.com/community/tutorials/how-to-implement-ssl-termination-with-haproxy-on-ubuntu-14-04) 详细的说明如何配置 HTTPS，主要包括证书 和 SSL 相关配置项。证书可以到 CA 机构购买，也可用 openssl 自制证书，甚至可用 [keystone 生成证书](http://docs.openstack.org/developer/keystone/configuration.html#ssl)。

```
[eventlet_server_ssl]
enable = True
certfile = <path to keystone.pem>
keyfile = <path to keystonekey.pem>
ca_certs = <path to ca.pem>
cert_required = False

[ssl]
ca_key = <path to cakey.pem>
key_size = 1024
valid_days=3650
cert_subject=/C=US/ST=Unset/L=Unset/O=Unset/CN=localhost

keystone-manage ssl_setup
```

配置样例如下：

```
global
    ...
 
defaults
	...

frontend ft_web
    bind <Virtual IP>:443 ssl crt /etc/haproxy/https.pem #证书
    reqadd X-Forwarded-Proto:\ https #相关配置
    default_backend bk_web
 
backend bk_web
    balance roundrobin
    redirect scheme https if !{ ssl_fc } #相关配置
    server http-server-01 private_IP1:8080 check inter 10s
    server http-server-02 private_IP2:8080 check inter 10s
```

----------
#DDOS

Haproxy 可在一定程度上提高服务端抗击 DDOS 攻击能力，DDOS 主要有 UDP，TCP sync flood，Http slowloris attack 等攻击手段，[use-a-load-balancer-as-a-first-row-of-defense-against-ddos](http://blog.haproxy.com/2012/02/27/use-a-load-balancer-as-a-first-row-of-defense-against-ddos/) 详细的阐述了配置过程，主要表现在以下几个方面：

- TCP syn flood attacks
- Slowloris like attacks
- Limiting the number of connections per users
- Limiting the connection rate per user

##TCP syn flood attacks

通过向服务器发送大量的 TCP syn 分组，恶意用户实现了了 TCP syn flood 攻击，幸运的是，简单的配置内核网络参数即可防止这种攻击。 

```bash
/etc/sysctl.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_max_syn_backlog = 1024 

sysctl -p
```

----------
##Slowloris like attacks
 
一个 HTTP 请求通常包括头部、url、methods 等，服务器需要接收整个请求后才能做出响应。如果恶意用户发送缓慢的请求，比如一个字节一个字节的发送头部，服务器将一直处于 waiting 状态，耗费服务器的资源。通过配置 timeout http-request 参数，当一个用户的请求时间超过设定值时，haproxy 自动断开该连接。


```
defaults
  option http-server-close
  mode http
  timeout connect 10s  #限制链接时间，该参数不宜过大
  timeout server  15s  #限制链接时间，该参数不宜过大
  timeout client  15s  #限制链接时间，该参数不宜过大

frontend ft_web
	...

backend bk_web
	...

```

通过 telnet 登录验证结果：

```bash
telnet 127.0.0.1 8080
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
HTTP/1.0 408 Request Time-out
Cache-Control: no-cache
Connection: close
Content-Type: text/html
408 Request Time-out
Your browser didn't send a complete request in time.
Connection closed by foreign host.
```

----------

##Limiting the number of connections per users
 
普通用户访问某个 OpenStack 服务时，如果恶意打开了大量 TCP 链接，会耗费服务器内存资源，影响其它用户的体验，因此我们需要根据实际情况，限制同一个用户的链接并发数。

```
defaults
	...

frontend ft_web
  bind <Virtual IP>:8080
  # Allow clean known IPs to bypass the filter
  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst } #白名单
  # Shut the new connection as long as the client has already 10 opened  #限制单个用户的连接数
  tcp-request connection reject if { src_conn_cur ge 10 }                #限制单个用户的连接数
  tcp-request connection track-sc1 src</span>

  default_backend bk_web

backend bk_web
	...

```

**注**：若某些用户在同一个私有网段通过 NAT 访问网站，这样的配置存在不合理之处，最好把 NAT 处的公网地址添加到 whitelist.lst 文件中。

利用 apache 测试工具做验证，和服务器一直保持建立 10 个链接。

```bash
ab -n 50000000 -c 10 http://127.0.0.1:8080/
用 telnet 打开第 11 个链接，服务器拒绝该链接。
telnet 127.0.0.1 8080
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Connection closed by foreign host.
```
----------

##Limiting the connection rate per user

仅仅限制单个用户的并发链接数并意味着万事大吉，如果用户在短时间内向服务器不断的发送建立和关闭链接请求，也会耗费服务器资源，影响服务器端的性能，因此需要控制单个用户的访问速率。通常情况下，我们认为普通用户在 3 秒内不应该向某个 openstack api 发送的请求不应该超过 20 次。

```
defaults
	...

frontend ft_web
  bind <Virtual IP>:8080
  stick-table type ip size 100k expire 30s store conn_cur, conn_rate(3s) # Table definition 

  # Allow clean known IPs to bypass the filter
  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst } #白名单
  # Shut the new connection as long as the client has already 10 opened or rate more than 20
  tcp-request connection reject if { src_conn_cur ge 10 }|| { src_conn_rate ge 20} #限制单个用户链接数和访问速率
  tcp-request connection track-sc1 src

  default_backend bk_web

backend bk_web
	...
```

测试步骤如下：

1. 采用 ab 打开 20 个链接。(本次测试把 限制单个用户并发数功能 去掉)
2. ab -n 20 -c 1 -r http://127.0.0.1:8080/
3. 再用 telnet 打开第 21 个链接，服务器拒绝该请求。

```bash
telnet 127.0.0.1 8080
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Connection closed by foreign host.
```

----------

#样例 nova-api

根据上述内容，以 nova-api 为例，其配置如下：

```
global
  chroot  /var/lib/haproxy
  daemon
  group  haproxy
  maxconn  4000
  pidfile  /var/run/haproxy.pid
  user  haproxy

defaults
  log  global
  maxconn  4000
  option  redispatch
  retries  3
  timeout  http-request 10s
  timeout  queue 1m
  timeout  connect 10s
  timeout  client 15s
  timeout  server 15s
  timeout  check 10s

frontend nova_api_VIP
  bind <Virtual IP>:443 ssl crt /etc/haproxy/https.pem
  reqadd X-Forwarded-Proto:\ https

  stick-table type ip size 100k expire 30s store conn_cur, conn_rate(3s)
  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
  tcp-request connection reject if { src_conn_cur ge 10 }|| { src_conn_rate ge 20}
  tcp-request connection track-sc1 src
  
  default_backend nova-api-cluster

frontend nova-api-cluster
  balance  source
  option  httpchk
  redirect scheme https if !{ ssl_fc }
  server controller1 10.0.0.1:8774 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.2:8774 check inter 2000 rise 2 fall 5
  server controller3 10.0.0.3:8774 check inter 2000 rise 2 fall 5

```
