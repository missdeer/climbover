---
layout: post
title: 移动端科学上网的需求
date: 2022-05-02 12:13:14 +0800
category: mobile
---
# 移动端的常用解决方案

&emsp;&emsp;其他名不见经传的~~商用~~解决方案就不提了。

## Android

&emsp;&emsp;Android端早期几乎是靠[Max Lv（GitHub id：madeye）](https://github.com/madeye)的一己之力撑起来的，最早的[GAEProxy](https://github.com/madeye/gaeproxy)，[ProxyDroid](https://github.com/madeye/proxydroid)，后来的[Shadowsocks for Android](https://github.com/shadowsocks/shadowsocks-android)（中文名：影梭），都是使用甚广。他使用的代理技术（早期靠root后用iptables，就是[基于通用Linux发行版的科学上网网关](/gateway/2022/04/27/common-linux-distribution-based-gateway)使用的技术，后期不需要root了就靠系统VPN service）为后来的各种代理app提供了巨大的参考价值，甚至有一些app直接套用了Shadowsocks for Android的源代码，把核心代理协议换掉就行了。

&emsp;&emsp;后来新出来的代理app主要做了两方面的工作，一个是换协议，比如换成v2ray的vmess/vless，换成trojan，另一个是换交互，比如基于规则的配置，现在最流行的大概就是clash了。移动端技术上没什么值得多说的，反正沿用Shadowsocks for Android那一套框架就行了。

## iOS

&emsp;&emsp;iOS端早期也是靠越狱解决翻墙问题的，比如有个叫gfwinterceptor的app在越狱后可以从cydia安装，总之比较麻烦。以前人们有很多理由给iOS越狱，翻墙反正是排后面的理由，随着技术的推陈出新，iOS系统及API、app不断丰富，人们软件版权意识也逐步增强，越狱的人越来越少，直到后来连cydia都没有了。

&emsp;&emsp;从苹果推出Network Extension接口后，第一个大流行的翻墙app应该是[Surge](https://nssurge.com/)了，作者一直声称Surge不是个翻墙软件而是个网络调试工具，后来产品方向上确实做出了一些改变，不过早期在国内靠切实的翻墙功能（比如内建支持shaodowsocks协议）实实在在割了一大波韭菜。后来又陆续出现了几个翻墙app，比如[Shadowrocket](https://apps.apple.com/us/app/shadowrocket/id932747118)，[Quantumult](https://apps.apple.com/us/app/quantumult/id1252015438)以及[Quantumult  X](https://apps.apple.com/us/app/quantumult-x/id1443988620)等等。注意，由于国内政策的限制，这些app全都没有上架中国区。

&emsp;&emsp;值得一提的是，iOS上的翻墙app无一例外是付费的，而Android上的都是免费的，从中可以看到两个系统生态的情况。

# 统一的移动端免费解决方案

&emsp;&emsp;如果是从前面的文章一路跟下来的，那么到现在为止， 已经有了一个解决了DNS污染，国内外路线分流，CDN友好的翻墙网关。我假设这个网关设备最好是挂在ISP光猫后，或一级路由器后面，也就是说，从外面连进来只要在光猫或一级路由器上做一次端口映射，就能访问到这个设备了。那么现在要解决移动端的翻墙需求其实非常简单了。

&emsp;&emsp;**注意这里移动端，不仅仅包括手机端，还包括笔记本电脑，涉及到多种不同的硬件和操作系统。**

&emsp;&emsp;首先，要让家里的网络拥有公网IP，这点在国内绝大多数地区的一级运营商（电信、移动、联通）都是可以解决的，电信基本上默认就有，如果默认没有就打客服电话申诉，就说自己要装监控等等找个合理的理由即可。

&emsp;&emsp;有了公网IP后，需要给它绑定一个域名，目前网上有一些提供动态域名的服务，比如[花生壳](https://hsk.oray.com/)，具体操作和注意事项看他们官方网站即可。也可以自己注册一个便宜的域名，托管到DNSPOD或Cloudflare上，然后运行一个程序（网上有很多这种小程序，随便搜一个就行，会写代码的自己写一个也行，比如我就[自己写了一个](https://github.com/missdeer/ddnsclient)）周期性地检测IP变化并更新域名设置。

&emsp;&emsp;如果实在搞不到公网IP，那只能买一个便宜的VPS，然后用[frp](https://github.com/fatedier/frp)之类做内网穿透，这是没办法的办法了。当然也可以和人一起拼车域名和VPS，以节省一些开支，一个人只用来做内网穿透也太浪费了。

&emsp;&emsp;然后，在这个网关设备上安装OpenVPN服务端，网上有一个[一键安装脚本](https://github.com/Nyr/openvpn-install)，非常简单易用，至今仍在积极维护：

```shell
wget https://git.io/vpn -O openvpn-install.sh
chmod a+x openvpn-install.sh
sudo ./openvpn-install.sh
```

&emsp;&emsp;安装完成后，稍微修改一下配置文件：

```shell
cd /etc/openvpn/server
sudo vi server.conf
```

&emsp;&emsp;主要是把VPN使用的IP段改得不容易和其他软件使用的IP段冲突，以及设置一下DNS，比如：

```txt
server 10.11.12.0 255.255.255.0
push "dhcp-option DNS 10.11.12.1"
```

&emsp;&emsp;然后重启服务：

```shell
sudo systemctl restart openvpn-server
```

&emsp;&emsp;接下来，创建几个客户端使用的证书：

```shell
sudo ./openvpn-install.sh
```

&emsp;&emsp;第一步选1，第二步随意输入一个名字即可，如下图所示：

![add OpenVPN client user](/public/img/2022-05-02/1.png)

&emsp;&emsp;这样就得到一个客户端使用的证书，想办法把这个文件发给客户端导入即可。

&emsp;&emsp;OpenVPN客户端软件可以从[官网](https://openvpn.net/vpn-client/)下载，手机端会跳转到对应的应用商店页面，注意iOS仍然没有上架中国区，可以注册一个美国区的Apple Id，免费下载使用。

&emsp;&emsp;导入证书操作在手机端也比电脑端稍微麻烦一点，可以把证书文件放在一个HTTP服务器上，通过浏览器下载，也可以通过电子邮件以附件的形式发送，在手机app里打开附件时用OpenVPN客户端打开。可以为每一个设备创建一个客户端证书，不会冲突，也随时可以吊销证书。

&emsp;&emsp;之后只要打开连接开关，OpenVPN就会连到翻墙网关上，手机端的界面大体上是这样的，Android端和iOS端几乎一样：

![OpenVPN client on Android](/public/img/2022-05-02/openvpn-android.jpg)

&emsp;&emsp;到此为止，就可以让所有笔记本电脑和手机在外也能连回家里的翻墙网关，享受到无污染的DNS，流量分流的翻墙服务。所有的客户端使用相同的方案，减少维护成本，基本上只要把网关维护好，那么所有设备都可以直接享受到所有便利。最重要的一点是这一切几乎不需要另外花钱。