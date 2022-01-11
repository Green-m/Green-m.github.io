---
layout: post
title:  "通过 OpenVPN 实现流量审计"
date:   2021-01-12 15:20:01 +0800
categories: Audit 
tag: OPS 
---

* content
{:toc}




0x00 前言
------------------

为了满足日益增长的渗透测试网络审计需要，特别是从普通的全流量记录，到进一步的HTTPS流量审计，特地设计了一套网络结构，满足简单的傻瓜化远程接入实验室，实现 http/https 全流量记录的功能。

更新记录： 

2022年1月11日 更新：**另一种流量分流的方式（进程分流）** 



0x01 流量镜像和审计系统
------------------

早期的流量审计需求不高，抓下全流量就可以，直接买一个流量分光的设备，双网口，一个网口直接串联到设备中间，另一个网口接到抓流量的服务器上，不用改任何网络架构，就实现目的了。 但现在的要求越来越高，需要对HTTPS这样的加密流量也进行解密了，我们只能重新设计网络架构。

#### 一、选择系统

最开始的想法是选用市面上已经有的路由器系统，不是有不少打着名号说自己特别牛逼的全流量审计，解码等等一应俱全的网关，比如 PFsense，WFilter-NGF，RouterOS，openwrt之类的。

经过测试了好几个产品，基本不太能满足需求。 比如 PFsense： SSL/TLS 解码是通过 Squirt 实现的，仅仅能解包记录域名头，并不支持全流量抓包和明文记录，而且由于使用的是 bsd 系统，想自己装一些工具也遇到了比较多的兼容性问题，无奈只好放弃。

尝试了多种系统选择后，最终决定使用 debian + mitmproxy 从轮胎开始一步一步造一辆能跑起来的车。



#### 二、物理架构图

![/styles/images/openvpn_traffic/p1.png]({{ '/styles/images/openvpn_traffic/p1.png' | prepend: site.baseurl }})



客户端接入有两条线路：



1. 流量记录线路： Client (192.168.3.x) --> 路由器(192.168.3.1) --> VirtualBox debian (192.168.1.1) --> 4G sim 卡路由器(192.168.8.1) --> 外网
2. 非流量记录线路：  Client (10.90.45.x) --> 路由器(10.90.45.1)  --> 外网

分成两条线路，保证工作和其他娱乐下载等流量分开，也避免造成不必要的流量浪费。



#### 三、配置 VirtualBox debian 10 作为路由器

虚拟机搭建步骤省略， 需要配置物理机的网卡 `enp0s1` 和 `enp0s2`通过桥接模式映射到虚拟机即可：

物理机 `enp0s1`  --> 虚拟机`enp0s3`

物理机 `enp0s2`  --> 虚拟机`enp0s8`

1. 开启流量转发：

```
sysctl –w net.ipv4.ip_forward=1
```

2. 使用iptables 配置转发策略:

```
iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 23333
iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m tcp --dport 443 -j REDIRECT --to-ports 23333
iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m tcp --dport 8080 -j REDIRECT --to-ports 23333
iptables -t nat -A POSTROUTING ! -d 192.168.2.0/24 -o enp0s3 -j MASQUERADE

iptables -t filter -A INPUT -i lo -j ACCEPT
iptables -t filter -A INPUT -i enp0s8 -j ACCEPT
iptables -t filter -A INPUT -i enp0s3 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -t filter -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
iptables -t filter -A FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

说明一下： nat 表里的规则是表示 80 443 8080 端口的流量转发到23333端口，实现透明代理，下文会解释 23333 端口的具体作用； filter 表的东西是一些转发和输入策略。

3. iptables 规则持久化

使用 `iptables-persistent ` 这个包，将上述规则保存到硬盘 `/etc/iptables/rulesv4.save` ，使用其他方式实现开机自动加载，方式自选，我使用的是 crontab 里的 `@reboot` 。



#### 四、配置 mitmproxy 实现透明代理

安装最新版 mitmproxy，使用如下命令抓取流量：

```
/usr/bin/mitmdump -q --mode transparent --showhost -p 23333 --ssl-insecure --cert=cert.pem -s mitm_parse.py -w /data/mitmproxy_traffic/mitm_files/mitm_raw_$(date +"%Y%m%d").mitm
```

默认的 `mitmdump` 抓取的流量不是标准的通用格式，是 `mitmdump`  自定义的，为了保证可读性，这里我使用了 `mitm_parse.py` 这个脚本来对抓取的内容进行处理，脚本内容如下：

```
from mitmproxy import http
import time,re
import logging

