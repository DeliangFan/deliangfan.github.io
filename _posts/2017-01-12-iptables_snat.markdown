---
layout: post
title:  "同局域网下的 Iptables DNAT"
categories: 踩坑杂记
---

某业务运行在服务器 A(10.0.0.2:80)，后来将其迁移至服务器 B(10.0.0.3:80)，为了保证平滑迁移，服务器 A 需要将 80 端口的入口流量转发至服务器 B 的 80 端口。[Iptables](http://ipset.netfilter.org/iptables.man.html) 强大的端口转发功能可以很好的满足该需求，因为服务器 A 需要将入口流量转发，因此使用 [DNAT](http://linux-ip.net/html/nat-dnat.html)。[官网文档](https://www.netfilter.org/documentation/HOWTO/NAT-HOWTO-6.html)的步骤如下：

开启转发功能：

~~~ shell
$ sysctl -w net.ipv4.ip_forward=1
~~~

设置如下转发规则，即把 A 的 80 端口的流量转发到服务器 B 的 80 端口。

~~~ shell
$ iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to  10.0.0.3:80
~~~

然后从客户端(10.0.0.10)访问服务器 A 的 80 端口，但是没有回应，原因是超时。

~~~ shell
$ curl http://10.0.0.2:80
curl: (7) Failed to connect to 10.0.0.2 port 80: Operation timed out
~~~

服务器 A 的抓包如下，由第一行可知服务器 A 的 80 端口收到请求，第二行表示服务器 A 将 80 端口的流量转发至服务器 B 的 80 端口(请注意相同的 TCP 序列号)。

~~~ shell
$ tcpdump port 80
... IP 10.0.0.10.52799 > 10.0.0.2.80 Flags [S], seq 100071038, ... length 0
... IP 10.0.0.10.52799 > 10.0.0.3.80 Flags [S], seq 100071038, ... length 0
... (repeat the two lines above)
~~~

服务器 B 的抓包如下，第一行表示服务器 B 收到 TCP 三次握手中的 sync 报文，之后服务器回送 sync 报文(第二行)，第三行表示收到客户端 reset 报文。客户端之所以发送 reset 报文，是因为客户端是和服务器 A 而非 B 建立的 TCP 链接，所以服务器 B 回送 sync 报文时，客户端并不认识服务器 B，故客户端重置连接。

~~~ shell
$ tcpdump port 80
... IP 10.0.0.10.52799 > 10.0.0.3.80: Flags [S], seq 100071038 ... length 0
... IP 10.0.0.3.80 > 10.0.0.10.52799: Flags [S.], seq 2076971142, ack 100071039 ... length 0
... IP 10.0.0.10.52799 > 10.0.0.3.80: Flags [R], seq 100071039 ... length 0
~~~

分析清楚原因后，对于该问题，解决的办法有多种，最为便捷的是在服务器 A 对 80 端口的流量上再做 SNAT。

~~~ shell
$ iptables -t nat -A POSTROUTING -d 10.0.0.3 -p tcp --dport 80 -j SNAT --to-source 10.0.0.2
~~~

Serverfault 亦有人提出类似问题 [how-to-do-the-port-forwarding-from-one-ip-to-another-ip-in-same-network](http://serverfault.com/questions/586486/how-to-do-the-port-forwarding-from-one-ip-to-another-ip-in-same-network)。