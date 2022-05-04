---
layout: post
title: 与其他网络互联
date: 2022-05-04 13:14:15 +0800
category: vpn
---
# 为什么要与其他网络互联

&emsp;&emsp;现在只要设备接入家里的网络，便自动获得了翻墙能力，不但DNS是CDN友好的，线路也是CDN友好的。

&emsp;&emsp;但是有了新的需求，与其他网络互联。为什么有这样的需求的，举个最常见的例子，公司网络是一个封闭的网络，需要VPN接入，一般情况往往是给一个客户端，一个账号密码或证书，同一时间只能供一个设备使用。不过我希望无论我的手机、电脑都能随时接入公司网络，同时又不失去翻墙的能力，而且最好不需要每个设备都装一遍客户端。

&emsp;&emsp;从简单操作网关的经历来看，其实原理很简单，只要在网关上运行公司的VPN客户端，再把路由设置一下，就能达到想要的效果。

# 实现步骤

&emsp;&emsp;以我们公司的情况为例，公司使用OpenVPN，在网关上安装OpenVPN客户端，然后把客户端证书拷贝过去，先打开证书文件看一下，是否需要修改，比如DNS和路由设置可以根据自己的需要来修改：

```
dhcp-option DNS 192.168.1.166
route 172.16.0.0 255.255.0.0
```

&emsp;&emsp;第1行自定义了DNS服务器，第2行加入了额外的路由表项。

&emsp;&emsp;然后运行客户端启动命令：

```bash
sudo /usr/sbin/openvpn --config /etc/openvpn/client/mywork.conf --daemon
```

&emsp;&emsp;如果能正常跑起来，用`ip a s`命令，应该能看到多出一个`tun`连接，如下图所示：

![ip a s](/public/img/2022-05-04/ip-a-s.png)

&emsp;&emsp;第1个`lo`是本地回环连接，第2个`ens18`是真正的网卡的连接，第3个`tun0`就是VPN新加的连接。

&emsp;&emsp;再用`ip route`命令看一下系统的路由表，如下图所示：

![ip route](/public/img/2022-05-04/ip-route.png)

&emsp;&emsp;说明OpenVPN客户端正常运行了。

&emsp;&emsp;最后，改一下iptables的nat链：

```bash

iptables -t nat -A POSTROUTING -s 10.8.0.0/16 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE
iptables -t nat -A SS -d 10.8.0.0/16 -j RETURN
iptables -t nat -A SS -d 172.16.0.0/16 -j RETURN
```

&emsp;&emsp;这里假设公司内网用到的IP段有两个，分别是`10.8.0.0/16`和`172.16.0.0/16`，加入后应该所有设备都能直接访问公司内网的能力了，而且因为只有这两个IP段是走公司VPN的，原有的翻墙能力并不受影响。