logfile = "/data/mitmproxy_traffic/logs/mitm_parsed_" + time.strftime("%Y%m%d") + ".log"
logging.basicConfig(filename=logfile,
                            filemode='a',
                            format='%(asctime)s,%(msecs)d %(name)s %(levelname)s %(message)s',
                            datefmt='%H:%M:%S',
                            level=logging.INFO)
logger = logging.getLogger('mylogger')


def response(flow):
    logtext = "\n" 
    logtext += "="*50 + "\n"
    logtext += flow.request.url + "\n"

    logtext += "-"*25 + "request headers:" + "-"*25 + "\n"
    for k, v in flow.request.headers.items():
        logtext += "%-20s: %s" % (k.upper(), v) + "\n"

    logtext += "-"*25 + "request content:" + "-"*25 + "\n"
    logtext += flow.response.content.decode('utf-8','ignore') + "\n"

    logtext += "-"*25 + "response headers:" + "-"*25 + "\n"
    for k, v in flow.response.headers.items():
        logtext += "%-20s: %s" % (k.upper(), v) + "\n"

    logtext += "-"*25 + "response content:" + "-"*25 + "\n"
    logtext += flow.response.content.decode('utf-8','ignore') + "\n"
    logger.info(logtext)

```

那么这里就有两份数据，一份数据是 `mitmdump` 抓取的原始数据，存放在 `/data/mitmproxy_traffic/mitm_files/`目录下，另一份经过脚本处理过的可读性较好的数据，存放在 `/data/mitmproxy_traffic/logs/` 目录下。

使用 curl 访问 qq.com 的日志如下：

```
18:34:12,49 mylogger INFO 
==================================================
https://125.39.52.26/
-------------------------request headers:-------------------------
:AUTHORITY          : qq.com
USER-AGENT          : curl/7.72.0
ACCEPT              : */*
-------------------------request content:-------------------------
<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.6.0</center>
</body>
</html>

-------------------------response headers:-------------------------
DATE                : Mon, 11 Jan 2021 10:34:12 GMT
CONTENT-TYPE        : text/html
SERVER              : squid/3.5.24
LOCATION            : https://www.qq.com?fromdefault
EXPIRES             : Mon, 11 Jan 2021 10:35:11 GMT
CACHE-CONTROL       : max-age=60
VARY                : Accept-Encoding
X-CACHE             : HIT from shenzhen.qq.com
-------------------------response content:-------------------------
<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.6.0</center>
</body>
</html>

```



#### 五、dumpcap 抓取全流量

可以看到我们的 `mitmproxy` 的方案并不完美，只抓取了部分端口号的内容，并不能保证所有流量的记录，因此也十分有必要抓取全部的流量作为备份。

使用 `dumpcap` 抓取全流量：

```
#!/bin/bash
dir=/data/net_traffic_storage/files/$(date +"%Y%m%d")/

if [ ! -d $dir ]
then
        mkdir $dir
fi

dumpcap -i enp0s8 -w $dir/traffic.pcapng -b filesize:1024000
```



使用 `supervisor` 或者其他类似的工具将上面的命令进行持久化（经常断电），保证每次开启都能自启动成功。



#### 六、使用 NFS 存储数据到物理机

虚拟机的硬盘容量较小，我们也没必要选择存在虚拟机上，所以选择存放在物理机上，使用NFS的方式。 这里也可以直接虚拟机映射硬盘，方式不局限，这里仅作我的选择的记录。

编辑 `/etc/fstab`，添加如下

```
192.168.1.128:/data/net_traffic_storage /data/net_traffic_storage nfs defaults 0 0 
192.168.1.128:/data/mitmproxy_traffic /data/mitmproxy_traffic nfs defaults 0 0 
```

在重启过程中出现过一些因为先后顺序导致的挂载失败，因此写了个计划任务去判断：

```
0 */1 * * * cat /proc/mounts | grep -q 192.168.1.128 || systemctl restart remote-fs.target && systemctl restart supervisor
```

#### 七、另一种流量分流的方式（进程分流）（2022年1月更新）

上面的方式分流是因为 debian 虚拟机网关已经在流量记录的线路中了，对于网关自己来说就只有一条出口线路，直接设置就OK了。

那么更常见的情况，如网关直连两条线路，要求在网关上进行分流应该怎么操作？ 最近遇到这样的需求，下面总结一下对应的方法。

大体思路是利用 cgroup 标记 mitmproxy 的流量，然后用 iptables 和  ip rule 控制其从流量记录的网卡出站，实现记录的目的。

网络介绍：

```
enp0s1 192.168.30.2
enp0s2 192.168.40.2
```
两张网卡，默认路由走 enp0s1 出口，为非流量记录线路，enp0s2 为流量记录线路。

iptables 规则：


```
iptables -t nat -A PREROUTING -s 192.168.40.0/24 -p tcp -m tcp --dport 80 -j DNAT --to-destination 192.168.40.2:23333
iptables -t nat -A PREROUTING -s 192.168.40.0/24 -p tcp -m tcp --dport 443 -j DNAT --to-destination 192.168.40.2:23333
iptables -t nat -A PREROUTING -s 192.168.40.0/24 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.40.2:23333
iptables -t mangle -A OUTPUT -m cgroup --cgroup 0x00110011 -j MARK --set-mark 11
iptables -t nat -A POSTROUTING -m cgroup --cgroup 0x00110011 -o enp0s2 -j MASQUERADE
```
PREROUTING 的规则和上面第一种方法一样，只不过指定了网卡的地址，效果没有区别。

mangle 和 POSTROUTING 的规则需要结合 cgroup 的规则一起理解，就是将其 mark ，然后将 mark 后的流量从 enp0s2 网卡出站。

下面设置 cgroup:

```
apt-get install  cgroup-tools cgroup-bin
mkdir /sys/fs/cgroup/net_cls/mitmproxy
cd /sys/fs/cgroup/net_cls/mitmproxy
echo 0x00110011 > net_cls.classid
```
这里是创建了一个 cgroup 规则，然后创建了一个 mitmproxy 的组，classid 设置为 `0x00110011`。

然后设置路由

```
echo 11 mitmproxy >> /etc/iproute2/rt_tables

