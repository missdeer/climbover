---
layout: post
title: 基于Geo的流量分流
date: 2022-04-29 11:12:13 +0800
category: traffic
---
# 为什么要流量分流

&emsp;&emsp;在x墙技术出现的早期（十五六年前？），基本上没有流量分流的做法，代理程序开始工作后，绝大多数场合都是把所有流量都通过代理程序走的，这就会导致绕远路的情况出现，比如访问国内的淘宝，先去美国绕了一圈再回来，速度慢不说，可能还会被淘宝识别为海外用户，直接跳转到海外版页面了。所以最基本的要求是国内网站就不要代理了，直连就行，海外站点才走代理线路。

&emsp;&emsp;最早流行的流量分流技术大概是用浏览器扩展实现的，有一个著名的[gfwlist列表](https://github.com/gfwlist/gfwlist)，记录了绝大多数被墙的网址，浏览器识别到后自动调用代理或直连。如果不用浏览器扩展，pac格式的代理文件也能通过域名分流。这种方法的前提是，代理程序提供的是一个http或socks代理端口。

&emsp;&emsp;如果代理程序是以VPN形式提供全局代理，常用的流量分流方法是修改路由表。网上可以找到中国大陆被分配使用的IP地址范围表，比如[这个](https://github.com/17mon/china_ip_list)，[这个](http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest)，或[这个](https://www.maxmind.com/en/geoip2-precision-country-service)。在连接上VPN后，再将所有这些IP段在路由表中修改为直连原网关，就能达到分流的效果。

&emsp;&emsp;一些代理程序比如[v2ray](https://github.com/v2fly/v2ray-core)，[clash](https://github.com/Dreamacro/clash)，[影梭](https://github.com/shadowsocks/shadowsocks-android)等，自带了分流的功能，基本原理也是根据gfwlist或IP地址表识别是要代理还是直连。

# 我为什么要流量分流

&emsp;&emsp;从7元/月开始入了喵帕斯的坑（早已跑路，并亏了刚充值的700多元钱），才知道很多机场是同时提供多个地区的网络出口的，于是想着怎么把这种资源利用起来。其实只要线路足够好，一条就基本够了，但实际上往往并没这么好，于是我就想照不同的目的IP使用不同的线路的方式来分流，比如目标IP是美国的，就用美国出口的线路，目标IP是澳洲的，就用澳洲出口的线路。事实证明这样的做法还是有点用的，例如我在新加坡买了一个VPS，如果用[Boom](https://www.boomssv.com/aff.php?aff=2340)在香港的线路测速结果就很差，如果换成新加坡的线路测速就跑上去了。最重要的是这个几乎不花费额外的成本，只需要一点设置即可。

# 实现步骤

&emsp;&emsp;在[基于通用Linux发行版的科学上网网关](/gateway/2022/04/27/common-linux-distribution-based-gateway)一文中已经说过，我是用iptables来做分流的，其中有一段中国大陆区IP的条目在那篇文章中并没有详细列出来，这次正好一起写了。

&emsp;&emsp;首先要注意一点，iptables的匹配过程是从上到下依次匹配，如果条目很多的话，比如几万几十万（光是中国大陆的IP段就有6000多条），那会拖慢整个系统的性能，所以我引入了[ipset](https://ipset.netfilter.org/)，它使用了hash表来存储和查询，效率非常高，按地区把所有IP段都存入一个ipset中，iptables只要匹配几个ipset就可以了。

&emsp;&emsp;再看我现在使用的[Boom](https://www.boomssv.com/aff.php?aff=2340)套餐，有香港、日本、台湾、韩国、新加坡、美国、意大利、俄罗斯、英国、法国、德国、西班牙、荷兰、瑞典、澳洲这些出口，所以把全世界已分配的IP段按这些地区拆分出来，并从欧洲、亚洲、美洲IP段里把上述这些地区的IP段剔除出去。

&emsp;&emsp;我是从[MaxMind](https://www.maxmind.com/en/geoip2-precision-country-service)下载到IP地区Geoip数据库的，下载需要license，可以免费在线申请，如果不想自己申请的话，可以在GitHub上搜一个，有些人不注意就把他申请的license一起提交了。用以下命令下载，记得把`{license}`替换成有效的license：

```bash
curl -L "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country-CSV&license_key={license}&suffix=zip" -o GeoLite2-Country-CSV.zip
```

&emsp;&emsp;得到按国家地区分布的IP数据库后，解压，再写几行shell脚本，生成前面说的ipset配置文件：

```bash
#!/bin/bash
unzip GeoLite2-Country-CSV.zip
find . -name 'GeoLite2-Country-CSV_*' -type d | while read dir; 
do 
    mv $dir "GeoLite2-Country-CSV"; 
    break;
done
cd GeoLite2-Country-CSV
curl -o cnroute.txt -L https://fastly.jsdelivr.net/gh/17mon/china_ip_list@master/china_ip_list.txt
n=`grep -r -a ',IT,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > itroute.txt
n=`grep -r -a ',US,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > usroute.txt
n=`grep -r -a ',JP,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > jproute.txt
n=`grep -r -a ',TW,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > twroute.txt
n=`grep -r -a ',HK,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > hkroute.txt
n=`grep -r -a ',SG,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > sgroute.txt
n=`grep -r -a ',KR,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > krroute.txt
n=`grep -r -a ',RU,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > ruroute.txt
n=`grep -r -a ',GB,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > gbroute.txt
n=`grep -r -a ',FR,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > frroute.txt
n=`grep -r -a ',DE,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > deroute.txt
n=`grep -r -a ',SE,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > seroute.txt
n=`grep -r -a ',ES,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > esroute.txt
n=`grep -r -a ',NL,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > nlroute.txt
n=`grep -r -a ',AS,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > asroute.txt
cat cnroute.txt | while read ip; do sed -i "\|$ip|d" asroute.txt; done
cat jproute.txt | while read ip; do sed -i "\|$ip|d" asroute.txt; done
cat twroute.txt | while read ip; do sed -i "\|$ip|d" asroute.txt; done
cat hkroute.txt | while read ip; do sed -i "\|$ip|d" asroute.txt; done
cat krroute.txt | while read ip; do sed -i "\|$ip|d" asroute.txt; done
cat sgroute.txt | while read ip; do sed -i "\|$ip|d" asroute.txt; done
n=`grep -r -a ',EU,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > euroute.txt
cat itroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat ruroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat gbroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat frroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat nlroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat seroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat esroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
cat deroute.txt | while read ip; do sed -i "\|$ip|d" euroute.txt; done
n=`grep -r -a ',OC,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > ocroute.txt
n=`grep -r -a ',AF,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > afroute.txt
n=`grep -r -a ',SA,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > saroute.txt
n=`grep -r -a ',NA,' GeoLite2-Country-Locations-zh-CN.csv | awk -F ',' '{print $1}'`
cat GeoLite2-Country-Blocks-IPv4.csv | awk -F ',' '{ $3=null;print $0}' | grep "$n" | awk -F ' ' '{print $1}' > naroute.txt
cat usroute.txt | while read ip; do sed -i "\|$ip|d" naroute.txt; done
mkdir ../routes
mv *route.txt ../routes/
cd ../routes
find . -name '*route.txt' -type f | while read f; 
do 
    basename=${f##*/}; 
    basename=${basename%.txt}; 
    sed -i "s|^|add $basename |" f; 
done
```

&emsp;&emsp;这个脚本的算法没有优化过，所以运行需要很长时间，我把它放到GitHub Actions上跑也要一个多小时，好在这个脚本并不需要经常跑，大概一个月跑一次也就差不多了，而且放在GitHub Actions上并不占用自己的机器时间。如果自己跑脚本嫌麻烦，可以直接从[GitHub](https://github.com/missdeer/daily-weekly-build/tree/routes)下载我定期跑的结果。

&emsp;&emsp;脚本跑完后会生成21个形如`*route.txt`的文件，`*`是诸如`us`、`cn`、`as`等的地区代码。文件内容大体是这个样子：

```
add cnroute 1.0.1.0/24
add cnroute 1.0.2.0/23
add cnroute 1.0.8.0/21
……
```

&emsp;&emsp;文件名和地区的对应关系如下表所示：

| 文件名      | 地区    |
|-------------|--------|
| cnroute.txt | 中国大陆 |
| afroute.txt | 非洲    |
| asroute.txt | 亚洲    |
| deroute.txt | 德国    |
| esroute.txt | 西班牙  |
| euroute.txt | 欧洲    |
| frroute.txt | 法国    |
| gbroute.txt | 英国    |
| hkroute.txt | 香港    |
| itroute.txt | 意大利  |
| jproute.txt | 日本    |
| krroute.txt | 韩国    |
| naroute.txt | 北美洲  |
| nlroute.txt | 荷兰    |
| ocroute.txt | 澳洲    |
| ruroute.txt | 俄罗斯  |
| saroute.txt | 南美洲  |
| seroute.txt | 瑞典    |
| sgroute.txt | 新加坡  |
| twroute.txt | 台湾    |
| usroute.txt | 美国    |

&emsp;&emsp;有了这些文件，再写一个shell脚本，把ipset创建起来：

```
#!/bin/bash
if [[ $EUID -ne 0 ]]; then
   echo "This script must be run as root"
   exit 1
fi

ipset list | grep route | while read line;
do
   name=`echo $line | awk -F ' ' '{print $2}'`;
   echo "flush $name"
   ipset flush $name;
done

find . -name '*route.txt' -type f | while read f
do
  basename=${f##*/}
  basename=${basename%.txt}
  elemcount=`cat $f | wc -l`
  has_route=`ipset list | grep -o $basename`
  if [ "$has_route"x = "$basename"x ]
  then
    echo "$basename exists"
  else
    echo "create $basename with max element count $elemcount"
    ipset create $basename hash:net family inet hashsize 1024 maxelem $elemcount
  fi
  echo "restore $basename"
  ipset restore -f $f
done
```

&emsp;&emsp;这个脚本一定要用`root`账号跑，所以一开始检测到不是root就打印一个提示信息然后退出。之后便是先列出所有当前系统中名如`*route`的ipset，并清空其内容。后面再按文件名按需创建新的ipset，并导入文件内容。之所以不每次直接删掉ipset再统一创建后导入内容，是因为iptables如果正在使用ipset，是删不掉ipset的，只能清空内容，除非先把iptables停掉。

&emsp;&emsp;跑完这个脚本，可以用命令看一下当前系统中已有的ipset：

```bash
sudo ipset list | grep route | sort -n
```

&emsp;&emsp;一切正常的话应该能列出21个ipset的名字，只会多，不会少：

```
Name: afroute
Name: asroute
Name: cnroute
Name: deroute
Name: esroute
Name: euroute
Name: frroute
Name: gbroute
Name: hkroute
Name: itroute
Name: jproute
Name: krroute
Name: naroute
Name: nlroute
Name: ocroute
Name: ruroute
Name: saroute
Name: seroute
Name: sgroute
Name: twroute
Name: usroute
```

&emsp;&emsp;然后运行多个`ss-redir`进程，将不同路线监听在不同的端口：

```bash
ss-redir -s 123.123.123.123 -p 8100 -k qwerasdfzxcv -m chacha20-ietf -l 58100 -b 0.0.0.0 -f hk.pid
ss-redir -s 123.123.123.123 -p 8101 -k qwerasdfzxcv -m chacha20-ietf -l 58101 -b 0.0.0.0 -f jp.pid
ss-redir -s 123.123.123.123 -p 8102 -k qwerasdfzxcv -m chacha20-ietf -l 58102 -b 0.0.0.0 -f us.pid
ss-redir -s 123.123.123.123 -p 8103 -k qwerasdfzxcv -m chacha20-ietf -l 58103 -b 0.0.0.0 -f tw.pid
ss-redir -s 123.123.123.123 -p 8104 -k qwerasdfzxcv -m chacha20-ietf -l 58104 -b 0.0.0.0 -f kr.pid
ss-redir -s 123.123.123.123 -p 8105 -k qwerasdfzxcv -m chacha20-ietf -l 58105 -b 0.0.0.0 -f sg.pid
ss-redir -s 123.123.123.123 -p 8106 -k qwerasdfzxcv -m chacha20-ietf -l 58106 -b 0.0.0.0 -f ru.pid
ss-redir -s 123.123.123.123 -p 8107 -k qwerasdfzxcv -m chacha20-ietf -l 58107 -b 0.0.0.0 -f it.pid
ss-redir -s 123.123.123.123 -p 8108 -k qwerasdfzxcv -m chacha20-ietf -l 58108 -b 0.0.0.0 -f gb.pid
ss-redir -s 123.123.123.123 -p 8109 -k qwerasdfzxcv -m chacha20-ietf -l 58109 -b 0.0.0.0 -f oc.pid
ss-redir -s 123.123.123.123 -p 8110 -k qwerasdfzxcv -m chacha20-ietf -l 58110 -b 0.0.0.0 -f de.pid
ss-redir -s 123.123.123.123 -p 8111 -k qwerasdfzxcv -m chacha20-ietf -l 58111 -b 0.0.0.0 -f nl.pid
ss-redir -s 123.123.123.123 -p 8112 -k qwerasdfzxcv -m chacha20-ietf -l 58112 -b 0.0.0.0 -f se.pid
ss-redir -s 123.123.123.123 -p 8113 -k qwerasdfzxcv -m chacha20-ietf -l 58113 -b 0.0.0.0 -f es.pid
ss-redir -s 123.123.123.123 -p 8114 -k qwerasdfzxcv -m chacha20-ietf -l 58114 -b 0.0.0.0 -f fr.pid
```

&emsp;&emsp;最后修改一下iptables设置：

```bash
iptables -t nat -A SS -m set --match-set cnroute dst -j RETURN
iptables -t nat -A SS -p tcp -m set --match-set hkroute dst -j REDIRECT --to-ports 58100
iptables -t nat -A SS -p tcp -m set --match-set asroute dst -j REDIRECT --to-ports 58100
iptables -t nat -A SS -p tcp -m set --match-set jproute dst -j REDIRECT --to-ports 58101
iptables -t nat -A SS -p tcp -m set --match-set usroute dst -j REDIRECT --to-ports 58102
iptables -t nat -A SS -p tcp -m set --match-set afroute dst -j REDIRECT --to-ports 58102
iptables -t nat -A SS -p tcp -m set --match-set saroute dst -j REDIRECT --to-ports 58102
iptables -t nat -A SS -p tcp -m set --match-set naroute dst -j REDIRECT --to-ports 58102
iptables -t nat -A SS -p tcp -m set --match-set twroute dst -j REDIRECT --to-ports 58103
iptables -t nat -A SS -p tcp -m set --match-set krroute dst -j REDIRECT --to-ports 58104
iptables -t nat -A SS -p tcp -m set --match-set sgroute dst -j REDIRECT --to-ports 58105
iptables -t nat -A SS -p tcp -m set --match-set ruroute dst -j REDIRECT --to-ports 58106
iptables -t nat -A SS -p tcp -m set --match-set itroute dst -j REDIRECT --to-ports 58107
iptables -t nat -A SS -p tcp -m set --match-set gbroute dst -j REDIRECT --to-ports 58108
iptables -t nat -A SS -p tcp -m set --match-set deroute dst -j REDIRECT --to-ports 58111
iptables -t nat -A SS -p tcp -m set --match-set nlroute dst -j REDIRECT --to-ports 58112
iptables -t nat -A SS -p tcp -m set --match-set seroute dst -j REDIRECT --to-ports 58113
iptables -t nat -A SS -p tcp -m set --match-set esroute dst -j REDIRECT --to-ports 58114
iptables -t nat -A SS -p tcp -m set --match-set frroute dst -j REDIRECT --to-ports 58115
iptables -t nat -A SS -p tcp -m set --match-set euroute dst -j REDIRECT --to-ports 58111
iptables -t nat -A SS -p tcp -m set --match-set ocroute dst -j REDIRECT --to-ports 58110
iptables -t nat -A SS -p tcp -j REDIRECT --to-ports 58102
```

&emsp;&emsp;可以看到中国大陆IP直接返回，不走代理，另外亚洲除了香港、日本、韩国、新加坡分别有代理节点后，其余的统一走香港出口。欧洲没有出口节点的地区统一走德国节点。美洲和非洲，以及其余如果还有没指定的统一走美国节点。要注意的是，鉴于iptables规则从上往下依次匹配的特点，要把亚洲规则放在所有亚洲地区规则之后，欧洲规则放在所有欧洲地区规则之后。至于其他走美国线路的规则，写不写区别不大。

&emsp;&emsp;这个需求用v2ray或clash也很容易实现，这些代理程序内置了Geoip数据库，可以在配置文件中写规则进行分流，效果几乎相同。用iptables分流只是提供了一种不依赖特定程序的另一种选择，这是非常Unix风格的做法，即一个程序（或工具）只做一件事，且把这件事做到最好，可以看到我这里脚本生成ipset配置，脚本更新ipset配置，ss-redir代理，iptables分流，看起来步骤繁多，但每一步的目标都很清晰和明确，且可以单独拿出去在其他地方重复利用。