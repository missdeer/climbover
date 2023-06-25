---
layout: post
title: macOS上的透明代理
date: 2023-06-25 09:10:11 +0800
category: gateway
---
macOS毕竟是UNIX系的，可以非常简单地实现透明代理。

网上能找到的绝大多数文章或软件在macOS上都是使用tun设备通过tun2socks之类的软件来实现全局代理的，本文不谈这些，只讲使用从正统BSD移植过来的pf防火墙实现透明代理。如果从没接触过pf，可以通过`man pf.conf`或Murus的 [macOS pf手册](https://murusfirewall.com/Documentation/OS X PF Manual.pdf) 进行学习。

使用pf实现透明代理和[Linux上使用iptables实现透明代理](/gateway/2022/04/27/common-linux-distribution-based-gateway)非常相似，比如pf和iptables都是防火墙软件，都劫持并转发流量到代理软件上，代理软件都使用特殊的方式读取流量数据（用户不用关心细节），都实现了全局代理。

# 实现步骤

## 配置pf.conf

使用root编辑`/etc/pf.conf`文件（注意做好备份）：

```txt
scrub-anchor "com.apple/*"

table <direct_cidr> persist file "/usr/local/share/climbover/direct_cidr.txt"
table <hk_cidr> persist file "/usr/local/share/climbover/cidr/hk_cidr.txt"
table <tw_cidr> persist file "/usr/local/share/climbover/cidr/tw_cidr.txt"
table <kr_cidr> persist file "/usr/local/share/climbover/cidr/kr_cidr.txt"
table <jp_cidr> persist file "/usr/local/share/climbover/cidr/jp_cidr.txt"
table <sg_cidr> persist file "/usr/local/share/climbover/cidr/sg_cidr.txt"
table <ru_cidr> persist file "/usr/local/share/climbover/cidr/ru_cidr.txt"
table <na_cidr> persist file "/usr/local/share/climbover/cidr/na_cidr.txt"
table <sa_cidr> persist file "/usr/local/share/climbover/cidr/sa_cidr.txt"
table <af_cidr> persist file "/usr/local/share/climbover/cidr/af_cidr.txt"
table <as_cidr> persist file "/usr/local/share/climbover/cidr/as_cidr.txt"

nat-anchor "com.apple/*"

rdr-anchor "com.apple/*"

rdr pass on lo0 proto tcp from any to <hk_cidr> -> 127.0.0.1 port 59080
rdr pass on lo0 proto tcp from any to <jp_cidr> -> 127.0.0.1 port 59081
rdr pass on lo0 proto tcp from any to <tw_cidr> -> 127.0.0.1 port 59083
rdr pass on lo0 proto tcp from any to <sg_cidr> -> 127.0.0.1 port 59085
rdr pass on lo0 proto tcp from any to <as_cidr> -> 127.0.0.1 port 59080
rdr pass on lo0 proto tcp from any to <ru_cidr> -> 127.0.0.1 port 59086
rdr pass on lo0 proto tcp from any to !<direct_cidr> -> 127.0.0.1 port 59082

pass out route-to (lo0 127.0.0.1) proto tcp from any to !<direct_cidr>

dummynet-anchor "com.apple/*"

anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

解释一下比较重要的几步：

1. 首先，在`/usr/local/share/climbover/cidr`目录下放入`direct_cidr.txt`，`hk_cidr.txt`等几个按地区分类的cidr文件，这些文件可以自己写脚本生成，也可以从[网上下载现成](https://github.com/missdeer/daily-weekly-build/tree/cidr)的。
2. `table <direct_cidr> persist file "/usr/local/share/climbover/direct_cidr.txt"`这一行的作用是将`direct_cidr.txt`的内容加载到系统中，以`<direct_cidr>`命名这块内容，用于标识这一大片CIDR。以下几行作用类似。
3. `rdr pass on lo0 proto tcp from any to <hk_cidr> -> 127.0.0.1 port 59080`这一行的作用是将本地任何发送到`<hk_cidr>`中标识的IP地址（顾名思意是香港区的IP）的TCP流量，都劫持并转发到本机的59080端口（这个端口是代理软件开的）。下面几行作用类似。
4. `rdr pass on lo0 proto tcp from any to !<direct_cidr> -> 127.0.0.1 port 59082`这一行与上几行的区别在于表名前多了个`!`，意思是任何目标IP不在`<direct_cidr>`内的TCP流量转发到59082端口，顾名思意，不要直连的都被转发。这里隐含另一个机制，`pf.conf`的规则是从上到下一条一条匹配，匹配到了就不再继续往下匹配，所以到`!<direct_cidr>`这一行时，必定是前面几个转发规则都不满足时，才进行匹配。
5. `pass out route-to (lo0 127.0.0.1) proto tcp from any to !<direct_cidr>`这一行的作用是将所有非直连的流量路由到本地`lo0`接口上。这是因为不像iptables redirect可以配置在`OUTPUT`链中，`pf` **rdr-to** 只对ingress流量起作用，如果想要把本机的egress流量劫持到透明代理上，需要将其路由到另一个interface，转变为后者的ingress流量，再利用 **rdr-to** 进行流量转发，所以在这里我们利用了本地lo0。

这里需要注意一点是，CIDR列表不能太大，太大会加载不成功，具体阈值我也不清楚，但是遇到过。

## 配置代理软件

很多代理软件是不能直接读取到pf转发到端口的流量数据的，这就需要[redsocks](https://github.com/darkk/redsocks)转换一下，配置文件大体如下：

```txt
base {
  log_debug = off;
  log_info = on;
  daemon = off;
  redirector = pf;
}

redsocks {
  local_ip = 127.0.0.1;
  local_port = 59080;
  ip = 127.0.0.1;
  port = 29090;
  type = socks5;
}
```

[redsocks](https://github.com/darkk/redsocks)既支持iptables又支持pf，注意`redirector = pf;`这一行。上面的配置中，它会将pf发送到59080端口的流量读取后，再转发到socks5端口29090。

不过我用的是[shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)直接就支持了，省掉了[redsocks](https://github.com/darkk/redsocks)转换这一步，可以直接配置使用：

```txt
{
  "locals": [
    {
      "local_address": "0.0.0.0",
      "local_port": 59080,
      "protocol": "redir"
    }
  ],
  "servers": [
    {
      "server": "xxxx.server.com",
      "tcp_weight": 1,
      "udp_weight": 1,
      "server_port": 8443,
      "password": "mypassword",
      "method": "cryptomethod",
      "plugin": "obfs-local",
      "plugin_opts": "plugin_opts;plugin_opts;plugin_opts;plugin_opts"
    }
  ]
}
```

这里的重点就是`"protocol": "redir"`，这样启动[shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust)后59080就能正常接收`pf`转发过来的流量了。

## 配置DNS代理

以上pf配置只代理了TCP流量，并没有处理DNS污染，因此仍需要另外配置DNS代理，这部分内容[之前的文章中有详细介绍](/dns/2022/04/28/custom-coredns)，此处不再重复。

## 启动pf服务

```bash
sysctl -w net.inet.ip.forwarding=1
pfctl -e
pfctl -F all
pfctl -f /etc/pf.conf
```

使用root用户执行以上命令。四句命令分别用于：

1. 开启流量转发
2. 开启pf（默认是关闭的）
3. 清空pf原有配置
4. 从配置文件中加载新配置

如果一切顺利，这时系统中的TCP流量应该已经被全局代理了，可以使用`curl`命令测试一下：

```bash
$ curl -I https://www.google.com --resolve 'www.google.com:443:216.58.200.36'
HTTP/2 302
location: https://www.google.com.hk/url?sa=p&hl=zh-CN&pref=hkredirect&pval=yes&q=https://www.google.com.hk/&ust=1550640983822937&usg=AOvVaw3PnKH6XFhOkLB56FH7sVHc
cache-control: private
content-type: text/html; charset=UTF-8
p3p: CP="This is not a P3P policy! See g.co/p3phelp for more info."
date: Wed, 20 Feb 2019 05:35:53 GMT
server: gws
content-length: 372
x-xss-protection: 1; mode=block
x-frame-options: SAMEORIGIN
set-cookie: 1P_JAR=2019-02-20-05; expires=Fri, 22-Mar-2019 05:35:53 GMT; path=/; domain=.google.com
set-cookie: NID=160=U44fC0UHxupm7ClkYUGknQQR8gT8JmqDIhrL3VDquqo6wFketgeSCqBEgNHea2cClfa8pyYwo1u2X44uU7vIaEd5Bxeoakgtwq0aauu5Kzv5hX0N65TNmPH7LYTaESyQAT5lVMSu_RO9JarbeukX2oNoVBL_y3q0d8sty2_u7eU; expires=Thu, 22-Aug-2019 05:35:53 GMT; path=/; domain=.google.com; HttpOnly
alt-svc: quic=":443"; ma=2592000; v="44,43,39"
```