ip rule add fwmark 11 table mitmproxy

ip route add default via 192.168.40.1 table mitmproxy
```

这些命令的作用在下文 **OpenVPN Client as gateway** 都有解释，这里就不详细多做解释了。

然后使用 cgroup 启动 mitmdump ：


```
cgexec -g net_cls:mitmproxy mitmdump --mode transparent --showhost -p 23333 
```
这样 mitmdump 的所有流量都会从网卡 enp0s2 出站了，流量也会被记录下来，这样就完美实现需求了。

值得注意的是， `ip route` 和 `cgroup` 相关的规则是临时性的，和 `iptables` 一样，需要进行设置才能实现重启自动生效，这里就不展开了。


这里我们已经实现了流量审计的功能，接下来需要考虑的是如何远程接入审计系统。



0x02 OpenVPN 的网络架构
------------------



![/styles/images/openvpn_traffic/p2.png]({{ '/styles/images/openvpn_traffic/p2.png' | prepend: site.baseurl }})


网络流量走向为（在openvpn 接入的一台服务器上执行）

```
└─$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.2.189 (192.168.2.189)
 2  192.168.2.200 (192.168.2.200)  
 3  192.168.1.1 (192.168.1.1)  
 4  192.168.8.1 (192.168.8.1)  
 5  * * *
 ......
```



0x03 OpenVPN Client as gateway 实现
------------------

这个需求出现的场景是，我们希望所有接入OpenVPN的客户端，不使用 Server(192.168.2.189) 作为最终的网关，而是使用某一个固定的客户端(192.168.2.200)作为最终的网关。 经过搜索，发现这个需求还比较小众，最终发现了[某个帖子](https://forums.openvpn.net/viewtopic.php?t=22069)里提到了同样的需求，我们以这个帖子为参考。



#### 一、OpenVPN 的 Bridge 和 Route 对比

这篇帖子中提到仅有OpenVPN bridge 模式才能实现这样的功能。先解释一下 bridge 模式和 route 模式的区别（来自官方论坛）：

TAP 设备（bridge 模式）：

- 可以传输非 TCP/IP 层的流量，也就是第二层。
- 需要桥接网络



有下列需求的话，必须使用 bridge 模式

- 需要 LAN 和 VPN 客户端处在同一个广播域
- 需要 LAN 的 DHCP 服务器给 VPN 客户端提供 DHCP 地址
- 需要 Windows 服务器在 VPN 网络中，使用网络发现等功能



TAP 设备的优点：

- 类似一个真正的网络适配器
- 可以传输任何网络协议（IPv4, IPv6, Netalk, IPX, 等等）
- 工作在第二层
- 可以用来桥接网络



TAP 设备的缺点：

- VPN 传输更多广播流量
- VPN 传输更多以太网报文头
- 扩展型差
- 不能用在安卓或者 IOS 设备



相应的 route 模式，也就是 TUN 设备，工作在第三层，优势和劣势与 bridge 模式相反。

根据我们的需求，这里我们选择更底层的 bridge 模式，流量开销的缺点基本可以忽略不计。



#### 二、搭建 OpenVPN （bridge 模式）

基本的搭建我们就不赘述了，可以使用官方文档或者一键搭建脚本，需要注意的是，根据官方文档，我们搭建 bridge 模式的话，需要自己创建虚拟网卡 brxxx ，但是在云上一创建就会挂掉，这里我们需要使用其他方式来实现该功能，参考：[is-it-possible-to-set-up-a-bridged-vpn-in-aws](https://superuser.com/questions/1531392/is-it-possible-to-set-up-a-bridged-vpn-in-aws-to-connect-multiple-clients-in-a-s)

在 Server 端配置中写入

```
dev tap

