---
layout: post
title: 移动端科学上网的需求
date: 2022-04-30 12:13:14 +0800
category: mobile
---
# 移动端的常用解决方案

&emsp;&emsp;其他名不见经传的~~商用~~解决方案就不提了。

## Android

&emsp;&emsp;Android端早期几乎是靠[Max Lv（GitHub id：madeye）](https://github.com/madeye)的一己之力撑起来的，最早的[GAEProxy](https://github.com/madeye/gaeproxy)，[ProxyDroid](https://github.com/madeye/proxydroid)，后来的[Shadowsocks for Android](https://github.com/shadowsocks/shadowsocks-android)（中文名：影梭），都是使用甚广。他使用的代理技术（早期靠root后用iptables，就是[基于通用Linux发行版的科学上网网关](/gateway/2022/04/27/common-linux-distribution-based-gateway)使用的技术，后期不需要root了就靠系统VPN service）为后来的各种代理app提供了巨大的参考价值，甚至有一些app直接套用了Shadowsocks for Android的源代码，把核心代理协议换掉就行了。

&emsp;&emsp;后来新出来的代理app主要做了两方面的工作，一个是换协议，比如换成v2ray的vmess/vless，换成trojan，另一个是换交互，比如基于规则的配置，现在最流行的大量就是clash了。移动端技术上没什么值得多说的，反正沿用Shadowsocks for Android那一套框架就行了。

## iOS

&emsp;&emsp;iOS端早期也是靠越狱解决翻墙问题的，比如有个叫gfwinterceptor的app在越狱后可以从cydia安装，总之比较麻烦。以前人们有很多理由给iOS越狱，翻墙反正是排后面的理由，随着技术的推陈出新，iOS系统及API、app不断丰富，人们软件版权意识也逐步增强，越狱的人越来越少，直到后来连cydia都没有了。

&emsp;&emsp;从苹果推出Network Extension接口后，第一个大流行的翻墙app应该是[Surge](https://nssurge.com/)了，作者一直声称Surge不是个翻墙软件而是个网络调试工具，后来产品方向上确实做出了一些改变，不过早期在国内靠切实的翻墙功能（比如内建支持shaodowsocks协议）实实在在割了一大波韭菜。后来又陆续出现了几个翻墙app，比如[Shadowrocket](https://apps.apple.com/us/app/shadowrocket/id932747118)，[Quantumult](https://apps.apple.com/us/app/quantumult/id1252015438)以及[Quantumult  X](https://apps.apple.com/us/app/quantumult-x/id1443988620)等等。注意，由于国内政策的限制，这些app全都没有上架中国区。

&emsp;&emsp;值得一提的是，iOS上的翻墙app无一例外是付费的，而Android上的都是免费的，从中可以看到两个系统生态的情况。

# 统一的移动端免费解决方案

