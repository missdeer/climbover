---
layout: post
title: 魔改CoreDNS防污染
date: 2022-04-28 10:11:12 +0800
category: dns
---
# 什么是DNS污染

&emsp;&emsp;众所周知，人们访问网站大多数时候是用域名的，比如在浏览器地址里输入`www.google.com`，这时浏览器会向DNS服务器请求将域名`www.google.com`解析成类似`142.251.43.4`这样的IP，通过IP浏览器才能直接连接到远程服务器，访问网页等资源。

&emsp;&emsp;DNS污染的主要表现在于让用户得不到正确的IP地址，比如为`www.google.com`请求解析时，要么返回一个错的IP，要么返回一个空的响应，要么什么都不返回。因为没有正确的IP，浏览器就无法连接到正确的远程服务器，无法访问网页等资源。

# 解决DNS污染的方案

&emsp;&emsp;从技术层面看，DNS污染主要有以下几种方式：

* 丢包
* 抢答
* 阻断

&emsp;&emsp;目前看来这些方式无规律出现。纯净DNS解析基本上就是为了解决这几个问题，现在主流的方法大体有以下几种：

* 换DNS服务器。一般ISP会自动分配一个或两个DNS服务器，这种服务器对于国内或ISP内网的CDN特别友好，但对国外的一些CDN就抓瞎了，所以有了114，阿里，DNSPOD之类的公共DNS服务器，针对用户常用的Apple等CDN做了优化，但它们不能解决DNS污染的问题，所以需要用8888之类国外的DNS来解决污染。
* 换端口。一些国外的知名DNS服务器树大招风了，标准的DNS使用53端口仍然会被污染，于是像OpenDNS之类一些公共DNS服务除了53端口外还在诸如443，5353等端口提供DNS解析服务，有些地区的ISP还没针对非标准端口进行污染，但操作系统对DNS解析的操作仅支持标准的53端口，所以需要额外的软件把请求端口从53转发过来。
* 换协议。操作系统在进行DNS解析时使用的是UDP协议，但现在有一些公共DNS服务器，比如8888，OpenDNS等提供了TCP之类的传输协议进行解析，除此之外还有DNS-over-TLS/DNS-over-HTTP/DNS-over-QUIC等等，一些公共DNS服务器已经支持其中的一个或多个。还有DNSCrypt，是OpenDNS在TCP之上实现的一种加密传输协议。跟换端口一样，为了让操作系统能够使用，所以也需要额外的软件实现协议。
* 代理。DNS解析本身也是一次数据传输，DNS污染就是对数据传输进行了阻碍和修改，所以代理就是让DNS协议走在一条相对可靠的传输线路上，除了可以直接对DNS协议本身即UDP代理，还可以通过DNS over VPN之类的方式代理，比如shadowsocks-libev版有一个ss-tunnel工具，可以把远端主机的端口映射到本地的端口，如果把8888的53端口映射到本地53端口，那么相当于开通了一条走shadowsocks协议的DNS代理。

&emsp;&emsp;有了纯净DNS，基本上浏览被墙网站是没问题了。不过功能实现后，就该考虑优化了，因为上面提到的方法最后都是让国外的DNS服务器进行最终的解析，所以对国内的CDN基本无解。现在常用的方法基本上是分流解析，国内域名让国内DNS服务器解析，国外域名让国外DNS服务器解析，细微的差别就在于如何判断出一个域名需要让哪边的DNS服务器解析。

# 百花齐放的纯净DNS代理程序