# Define your IP address range within a valid subnet
server-bridge 192.168.2.189 255.255.255.0 192.168.2.200 192.168.2.250
client-to-client

# Set server tap0 ip address on startup
up /etc/openvpn/server/openvpn-server-up.sh

# Set client config dir
client-config-dir /etc/openvpn/ccd
```

其中 `openvpn-server-up.sh` 的内容为，并添加可执行权限。

```
#!/bin/sh
ifconfig tap0 192.168.2.189 netmask 255.255.255.0 broadcast 192.168.2.255
```

在 `/etc/openvpn/ccd/` 目录下创建跟 Client 配置文件同名的文件，如 wangxiaoqing， 写入

```
ifconfig-push 192.168.2.211 255.255.255.0
```

代表给 wangxiaoqing 这个用户固定 192.168.2.211 的 IP 地址，有多个用户就以此类推。

这样 Server 端启动时会自动启动 `openvpn-server-up.sh` 的脚本，给 tap0 的虚拟网卡指定网络配置。



最后的 Server 配置发出来大家参考：



```
port 1194
proto tcp
dev tap
ca ca.crt
cert server.crt
key server.key
dh dh.pem
auth SHA512
tls-crypt tc.key
topology subnet
script-security 2
server-bridge 192.168.2.189 255.255.255.0 192.168.2.200 192.168.2.250
push "dhcp-option DNS 192.168.8.1"
push "redirect-gateway def1 bypass-dhcp"
keepalive 10 120
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
crl-verify crl.pem

client-to-client
up /etc/openvpn/server/openvpn-server-up.sh
client-config-dir /etc/openvpn/ccd
```

通过这样的配置，我们就可以实现 Client 接入 VPN 后，访问其他 Client ，或者通过 Server 端上网了。



#### 三、配置 Server 端的路由

通过上面的配置，我们实现了最基础也是最常用的 OpenVPN 架构，多 Client 通过 Server 上网。如果仅仅是这样的话，bridge 和 route 模式几乎没区别。 但接下来的设置，就只有 bridge 模式才能实现了。

为了让所有 Client 使用其中一个 Client (`192.168.2.200`)作为路由，我们需要在 Server 端进行如下设置。

1. 先添加一个路由表

```
echo 100 vpnroute >> /etc/iproute2/rt_tables
```

2. 给 路由表 vpnroute 定义地址范围

```
ip rule add from 192.168.2.0/24 table vpnroute
```

3.  给 vpnroute 添加路由

```
ip route add 192.168.2.0/24 via 192.168.2.189 dev tap0 table vpnroute
ip route add default via 192.168.2.200 dev tap0 table vpnroute
```

第一条为`192.168.2.0/24` 网段的地址走原始的 Server 网关 `192.168.2.189`,

第二条为其他的地址走默认的 `192.168.2.200` 的 Client 网关。

通过命令 `ip route show table vpnroute` 就能看到我们刚刚添加的路由：

```
root@openvpn-server:~# ip route show table vpnroute
default via 192.168.2.200 dev tap0
192.168.2.0/24 via 192.168.2.189 dev tap0
```

这种根据源地址来定义路由的方式，叫 **policy based routing** ，而直接通过 route 命令修改的系统路由表，只能根据目的地址的不同来定义路由策略，不能满足我们的需求。



#### 四、配置 Client 端作为网关

Client(`192.168.2.200`) 本来是一个普通的 Linux 服务器，想把它作为 gateway ，是需要一些额外配置的。

开启转发：

```
sysctl –w net.ipv4.ip_forward=1
```

设置 SNAT

```
iptables -t nat  -A POSTROUTING -s 192.168.2.0/24 ! -d 192.168.2.0/24  -j MASQUERADE  -o eth0
```

通过这样设置，如果 Client 只有一个网络出口的话，就已经可以实现我们需要的功能了。 之后会具体讲到一种多网络接口的特殊情况。



#### 五、Client 多网络出口的情况下

`192.168.2.200` 这台服务器，作为网关时本地有好几张网卡都可以出外网：

```
10.90.45.20/24
192.168.1.128/24
192.168.8.105/24
```



默认的路由表是走 `10.90.45.1` 这个网关出去上网的，正常的情况下这个机器并不需要走流量审计的网络。这里就涉及到如何让来自 OpenVPN 的流量走 `192.168.1.128/24` 这个网卡的路由。（ 考虑过使用 `192.168.1.1` 这台审计网关直接接入 OpenVPN ，但因为虚拟机的网卡是通过桥接模式映射的，再通过 bridge 接入 openvpn 冲突了（猜测），所以就选择直接在物理机上接入。）

这里我们还是使用上文提到的  `policy-based route `，通过路由表规则来配置：

先确定通过其中一个网卡能顺利出网：

```
traceroute --interface=enp1s0f1 8.8.8.8
```

然后配置路由规则：

```
echo 201 debian_lan >> /etc/iproute2/rt_tables

