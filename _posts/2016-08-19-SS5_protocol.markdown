---
layout: post
title:  "实现一个简单的 SOCK V5 代理服务器 --- 协议"
categories: Python 
---


![twitter](http://7xp2eu.com1.z0.glb.clouddn.com/twitter_ss5.png)


[SOCK](https://zh.wikipedia.org/wiki/SOCKS) 基于 TCP 之上，属于会话层协议，最初由 David Koblas 提出，第一个成熟通用的版本是 NEC 开发的 [SOCK V4](http://www.openssh.com/txt/socks4.protocol)，但是它不支持认证，不支持代理 UDP 协议，所以功能更为强大的 [SOCK V5(RFC 1928)](https://www.ietf.org/rfc/rfc1928.txt) 运用而生。 

~~~
                        Firewall
                           ++
                           ||
                           ++                            
                           ||
+--------+           +-----++------+             +--------------------+
| Client | <-------> | SOCK Server | <---------> | Destination Server |
+--------+           +-----++------+             +--------------------+
                           ||
                           ++
~~~

当使用 SOCK V5 代理服务器代理防火墙内的客户端访问外部网络时，客户端首先需要和代理服务器建立连接，协商认证方式，认证通过后将客户端的请求转发至外部服务器。大体的步骤如下：

1. Client 发送请求至 SOCK 服务器，协商认证的方法
2. SOCK 服务器回复认证方法
3. 按协商的方法进行认证(略)
4. Client 发送请求至 SOCK 服务器，告知外部服务器的地址和端口
5. SOCK 服务器评估后，与外部服务器建立链接，返回自身地址信息
6. SOCK 服务器把客户端的请求转发至外部服务器，把外部服务器的返回信息转发回 Client

下面以访问 https://twitter.com 为例，讲述每个步骤的细节，在第一步的请求中，Client 端的报文格式为：

~~~
+----+----------+----------+
|VER | NMETHODS | METHODS  |
+----+----------+----------+
| 1  |    1     | 1 to 255 |
+----+----------+----------+

VER:      set to X'05' for this version of the protocol
NMETHODS: the number of method identifier octets that appear in the METHODS field
METHODS:  the values currently defined for METHOD are
       o  X'00' NO AUTHENTICATION REQUIRED
       o  X'01' GSSAPI
       o  X'02' USERNAME/PASSWORD
       o  X'03' to X'7F' IANA ASSIGNED
       o  X'80' to X'FE' RESERVED FOR PRIVATE METHODS
       o  X'FF' NO ACCEPTABLE METHODS
~~~

本文实现的 SOCK 服务器暂时不支持认证，所以 METHODS 设置为 X'00'，具体的报文为：

~~~
'\x05\x01\x00'
~~~

SOCK 服务器收到请求后，选择一种双方都支持的认证方法(如果有)，返回的报文格式为：

~~~
+----+--------+
|VER | METHOD |
+----+--------+
| 1  |   1    |
+----+--------+
~~~

本例具体的报文为：

~~~
'\x05\x00'
~~~

第 3 步是认证步骤，SOCK V5 支持的认证方法主要有：

- NO AUTHENTICATION REQUIRED: 不需要认证
- GSSAPI: 详见 [GSS-API Authentication Method for SOCKS Version 5 (RFC 1961)](https://tools.ietf.org/html/rfc1961)
- USERNAME/PASSWORD: 详见[Username/Password Authentication for SOCKS V5
(RFC 1929)](https://tools.ietf.org/html/rfc1929)

因本文实现的 SOCK 服务器没有实现认证，所以略过该步骤。

之后 Client 发送请求至 SOCK 服务器，告知需要访问的外部服务器的地址和端口，报文格式如下：

~~~
+----+-----+-------+------+----------+----------+
|VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
+----+-----+-------+------+----------+----------+
| 1  |  1  | X'00' |  1   | Variable |    2     |
+----+-----+-------+------+----------+----------+

VER:  protocol version, X'05'
CMD:
      o  CONNECT X'01'
      o  BIND X'02'
      o  UDP ASSOCIATE X'03'
RSV:  RESERVED, X'00'
ATYP: address type of following address
      o  IP V4 address: X'01'
      o  DOMAINNAME: X'03'
      o  IP V6 address: X'04'
DST.ADDR:  desired destination address
DST.PORT:  desired destination port in network octet
        order
~~~

注意到 CMD 参数，多数协议使用的是 CONNECT，某些需要两条 TCP 链接的协议必须使用 BIND，如 FTP 协议，UDP 表示代理的是 UDP 协议，本文实现的 SOCK 服务器只支持了 CONNECT 模式，具体的报文如下：

~~~
'\x05\x01\x00\x03\x0btwitter.com\x01\xbb'
~~~ 

SOCK 服务器评估该请求，并与外部服务器建立 TCP 链接，返回的报文格式如下：

~~~
+----+-----+-------+------+----------+----------+
|VER | REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
+----+-----+-------+------+----------+----------+
| 1  |  1  | X'00' |  1   | Variable |    2     |
+----+-----+-------+------+----------+----------+

VER:   protocol version, X'05'
REP:   Reply field:
       o  X'00' succeeded
       o  X'01' general SOCKS server failure
       o  X'02' connection not allowed by ruleset
       o  X'03' Network unreachable
       o  X'04' Host unreachable
       o  X'05' Connection refused
       o  X'06' TTL expired
       o  X'07' Command not supported
       o  X'08' Address type not supported
       o  X'09' to X'FF' unassigned
RSV:   RESERVED, X'00'
ATYP:  address type of following address
       o  IP V4 address: X'01'
       o  DOMAINNAME: X'03'
       o  IP V6 address: X'04'
BND.ADDR:   server bound address
BND.PORT:   server bound port in network octet order
~~~

注意到 REP，如果 SOCK 服务器与 twitter.com 成功建立链接，REP 的返回值为 X'00'，否则为其它类型的错误，如 Network unreachable，Connection refused 等等。本例 SOCK 服务器与 twitter.com 成功建立链接，返回给 Client 的具体报文如下：

~~~
'\x05\x00\x00\x01\xac\x1f\x1c\x8e\x048'
~~~

之后进入第 6 步，SOCK 服务器把客户端的请求转发至外部服务器，把外部服务器的返回信息转发给 Client。所有代码如下：

~~~ python
import gevent
from gevent import monkey
from gevent.pool import Pool
from gevent import select
from gevent.server import StreamServer
from gevent import socket


monkey.patch_all()

BUFFER = 4096
SOCK_V5 = 5
RSV = 0
ATYP_IP_V4 = 1
ATYP_DOMAINNAME = 3
CMD_CONNECT = 1
IMPLEMENTED_METHODS = (2, 0)


class SockV5Server(object):

    def __init__(self, port=1080):
        self.port = port
        self.pool = Pool(1000)
        self.server = StreamServer(('0.0.0.0', self.port),
                                   self.handler)

    def close_sock_and_exit(self, client_sock=None, server_sock=None):
        if client_sock:
            if not client_sock.closed:
                client_sock.close()

        if server_sock:
            if not server_sock.closed:
                server_sock.close()

        g = gevent.getcurrent()
        g.kill()

    def process_version_and_auth(self, client_sock):
        recv = client_sock.recv(BUFFER)
        if ord(recv[0]) != SOCK_V5:
            self.close_sock_and_exit(client_sock)

        method = None
        num_methods = ord(recv[1])
        methods = [ord(recv[i + 2]) for i in range(num_methods)]
        for imp_method in IMPLEMENTED_METHODS:
            if imp_method in methods:
                method = imp_method
                break

        if method is None:
            self.close_sock_and_exit(client_sock)

        send_msg = '\x05' + chr(method)
        client_sock.send(send_msg)

    def process_sock_request(self, client_sock):
        recv = client_sock.recv(BUFFER)
        if ord(recv[0]) != SOCK_V5 or ord(recv[2]) != RSV:
            self.close_sock_and_exit(client_sock)

        addr_type = ord(recv[3])
        if addr_type == ATYP_IP_V4:
            addr = socket.inet_ntoa(recv[4:8])
        elif addr_type == ATYP_DOMAINNAME:
            addr_len = ord(recv[4])
            addr = socket.gethostbyname(recv[5:5 + addr_len])
        else:
            # only ipv4 addr or domain name is supported.
            self.close_sock_and_exit(client_sock)

        port = ord(recv[-2]) * 256 + ord(recv[-1])

        cmd = ord(recv[1])
        if cmd == CMD_CONNECT:
            # Only connect cmd is supported.
            server_sock = self.connect_target_server_and_reply(client_sock,
                                                               addr, port, cmd)
        else:
            self.close_sock_and_exit(client_sock)

        return server_sock

    def connect_target_server_and_reply(self, client_sock, addr, port, cmd):
        sock_name = client_sock.getsockname()
        server_hex_addr = socket.inet_aton(sock_name[0])
        server_hex_port = self.port_to_hex_string(sock_name[1])
        server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

        try:
            server_sock.connect((addr, port))
            send_msg = '\x05\x00\x00\x01' + server_hex_addr + server_hex_port
            client_sock.send(send_msg)
        except Exception:
            send_msg = '\x05\x01\x00\x01' + server_hex_addr + server_hex_port
            client_sock.send(send_msg)
            self.close_sock_and_exit(client_sock)

        return server_sock

    def piping_client_and_target(self, client_sock, server_sock):
        inputs = [client_sock, server_sock]
        while True:
            try:
                in_ready, out_ready, ex_ready = select.select(inputs, [], [])
                for sock in in_ready:
                    if sock == client_sock:
                        self.recv_and_send_msg(client_sock, server_sock)
                    elif sock == server_sock:
                        self.recv_and_send_msg(server_sock, client_sock)
            except Exception:
                self.close_sock_and_exit(client_sock, server_sock)
    def recv_and_send_msg(self, recv_sock, send_sock):
        # recv() is a block I/O, it returns '' when remote has been closed.
        msg = recv_sock.recv(BUFFER)
        if msg == '':
            self.close_sock_and_exit(recv_sock, send_sock)

        send_sock.sendall(msg)

    def port_to_hex_string(self, int_port):
        port_hex_string = chr(int_port / 256) + chr(int_port % 256)
        return port_hex_string

    def handler(self, client_sock, address):
        self.process_version_and_auth(client_sock)
        server_sock = self.process_sock_request(client_sock)
        self.piping_client_and_target(client_sock, server_sock)

    def serve_forever(self):
        self.server.serve_forever()


if '__main__' == __name__:
    sock_v5_server = SockV5Server(1080)
    sock_v5_server.serve_forever()
~~~

最后推荐两个 SS5 的开源库：

- [SS5](https://github.com/twg/ss5): C 语言版
- [shadowsocks](https://github.com/shadowsocks/shadowsocks): Python 语言版
