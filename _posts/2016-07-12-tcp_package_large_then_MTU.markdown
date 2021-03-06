---
layout: post
title:  "tcpdump 抓了大于 MTU 的包"
categories: 踩坑杂记
---

在一次抓包中，意外的发现许多包竟高达 18 KB 时，我是非常惊奇的，因为我确信局域网内的两主机网卡的 MTU 均是 1500 Byte。如下是在发送端抓取的部分包：

~~~
$ tcpdump -i eth0 src net src_ip and src port 39200
...
IP src_ip.39200 > dst_ip.22: Flags [.], seq 46101424:46120248, ack 12673, win 296, options [nop...], length 18824
IP src_ip.39200 > dst_ip.22: Flags [.], seq 46120248:46139072, ack 12673, win 296, options [nop...], length 18824
IP src_ip.39200 > dst_ip.22: Flags [.], seq 46139072:46156448, ack 12673, win 296, options [nop...], length 17376
IP src_ip.39200 > dst_ip.22: Flags [.], seq 46156448:46157896, ack 12673, win 296, options [nop...], length 1448
...
~~~

两主机网卡的 MTU 信息分别为：

~~~
$ ifconfig en0
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
...

$ ifconfig eth0
eth0      Link encap:Ethernet  HWaddr ...
          ... 
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
...
~~~

Google 大法迅速得出答案，一篇名为 [how-can-the-packet-size-be-greater-than-the-mtu](http://packetbomb.com/how-can-the-packet-size-be-greater-than-the-mtu/) 给出了很好的解释。不仅是 linux 下的 tcpdump，还是 windows 下的 wireshark，均会遇上相同的问题。原因在于系统开启了 [TSO(TCP Segment Offload)](https://en.wikipedia.org/wiki/Large_segment_offload)。

为了降低 CPU 的负载，提高网络的出口带宽，TSO 提供一些较大的缓冲区来缓存 TCP 发送的包，然后由网卡负责把缓存的大包拆分成多个小于 MTU 的包。tcpdump 或者 wireshare 抓取的是网卡上层的包，所以我们可能会观察到大小超过 MTU 的包：

~~~
        +---------------+
        |  Application  |
        +---------------+
        |    TCP/IP     |
        +---------------+
        |  nit_if/PCAP  | <--- tcpdump / wireshark
        +---------------+
        |      Nic      | Split the big package into separate packets
        +---------------+
                |
                |         Network(Packets size less than MTU)
                +---------------------------------------------------->
~~~

如果在交换机或者目的主机处，抓取的包肯定都小于 MTU，如下是同时在目的主机抓取的部分包，其大小多为 1448 Byte：

~~~
$ tcpdump -i eth0 src net src_ip and src port 39200
...
IP src_ip.39200 > dst_ip.22: Flags [.], seq 76884456:76885904, ack 21097, win 296, options [nop...], length 1448
IP src_ip.39200 > dst_ip.22: Flags [.], seq 76885904:76887352, ack 21097, win 296, options [nop...], length 1448
IP src_ip.39200 > dst_ip.22: Flags [.], seq 76887352:76888800, ack 21097, win 296, options [nop...], length 1448
IP src_ip.39200 > dst_ip.22: Flags [.], seq 76888800:76890248, ack 21097, win 296, options [nop...], length 1448
IP src_ip.39200 > dst_ip.22: Flags [.], seq 76890248:76891696, ack 21097, win 296, options [nop...], length 1448
...
~~~