ip route add default via 192.168.1.1 dev enp1s0f1 table debian_lan
ip rule add from 192.168.1.0/24 table debian_lan
ip rule add fwmark 0x1111 table debian_lan
```

使用 iptables 转发

```
/sbin/iptables -t mangle -A PREROUTING -i tap0 -s 192.168.2.0/24 ! -d 192.168.2.0/24 -j MARK --set-mark 0x1111
/sbin/iptables -t nat  -A POSTROUTING -s 192.168.2.0/24 ! -d 192.168.2.0/24  -j MASQUERADE  -o enp1s0f1
```

其他命令在前文已经解释过了，这里需要特别注意的就是如下两句：

```
ip rule add fwmark 0x1111 table debian_lan
iptables -t mangle -A PREROUTING -i tap0 -s 192.168.2.0/24 ! -d 192.168.2.0/24 -j MARK --set-mark 0x1111
```

第一条的意思是给 路由表 debian_lan 添加标记，第二条 iptables 的意思是在走路由策略之前，给流量包添加标记。符合这个标记的流量包就会走 debian_lan 这个路由表。 最开始我缺少这两条规则，NAT 一直不成功，但是如果尝试默认路由的话又可以 ，加上这两条规则做一下标记后就完美成功了。



0x04 一些不足
------------------

以上这套方案，其实还是有很多问题的，比如结构比较复杂耦合程度较高，最严重的问题还是：

1. 只能根据端口号判断 http(s) 流量，如果是非标准端口号，比如 8443 甚至 38443 这种端口，那就没有办法做证书劫持了，只能记录加密流量，同时其他协议流量也不能解密。所以也记录了全流量作为补充。
2. 用户体验不太好，中间人劫持肯定是会弹出不信任的警告的，要么手动接受要么导入根证书，就算是根证书也因为 CN 和 SAN 的限制，无法让浏览器或者其他检验了该名称的工具实现完美信任。



能实现目前的程度我暂时觉得已经不错了，后续如果有其他思路的话再进一步改进。



0x05 参考链接
------------------

[Using a client as a gateway](https://forums.openvpn.net/viewtopic.php?t=22069)

[Is it possible to set up a bridged VPN in AWS to connect multiple clients in a single LAN that will allow broadcasts between them?](https://superuser.com/questions/1531392/is-it-possible-to-set-up-a-bridged-vpn-in-aws-to-connect-multiple-clients-in-a-s)

[Set up your Debian router / gateway in 10 minutes](https://gridscale.io/en/community/tutorials/debian-router-gateway/)

[howto-transparent](https://docs.mitmproxy.org/stable/howto-transparent/)

[iptables深入解析-mangle篇](https://developer.aliyun.com/article/417740)

[Route the traffic over specific interface for a process in linux](https://superuser.com/questions/271915/route-the-traffic-over-specific-interface-for-a-process-in-linux)

