* Shadowsocks作者曾经写过一个叫ChinaDNS的程序，老早停更了，不过提供的思路仍然值得参考。最早的版本是维护了一个IP黑名单，这是基于这么一个假设，即GFW抢答给出的IP属于一个并不大的集合，于是只要收到了黑名单中的IP时，就走纯净DNS解析。后来的版本改成同时向国内外的DNS服务器请求，如果得的IP是国外的，则取纯净DNS解析结果，否则取国内DNS解析结果。这是基于这么一个假设，国外的DNS解析结果总是纯净的，并且总是比国内的DNS解析结果晚到。
* 曾经有个网名叫28小朋友的牛人写过一个叫PCap_DNSProxy的程序，用C++开发，依赖libpcap/Winpcap，从驱动层抓捕到DNS解析返回的UDP包，把GFW抢答的包丢弃，以此来获得纯净DNS解析结果。实际上如何辨别出一个DNS应答包是抢答的是件很费神的事，28小朋友对此也有点诲莫如深。除此之外它还有一系列跟DNS解析相关的功能，比如CNAME hosts之类的。这个程序存在了好几年，最后以28小朋友被请喝茶，删库，退网为结局。
* dnsmasq+china domain list。我觉得这是最unix-like style的方案，作者Felix Yan也是Archlinux的开发者，维护了一个包含几千个域名的国内网站域名列表，配合dnsmasq，可以让这些域名走国内的DNS服务器解析，其他域名默认走纯净DNS解析。不过dnsmasq在列表大的时候使用遍历算法，效率比较低。
* 其他轮子，如[smartdns](https://github.com/pymumu/smartdns)、[overture](https://github.com/shawn1m/overture)、[Stubby](https://github.com/getdnsapi/stubby)、[AdGuard](https://adguard.com/)等等，大多是开源的，基本原理几乎不变。

# 什么是CoreDNS，以及为什么是CoreDNS

&emsp;&emsp;[CoreDNS](https://github.com/coredns/coredns)的来头不小，它的[作者](https://github.com/miekg)也是最好的开源[DNS package](https://github.com/miekg/dns)的作者，CoreDNS底层也使用了这个package。大名鼎鼎的k8s使用CoreDNS进行服务发现。CoreDNS完全符合我的需求：

* 无污染
* 国内CDN友好
* 跨平台，支持多种CPU、操作系统
* 可扩展性好
* 社区活跃，更新维护积极

# 实现步骤

&emsp;&emsp;CoreDNS基本沿用了[Caddy](https://caddyserver.com/) v1 的插件架构，所以CoreDNS的配置文件的语法跟Caddy v1 的配置文件语法相同。一个最简单的配置文件可以是这样：

```
.:53{
    forward . 8.8.8.8
    log
    health
}
```

&emsp;&emsp;将配置保存为文件`Corefile`，运行命令`sudo coredns -conf Corefile`，如果是在Linux系统下，可以先执行一次`sudo setcap CAP_NET_BIND_SERVICE+ep ./coredns`，以后不需要`sudo`直接运行`coredns`就可以了。如此即可在本地同时监听TCP和UDP 53端口，将所有UDP查询请求转发到`8.8.8.8`再返回，可以通过`dig @::1 -p 53 twitter.com`进行测试。

&emsp;&emsp;但是这个配置文件在国内几乎是没啥用的，原因自然是`8.8.8.8`乃老大哥重点关注对象，直接访问得到的结果都是二手信息。一个好一点的方案是使用非标准端口，比如：

```
.:53{
    forward . 208.67.222.222:443
    log
    health
}
```

&emsp;&emsp;`forward`插件支持多个上游服务器以实现简单的负载均衡：

```
.:53{
    forward . 208.67.222.222:443 208.67.222.222:5353 208.67.220.220:443 208.67.220.220:5353
    log
    health
}
```

&emsp;&emsp;大陆的网络环境非常复杂，UDP非标准端口也只在某些地区某些运营商有用，现在比较好的一个选择是DoT，即DNS over TLS，知名的支持DoT的公共DNS服务有Quad9的`9.9.9.9`，Google的`8.8.8.8`以及Cloudflare的`1.1.1.1`，可以这么使用：

```
.:53{
    forward . 127.0.0.1:5301 127.0.0.1:5302 127.0.0.1:5303
    log
    health
}
.:5301 {
   forward . tls://9.9.9.9 {
       tls_servername dns.quad9.net
   }
   cache
}
.:5302 {
    forward . tls://1.1.1.1 tls://1.0.0.1 {
        tls_servername 1dot1dot1dot1.cloudflare-dns.com
    }
    cache
}
.:5303 {
    forward . tls://8.8.8.8 tls://8.8.4.4 {
        tls_servername dns.google
    }
    cache
}
```

&emsp;&emsp;这样除了老大哥把连接reset，基本可以得到正确的DNS解析结果。

&emsp;&emsp;另一个问题是国内CDN友好，我一直以来的做法是使用[FelixOnMars的大陆区域名列表](https://github.com/felixonmars/dnsmasq-china-list)过滤。这个列表是给dnsmasq用的，经过转换可以给CoreDNS用，这利用了CoreDNS的两个插件来实现，分别是`forward`和`proxy`，这两个插件的功能非常相似，都是将DNS解析请求发给上游DNS server，再将结果取回返回给客户端。为了实现分流解析，可以将所有请求都通过`forward`转发到无污染上游解析，将大陆区域名列表加到异常列表，再把剩下的所有请求（其实就是异常列表中的域名）通过`proxy`转发到国内（最好是当前ISP的）DNS server，比如：

```
.:53{
    forward . 127.0.0.1:5301 127.0.0.1:5302 127.0.0.1:5303 {
        except www.taobao.com
    }
    proxy . 116.228.111.118 180.168.255.18
    log
    health
}
.:5301 {
   forward . tls://9.9.9.9 {
       tls_servername dns.quad9.net
   }
   cache
}
.:5302 {
    forward . tls://1.1.1.1 tls://1.0.0.1 {
        tls_servername 1dot1dot1dot1.cloudflare-dns.com
    }
    cache
}
.:5303 {
    forward . tls://8.8.8.8 tls://8.8.4.4 {
        tls_servername dns.google
    }
    cache
}
```

&emsp;&emsp;这里`except www.taobao.com`表示`www.taobao.com`这个域名不要通过`forward`解析，后面可以跟多个域名，于是这些域名会掉到下面的`proxy`插件进行解析，而`116.228.111.118`和`180.168.255.18`则是我的ISP提供的DNS服务器，可以得到最好的CDN友好的结果。

&emsp;&emsp;这时就可以用上[FelixOnMars的大陆区域名列表](https://github.com/felixonmars/dnsmasq-china-list)了，用以下命令可以得到所有域名连接而成的长字符串，放在`except`标识符后面:

```
china=`curl https://cdn.jsdelivr.net/gh/felixonmars/dnsmasq-china-list/accelerated-domains.china.conf -s | while read line; do awk -F '/' '{print $2}' | grep -v '#' ; done |  paste -sd " " -`
echo "  except $china " >> Corefile
```

&emsp;&emsp;FelixOnMars同时还提供了Google和Apple的域名列表，这在某些地区某些ISP可以得到国内镜像的IP，所以最后可以写一个这样的shell脚本用于生成Corefile：

{% gist 5c7c82b5b67f8afb41cfd43d51b82c2d %}

&emsp;&emsp;我把这个脚本放在gist上，于是可以这样生成Corefile：

```bash
curl -sSL git.io/corefile | bash
```

&emsp;&emsp;到此为止，就已经得到国内CDN友好的无污染DNS解析服务了。

&emsp;&emsp;我还想得到更多，比如去广告！github上有非常多的列表，包括广告和有害软件等等，CoreDNS官方尚未提供一个block插件，好在已经有一些非官方的实现，比如[block](https://github.com/missdeer/block/)，可以用如下的方式使用：

```
.:53{
    block https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
    block https://mirror1.malwaredomains.com/files/justdomains
    block https://s3.amazonaws.com/lists.disconnect.me/simple_ad.txt
    forward . 127.0.0.1:5301 127.0.0.1:5302 127.0.0.1:5303 {
        except www.taobao.com
    }
    proxy . 116.228.111.118 180.168.255.18
    log
    health
}
```

&emsp;&emsp;非常浅显易懂。如果遇到的请求域名是在列表中，则会返回`NXDOMAIN`。

&emsp;&emsp;最后一个问题，由于`proxy`插件和`block`插件都不是官方内置插件，从CoreDNS官方下载页下载的可执行程序并不包括这两个插件，所以需要自己编译CoreDNS。

&emsp;&emsp;编译CoreDNS并不复杂：

```
git clone https://github.com/coredns/coredns.git
cd coredns
make
```

&emsp;&emsp;CoreDNS使用了go modules机制，所以在`make`过程中会自动下载依赖的package。其中一些package是放在诸如`golang.org/x/`的路径下的，所以需要自备梯子，可以全局翻，也可以通过`HTTP_PROXY`环境变量指定，或者使用国内的一些镜像（如果你信得过的话）通过`GOPROXY`环境变量指定。

&emsp;&emsp;如果要加入以上两个插件，则在`make`前，要修改`plugin.cfg`文件，加入以下两行：

```
block:github.com/missdeer/block
proxy:github.com/coredns/proxy
```

&emsp;&emsp;再`make`，就会把这两个插件编译进去。如果发现没有编译进去，可以先执行一下`go generate coredns.go`再`make`。

&emsp;&emsp;如果要给其他平台交叉编译CoreDNS，需要先以当前平台为参数`make`一次，再以目标平台为参数进行`make`，因为第一次`make`时会调用`go generate`跑两个程序，如果不是当前平台的可执行文件是跑不起来的。

&emsp;&emsp;最后，我把这个编译过程放到[github](https://github.com/missdeer/coredns_custom_build)上了，用`appveyor`服务编译出[各个目标平台的CoreDNS](https://ci.appveyor.com/project/missdeer/coredns-custom-build)。所有最近编译好的可执行文件我都做了个可读性更好的跳转链接，列在下方表格中（注意，精简版缺少geoip/nsid/debug/trace/autopath/erratic/metadata/cancel/pprof/dnstap/dns64/acl/chaos/dnssec/secondary/loop/grpc/sign等插件，这些插件对科学上网并无帮助，可以节省一点系统资源消耗）：

| 系统         | 架构           | 精简  | 选项       | 链接                                                                        |
|--------------|----------------|-------|-----------|-----------------------------------------------------------------------------|
| Windows      | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-windows-amd64.zip](https://coredns.minidump.info/dl/coredns-windows-amd64.zip)                  |
| Windows      | x86            |       |           | [https://coredns.minidump.info/dl/coredns-windows-386.zip](https://coredns.minidump.info/dl/coredns-windows-386.zip)                    |
| Windows      | arm            |       |           | [https://coredns.minidump.info/dl/coredns-windows-arm.zip](https://coredns.minidump.info/dl/coredns-windows-arm.zip)                    |
| Windows      | arm64          |       |           | [https://coredns.minidump.info/dl/coredns-windows-arm64.zip](https://coredns.minidump.info/dl/coredns-windows-arm64.zip)                  |
| Windows      | arm            |  ✔   |           | [https://coredns.minidump.info/dl/coredns-windows-arm-lite.zip](https://coredns.minidump.info/dl/coredns-windows-arm-lite.zip)               |
| Windows      | arm64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-windows-arm64-lite.zip](https://coredns.minidump.info/dl/coredns-windows-arm64-lite.zip)             |
| macOS        | x86_64 & arm64 |       |           | [https://coredns.minidump.info/dl/coredns-darwin-universal.zip](https://coredns.minidump.info/dl/coredns-darwin-universal.zip)               |
| macOS        | x86_64 & arm64 |  ✔   |           | [https://coredns.minidump.info/dl/coredns-darwin-universal-lite.zip](https://coredns.minidump.info/dl/coredns-darwin-universal-lite.zip)          |
| Linux        | x86            |       |           | [https://coredns.minidump.info/dl/coredns-linux-386.zip](https://coredns.minidump.info/dl/coredns-linux-386.zip)                      |
| Linux        | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-linux-amd64.zip](https://coredns.minidump.info/dl/coredns-linux-amd64.zip)                    |
| Linux        | arm            |       | 5         | [https://coredns.minidump.info/dl/coredns-linux-armv5.zip](https://coredns.minidump.info/dl/coredns-linux-armv5.zip)                    |
| Linux        | arm            |       | 6         | [https://coredns.minidump.info/dl/coredns-linux-armv6.zip](https://coredns.minidump.info/dl/coredns-linux-armv6.zip)                    |
| Linux        | arm            |       | 7         | [https://coredns.minidump.info/dl/coredns-linux-armv7.zip](https://coredns.minidump.info/dl/coredns-linux-armv7.zip)                    |
| Linux        | arm64          |       |           | [https://coredns.minidump.info/dl/coredns-linux-arm64.zip](https://coredns.minidump.info/dl/coredns-linux-arm64.zip)                    |
| Linux        | ppc64          |       |           | [https://coredns.minidump.info/dl/coredns-linux-ppc64.zip](https://coredns.minidump.info/dl/coredns-linux-ppc64.zip)                    |
| Linux        | ppc64le        |       |           | [https://coredns.minidump.info/dl/coredns-linux-ppc64le.zip](https://coredns.minidump.info/dl/coredns-linux-ppc64le.zip)                  |
| Linux        | riscv64        |       |           | [https://coredns.minidump.info/dl/coredns-linux-riscv64.zip](https://coredns.minidump.info/dl/coredns-linux-riscv64.zip)                  |
| Linux        | mips64         |       | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64-hardfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mips64-hardfloat.zip)         |
| Linux        | mips64         |       | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64-softfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mips64-softfloat.zip)         |
| Linux        | mips64le       |       | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64le-hardfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mips64le-hardfloat.zip)       |
| Linux        | mips64le       |       | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64le-softfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mips64le-softfloat.zip)       |
| Linux        | mips           |       | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mips-hardfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mips-hardfloat.zip)           |
| Linux        | mips           |       | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mips-softfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mips-softfloat.zip)           |
| Linux        | mipsle         |       | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mipsle-hardfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mipsle-hardfloat.zip)         |
| Linux        | mipsle         |       | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mipsle-softfloat.zip](https://coredns.minidump.info/dl/coredns-linux-mipsle-softfloat.zip)         |
| Linux        | arm            |  ✔   | 5         | [https://coredns.minidump.info/dl/coredns-linux-armv5-lite.zip](https://coredns.minidump.info/dl/coredns-linux-armv5-lite.zip)               |
| Linux        | arm            |  ✔   | 6         | [https://coredns.minidump.info/dl/coredns-linux-armv6-lite.zip](https://coredns.minidump.info/dl/coredns-linux-armv6-lite.zip)               |
| Linux        | arm            |  ✔   | 7         | [https://coredns.minidump.info/dl/coredns-linux-armv7-lite.zip](https://coredns.minidump.info/dl/coredns-linux-armv7-lite.zip)               |
| Linux        | arm64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-linux-arm64-lite.zip](https://coredns.minidump.info/dl/coredns-linux-arm64-lite.zip)               |
| Linux        | ppc64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-linux-ppc64-lite.zip](https://coredns.minidump.info/dl/coredns-linux-ppc64-lite.zip)               |
| Linux        | ppc64le        |  ✔   |           | [https://coredns.minidump.info/dl/coredns-linux-ppc64le-lite.zip](https://coredns.minidump.info/dl/coredns-linux-ppc64le-lite.zip)             |
| Linux        | mips64         |  ✔   | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64-hardfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mips64-hardfloat-lite.zip)    |
| Linux        | mips64         |  ✔   | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64-softfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mips64-softfloat-lite.zip)    |
| Linux        | mips64le       |  ✔   | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64le-hardfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mips64le-hardfloat-lite.zip)  |
| Linux        | mips64le       |  ✔   | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mips64le-softfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mips64le-softfloat-lite.zip)  |
| Linux        | mips           |  ✔   | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mips-hardfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mips-hardfloat-lite.zip)      |
| Linux        | mips           |  ✔   | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mips-softfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mips-softfloat-lite.zip)      |
| Linux        | mipsle         |  ✔   | hardfloat | [https://coredns.minidump.info/dl/coredns-linux-mipsle-hardfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mipsle-hardfloat-lite.zip)    |
| Linux        | mipsle         |  ✔   | softfloat | [https://coredns.minidump.info/dl/coredns-linux-mipsle-softfloat-lite.zip](https://coredns.minidump.info/dl/coredns-linux-mipsle-softfloat-lite.zip)    |
| Linux        | riscv64        |  ✔   |           | [https://coredns.minidump.info/dl/coredns-linux-riscv64-lite.zip](https://coredns.minidump.info/dl/coredns-linux-riscv64-lite.zip)             |
| Linux        | s390x          |       |           | [https://coredns.minidump.info/dl/coredns-linux-s390x.zip](https://coredns.minidump.info/dl/coredns-linux-s390x.zip)                    |
| FreeBSD      | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-freebsd-amd64.zip](https://coredns.minidump.info/dl/coredns-freebsd-amd64.zip)                  |
| FreeBSD      | x86            |       |           | [https://coredns.minidump.info/dl/coredns-freebsd-386.zip](https://coredns.minidump.info/dl/coredns-freebsd-386.zip)                    |
| FreeBSD      | arm            |       |           | [https://coredns.minidump.info/dl/coredns-freebsd-arm.zip](https://coredns.minidump.info/dl/coredns-freebsd-arm.zip)                    |
| FreeBSD      | arm64          |       |           | [https://coredns.minidump.info/dl/coredns-freebsd-arm64.zip](https://coredns.minidump.info/dl/coredns-freebsd-arm64.zip)                  |
| FreeBSD      | arm            |  ✔   |           | [https://coredns.minidump.info/dl/coredns-freebsd-arm-lite.zip](https://coredns.minidump.info/dl/coredns-freebsd-arm-lite.zip)                |
| FreeBSD      | arm64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-freebsd-arm64-lite.zip](https://coredns.minidump.info/dl/coredns-freebsd-arm64-lite.zip)              |
| NetBSD       | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-netbsd-amd64.zip](https://coredns.minidump.info/dl/coredns-netbsd-amd64.zip)                   |
| NetBSD       | x86            |       |           | [https://coredns.minidump.info/dl/coredns-netbsd-386.zip](https://coredns.minidump.info/dl/coredns-netbsd-386.zip)                     |
| NetBSD       | arm            |       |           | [https://coredns.minidump.info/dl/coredns-netbsd-arm.zip](https://coredns.minidump.info/dl/coredns-netbsd-arm.zip)                     |
| NetBSD       | arm64          |       |           | [https://coredns.minidump.info/dl/coredns-netbsd-arm64.zip](https://coredns.minidump.info/dl/coredns-netbsd-arm64.zip)                   |
| NetBSD       | arm            |  ✔   |           | [https://coredns.minidump.info/dl/coredns-netbsd-arm-lite.zip](https://coredns.minidump.info/dl/coredns-netbsd-arm-lite.zip)                |
| NetBSD       | arm64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-netbsd-arm64-lite.zip](https://coredns.minidump.info/dl/coredns-netbsd-arm64-lite.zip)              |
| OpenBSD      | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-openbsd-amd64.zip](https://coredns.minidump.info/dl/coredns-openbsd-amd64.zip)                  |
| OpenBSD      | x86            |       |           | [https://coredns.minidump.info/dl/coredns-openbsd-386.zip](https://coredns.minidump.info/dl/coredns-openbsd-386.zip)                    |
| OpenBSD      | arm            |       |           | [https://coredns.minidump.info/dl/coredns-openbsd-arm.zip](https://coredns.minidump.info/dl/coredns-openbsd-arm.zip)                    |
| OpenBSD      | arm64          |       |           | [https://coredns.minidump.info/dl/coredns-openbsd-arm64.zip](https://coredns.minidump.info/dl/coredns-openbsd-arm64.zip)                  |
| OpenBSD      | arm            |  ✔   |           | [https://coredns.minidump.info/dl/coredns-openbsd-arm-lite.zip](https://coredns.minidump.info/dl/coredns-openbsd-arm-lite.zip)               |
| OpenBSD      | arm64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-openbsd-arm64-lite.zip](https://coredns.minidump.info/dl/coredns-openbsd-arm64-lite.zip)             |
| DragonflyBSD | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-dragonfly-amd64.zip](https://coredns.minidump.info/dl/coredns-dragonfly-amd64.zip)                |
| Solaris      | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-solaris-amd64.zip](https://coredns.minidump.info/dl/coredns-solaris-amd64.zip)                  |
| illumos      | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-illumos-amd64.zip](https://coredns.minidump.info/dl/coredns-illumos-amd64.zip)                  |
| Android      | x86_64         |       |           | [https://coredns.minidump.info/dl/coredns-android-amd64.zip](https://coredns.minidump.info/dl/coredns-android-amd64.zip)                  |
| Android      | x86            |       |           | [https://coredns.minidump.info/dl/coredns-android-386.zip](https://coredns.minidump.info/dl/coredns-android-386.zip)                    |
| Android      | arm            |       |           | [https://coredns.minidump.info/dl/coredns-android-arm.zip](https://coredns.minidump.info/dl/coredns-android-arm.zip)                    |
| Android      | arm64          |       |           | [https://coredns.minidump.info/dl/coredns-android-aarch64.zip](https://coredns.minidump.info/dl/coredns-android-aarch64.zip)                |
| Android      | x86_64         |  ✔   |           | [https://coredns.minidump.info/dl/coredns-android-amd64-lite.zip](https://coredns.minidump.info/dl/coredns-android-amd64-lite.zip)             |
| Android      | x86            |  ✔   |           | [https://coredns.minidump.info/dl/coredns-android-386-lite.zip](https://coredns.minidump.info/dl/coredns-android-386-lite.zip)               |
| Android      | arm            |  ✔   |           | [https://coredns.minidump.info/dl/coredns-android-arm-lite.zip](https://coredns.minidump.info/dl/coredns-android-arm-lite.zip)               |
| Android      | arm64          |  ✔   |           | [https://coredns.minidump.info/dl/coredns-android-aarch64-lite.zip](https://coredns.minidump.info/dl/coredns-android-aarch64-lite.zip)           |
