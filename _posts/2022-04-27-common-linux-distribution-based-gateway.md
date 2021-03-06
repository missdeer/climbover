---
layout: post
title: 基于通用Linux发行版的科学上网网关
date: 2022-04-27 09:10:11 +0800
category: gateway
---
# 什么是科学上网网关

&emsp;&emsp;设备无论是电脑还是手机，要上网就需要网络连接，在一个正常可用的网络连接中，无论是自动获取还是手动配置，一般会有4项基本要素：本机IP，子网掩码，网关（苹果系统中叫路由器）IP，以及DNS服务器IP。

&emsp;&emsp;其中网关，我们可以简单地认为，设备上网时所有流量进出都是经过网关中转的。而科学上网网关，顾名思义就是我们在网关上做好各项设置，只要设备的网络连接中设置使用了此网关，那么设备就自动获得到科学上网的能力，不需要在设备上做任何额外的设置。现在网上更多地将其称为“**旁路由**”。

# 为什么基于通用Linux发行版

&emsp;&emsp;现在国内家用路由器或网关进行科学上网的系统，基本被**OpenWRT**一统天下了，网上各种魔改版层出不穷，来源纷乱，安全性可靠性都未经证明，所以我使用基本通用的Linux发行版作为网关的操作系统。

&emsp;&emsp;通用Linux发行版有很多，最流行的大体上分2大系：**Debian**系和**RedHat**系，非常多的其他发行版都是从这2者派生出来的子子孙孙，比如非常有名且用户众多的Ubuntu和CentOS就分别是这2系派生发行版的代表。另外树莓派早期的官方系统Raspbian就是Debian移植的，很多同类小板子都支持了Raspbian系统。

&emsp;&emsp;使用通用Linux发行版的原因，除了前面提到的信不过OpenWRT之外，最主要的好处是软件和文档非常丰富，遇到问题很容易在网上找到解决方案。还有一个原因是，我用来做网关的硬件设备比一般家用小路由器配置高得多，比如曾经用淘汰下来的笔记本做网关，用OpenWRT简直是小马拉大车，而通用Linux发行版功能更多，可玩性比OpenWRT更好。

# 实现步骤

&emsp;&emsp;首先准备好硬件设备，基本要求是能装Linux，我选用的是Debian，因为用得最多最熟悉。

&emsp;&emsp;系统装好后，设置流量转发。打开`/etc/sysctl.conf`，设置`net.ipv4.ip_forward=1`，让更新实时生效: `sysctl -p /etc/sysctl.conf`。这个设备就已经可以做为网关使用了。

&emsp;&emsp;然后要让流量全部由代理程序代理转发。现在的常见的代理软件（如shadowsocks, v2ray, trojan, clash等等），只要是支持Linux系统的，基本都实现了一种叫`redir`的模式。这里以[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)版本为例，编译出来后有一个叫`ss-redir`的可执行文件，这个程序就是在本地打开一个端口并以`redir`方式工作。以下面的命令行为例：

```shell
ss-redir -s 123.123.123.123 -p 6789 -k qwerasdfzxcv -m chacha20-ietf -l 58100 -b 0.0.0.0 -f ss-redir.pid
```

&emsp;&emsp;其中`123.123.123.123`是服务器地址，`6789`是服务器端口，`qwerasdfzxcv`是密码，`chacha20-ietf`是传输协议使用的加密算法，`58100`是本地监听的以`redir`模式运行的端口，`0.0.0.0`则是指定监听端口绑定的地址，一定要用`0.0.0.0`才能让局域网中其他设备连上来，否则只能本机连接。最后的`ss-redir.pid`是给程序一个写入该进程id的文件名，这样程序会切换到后台运行，而不会占用前台。

&emsp;&emsp;接下来将流量转发到ss-redir开的端口（上面用的是58100）上去，Linux上用iptables：

```shell
iptables -F
iptables -X
iptables -Z

iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

iptables -A INPUT -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp --dport 58100 -m state --state NEW,ESTABLISHED -j ACCEPT

# dnat
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -j MASQUERADE
iptables -t nat -N SS
# 过滤私有地址，shadowsocks服务器地址
iptables -t nat -A SS -d 127.0.0.1 -j RETURN
iptables -t nat -A SS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SS -d 172.16.0.0/21 -j RETURN
iptables -t nat -A SS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SS -d 123.123.123.123 -j RETURN

# 过滤中国ip地址，网上搜一下“中国ip段"
iptables -t nat -A SS -d 58.14.0.0/15 -j RETURN
iptables -t nat -A SS -d 58.16.0.0/13 -j RETURN
iptables -t nat -A SS -d 58.24.0.0/15 -j RETURN

# 所有其他的ip，都提交给shadowsocks
iptables -t nat -A SS -p tcp -j REDIRECT --to-port 58100

# 使用SS链，其他设备走PREROUTING链
iptables -t nat -A PREROUTING -p tcp -j SS
# 使用SS链，自己走OUTPUT链
iptables -t nat -A OUTPUT -p tcp -j SS
```

&emsp;&emsp;中间有一段中国IP地址的，以后有一篇专门讲流量分流的文章仔细讲一下怎么弄，这里可以先不管。

&emsp;&emsp;最后，把家里所有要翻墙的设备都在网关一项里填成此设备的IP。比较方便的做法是，在DHCP那里就直接返回这个IP当网关，但要注意把这个设备的网络连接里设成手动设置信息，此设备的网关仍然是原来路由器的IP，不能被DHCP里自动获取的冲掉。

&emsp;&emsp;到此为止，基本实现了所有设备自动获得流量科学上网的能力。但还有一个DNS污染的问题没有解决，如果某个域名是被污染的，那仍然不能正常访问，下一篇文章将解决这个问题。