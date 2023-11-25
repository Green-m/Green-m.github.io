---
layout: post
title:  "家庭 Wi-Fi 晋级路线"
date:   2023-11-24 18:00:01 +0800
categories: Misc
---

* content
{:toc}



### 0x00 前言
--------------------

每个爱折腾的人都不会放弃折腾家里的网络的，无论是NAS，软路由，还是电视盒子，游戏机，WI-FI漫游等，甚至选择终极方案把家里整成一个机房，只要你想就没有不能折腾的地方。

由于提到的NAS，软路由这些我都基本上折腾过了，本文主要聚焦的是  WI-FI 这一块，毕竟普通人最常最直接的体验就是 Wi-Fi，WI-FI 的稳定流畅与否某种程度上和幸福程度直接挂钩。

没有人想在玩着游戏的时候，被夫人大喊，“网怎么这么卡，视频卡住了，快帮我看一下！”————那真是痛苦的回忆。


### 0x01 蛮荒阶段
--------------------

从有上外网需求开始，家里经历过多个无线 AP 设备，从 K2 到 K2P，最开始从刷机 K2P 装某些插件实现上网，是稳定运行了很长一段时间。 

后来搬家，设备也添上了 NAS 后，随着硬盘越来越大，媒体资源越来越高清，K2P 的信号强度和 WI-FI 速度已经捉襟见肘，主要的矛盾体现在日益增长的网络传输需求与羸弱的硬件传输能力的冲突。 

K2P宣称的是能够到千兆，但是刷了固件上了插件后怎么测试都只有600M左右，都2023年了，在近距离都跑不到千兆，没有 Wi-Fi 6，也没有自动漫游切换，这样的路由肯定要被淘汰的，因此 K2P 就被我扔到了架子上吃灰。



### 0x02 失望的 AP 选择
--------------------


考虑到已经购置了软路由，拨号和主路由就由 x86 软路由搞定了，所以购买硬路由只需要考虑 AP 的信号和稳定性了，当然性价比也应该考虑，毕竟没土豪到那种地步。

之前担心过软路由的稳定性，但作为主路由跑过很长一段时间发现真的挺稳定的，除了偶尔因为温度高导致一直卡顿，在后来加了一个小风扇之后就完全没有这个问题了。

对比了很多家，最终选择了 TPLINK  XDR5480 和 XDR 3040 路由器， 也看了很多评测，比如 acwifi.net 的评测，对于 TPLINK 的评价都还行，考虑到战未来的可能性，选择直接上 2.5G 网口，XDR5480 的2.5G 还不是电口，我甚至还斥巨资(100RMB)入了一个转换器。

看着 TPLINK 路由器的硬件参数，无线都3000M/5000M的，再加上这个宣传的 “易展” 功能，我想着这不是直接起飞啊，地球村近在眼前，世界与我近在咫尺！

当然如果真的这么爽，也就不会有此文了。

#### 2.1 约等于无的可配置项

首先说 TPLINK 的路由器，硬件堆料确实没问题，但软件功能也太少了，简直是残缺不堪。 我就想买回来当 AP 使用，结果就这最基本的功能都做不好。

网上找了张图：

![/styles/images/wifi_upgrade/tplink-web-ap1.png]({{ '/styles/images/wifi_upgrade/tplink-web-ap1.png' | prepend: site.baseurl }})


选择AP（有线中继）模式，IP地址和网关等全都不能手动设置，必须等自动分配。 至于那个兼容模式，不明所以，没有任何专业性的解释，仅有一行说明，但开和关我完全没感受到区别。 

还有另外一种方法，关闭DHCP服务后，去lan口主动配置 IP，网关，子网掩码等设置，好歹是可以手动配置这个路由器的地址了。

后来我选择的是手动关闭 DHCP 服务来作为 AP，至于前面那一种有什么坑，我已经忘记了，反正不太行，也不想再为了写这篇文章去复现一下了，等想起来了再说。

![/styles/images/wifi_upgrade/tplink-web-ap2.png]({{ '/styles/images/wifi_upgrade/tplink-web-ap2.png' | prepend: site.baseurl }})

当然能手动配置 LAN 口当然是好的，但是注意看图，图里只让你填了本机的 IP 地址和掩码，**并没有让你选择网关和DNS服务器**。

如果你不熟悉网络的话可能不了解这代表什么，稍微了解一点就知道这两个选项对于能不能上网至关重要。

我不知道 TPLINK 的开发者是怎么想的，因为缺失了这两个配置，导致我接入WI-FI的设备无法正确设置 DNS 。 而我的软路由作为网关，最后的 IP 地址并不是 .1，而是 .2（历史原因），我甚至怀疑是 TPLINK 默认认为网关地址是 .1 结尾，而自动根据 LAN 口设置配置了，才导致在我的环境中出现各种问题不能正确配置。

后来我只好在软路由上开启了强制 DHCP，同时设置 DHCP options 去广播 DNS。

![/styles/images/wifi_upgrade/openwrt-dhcp3.png]({{ '/styles/images/wifi_upgrade/openwrt-dhcp3.png' | prepend: site.baseurl }})

图里设置这里是 192.168.6.2，但我之前的 DNS 服务器并不是这个，我需要广播其他的DNS服务器，可能这也是导致 TPLINK 找不到 DNS 的可能之一。


而我的环境相对来说还算比较简单的，网关和 DHCP 服务器好歹还是同一个，还遇到这么多问题，如果你有旁路由这样的二级网关，那估计踩的坑会更多。




#### 2.2 坑爹的稳定性

好，到这里我所有的配置基本都好了，客户端测试连接也可以正常上网了，各种 iperf3 ，speedtest 测试网络都还可以（为了测试2.5G我还换了一个交换机），Mesh 的快速漫游也测试OK，想着终于可以起飞了，软件差点就差点，硬件总还行吧。

体验的结果就是： 光看斗鱼就经常缓存，不知道是掉线还是什么，我千兆的网清晰度连个蓝光都看不了，坑爹。

DHCP 分配IP也光出现问题，经常出现加入不了  WI-FI 的情况，我甚至一度怀疑是 openwrt 强制 DHCP 导致的问题， 但我排查 openwrt 的 DHCP 日志的时候，发现租约还没过期的客户端也会频繁触发 DHCP 的分配日志，不知道是为什么，但我感觉应该跟这个有关系。

这个断线断连我也能忍，毕竟可能不是 TPLINK 的问题，是我环境跟它的不兼容，但最不能忍的是下面这种情况：

![/styles/images/wifi_upgrade/join-wifi-issus4.png]({{ '/styles/images/wifi_upgrade/join-wifi-issus4.png' | prepend: site.baseurl }})

直接完全无法加入网络，不管是完全忘记 Wi-Fi，还是回收所有 DHCP 租约，都无法加入网络。 我可能需要睡一觉或者放那，过个半天左右就又能连上了，真的很奇怪。 关键它是偶发性的，不是持续出现的问题，我还不好排查。

我用各种测速软件测出来的结果很美好，但实际体验太差劲了，只能说一分钱一分货，我就不该贪这个便宜。  用上了 TPLINK 后， 经常听到夫人大喊“老公网卡了！”，这个时候我心里总是一紧，总会后悔自己一时冲动买的中看不中用的东西。


### 0x03 近乎完美的 AP 设备
--------------------

今年双11，我痛定思痛，痛下决心下血本买了两个 Unifi 的 U6 mesh 可乐罐，准备用来替换家里的 TPLINK。

它的包装真的挺精致的，设备造型也小巧，不像是一个路由器，据说是参考苹果的设计来的，我个人觉得还是挺符合的。

![/styles/images/wifi_upgrade/u6-mesh5.png]({{ '/styles/images/wifi_upgrade/u6-mesh5.png' | prepend: site.baseurl }})

![/styles/images/wifi_upgrade/u6-mesh6.png]({{ '/styles/images/wifi_upgrade/u6-mesh6.png' | prepend: site.baseurl }})

（图盗自什么值得买，自己忘拍了）

当然，精致是有代价的，我买的是美版的，插头是美标的插头，买回来不能直接用，还要自己去买个转接头，当然如果你家里有 POE 供电的设备就不需要了，直接接网线供电就行了。

同时，你很难想象2023年了，一个1500块钱的 AP 设备，里面连根网线都不带，这点也完全符合我对苹果设计风格的一贯印象。

想使用  Unifi 的 AP 设备，需要结合一个 AC 来控制， 虽然 Unifi 官方的 AC 非常的贵，但好处是可以自建，甚至可以用 docker 来搭建，网上已经有成熟的 docker 配置，我用的是 jacobalberty/unifi 的镜像。


#### 3.1 无可挑剔的软件体验

这里我就不细讲具体搭建过程，就讲讲我搭建好之后的实际体验效果。

简单的总结， Unifi 的软件堪称完美，基本上你能想象的功能，在 Unifi 的 AC 中你都能够控制，包括但不限于发射功率，信道，RSSI 断开链接的信号强度，2.4G 和5G 的信道宽度等等都能够设置。

甚至 LED 灯的 RGB 颜色和亮度都能够控制，我在家里把所有默认蓝色的灯都换成了绿色。

![/styles/images/wifi_upgrade/meme7.png]({{ '/styles/images/wifi_upgrade/meme7.png' | prepend: site.baseurl }})

跟其他国内大部分路由器厂商不同的是，Unifi 的 AP 是允许你直接 SSH 登陆到该设备上操作的，也就是当你遇到问题或者bug，可以直接上去看对应的 /var/log 目录来进一步分析，当然你也可以把它当成 Linux 设备，进行其他的操作。

除了高度自定义和全面的软件功能支持之外， Unifi 的界面也是非常美观和简洁的。

总览，可以一目了然的看到目前的Wi-Fi状况：

![/styles/images/wifi_upgrade/unifi_web8.png]({{ '/styles/images/wifi_upgrade/unifi_web8.png' | prepend: site.baseurl }})


目前网络的 拓扑图和流量状态（设备的图标也可以快速匹配和指定）：

![/styles/images/wifi_upgrade/unifi_web9.png]({{ '/styles/images/wifi_upgrade/unifi_web9.png' | prepend: site.baseurl }})

各个客户端的流量统计和运行状态：

![/styles/images/wifi_upgrade/unifi_web10.png]({{ '/styles/images/wifi_upgrade/unifi_web10.png' | prepend: site.baseurl }})

以及Wi-Fi接入点的覆盖，还有各个客户端的信号强度和理论最大速度。

![/styles/images/wifi_upgrade/unifi_web11.png]({{ '/styles/images/wifi_upgrade/unifi_web11.png' | prepend: site.baseurl }})






#### 3.2 硬件略有不足

Unifi 的 AP 设备，硬件上还是存在一点点不足，U6 mesh 可乐罐发热比较高，直接放置的话不算很稳容易倒。 我感觉是过度追求外观了损失了部分散热甚至是寿命。 当然目前暂时还没有发现因为散热导致的卡顿，但我也怀疑如果长期在高温的情况下寿命能否长久。

其次，1500左右的 AP 设备，居然还是千兆，这个价格和这个配置，不得不说略显落后。 据说接下来很快会出 2.5G 的 AP，但考虑到 Unifi 一贯的价格，瞬间觉得自己的千兆也完全够用了。

另外，Unifi 全家桶还有其他的设备，包括网关交换一体机，功能也十分强大，和这个 AP 可以统一配置，就是价格实在是承受不起，当然这倒不是他们硬件的不足，主要是我的资金不足。

#### 3.3 自建 Unifi controller 遇到的一点小 BUG (Docker)

使用 Docker 自建 AC 控制器挺顺利的，但是使用一段时间后发现，AC 经常会出现已采用的设备丢失的问题，只要 AC 或 AP 一重启，就会丢失。 然后一直卡在接管中或者接管失败的状态上，虽然不影响 AP 的网络，但影响页面统计和状态还是挺不爽的。

排查和搜索资料一段时间后发现，主要原因就是 AC 会根据自身的IP和端口接管AP，这就导致用 docker 搭建的 AC 会出问题，因为容器内部的IP地址跟宿主机不同。

解决方案是

1. 如果你的端口没有改过，就是默认的 8080，那么可以直接在 AC 中覆盖 IP 地址： 系统 -> 高级 -> 通知主机，选择覆盖，填写需要覆盖的 IP 地址即可。

2. 如果不只是IP地址需要修改，还包括端口的话，就需要修改容器中的配置文件，手动修改端口和 Host ip地址：

```
unifi.http.port=48080
system_ip=192.168.x.xxx
```

配置文件地址：unifi/data/system.properties

如果映射的端口不一样的话，记得docker-compose文件也要修改。

当然如果你不是用 Docker 搭建的，那就完全不存在这个问题。


### 0x04 软路由 + AP 实施 VLAN
--------------------

既然 Unifi 的 AP 什么都能做，那我也正好把我的 VLAN 计划跟提上日程了。

先可以介绍一下背景，为什么要规划 VLAN。 因为我现在的设备不算多，VLAN 对于我来说主要是为了解决网络隔离的问题，毕竟作为安全研究员，网络安全是最关心的，国内的设备也不太信得过。 当然以后设备也能避免广播风暴。那些什么某米的路由，电视，智能设备之类的，我是能不用就尽量不用，就算用了也不会让它跟我的主要设备在同一个网段的。

网上的资料划分 VLAN 主要是靠网管交换机来划分，而我的 VLAN 划分主要是给无线设备划分的，考虑到多个房间多个设备的统一，用不同的网口划分好像不太合适（当然也许可以，毕竟我没真正接触过网管型的交换机）。

我决定直接用手上现成的 x86  openwrt 来划分 VLAN。找了网上一圈资料，很多配置的时候都是这样的：

![/styles/images/wifi_upgrade/openwrt_vlan12.png]({{ '/styles/images/wifi_upgrade/openwrt_vlan12.png' | prepend: site.baseurl }})

我在我的设备上怎么都找不到这个交换机的页面，后来才知道 x86 的 openwrt 如果没有**交换机芯片**的话是没有这个配置界面的，只能通过虚拟接口去实现，还好也不算麻烦。



#### 4.1 openwrt 配置软件 VLAN

首先在 openwrt 里的 网络 -> 接口 点击添加新接口，添加虚拟 VLAN 接口。

![/styles/images/wifi_upgrade/openwrt_vlan13.png]({{ '/styles/images/wifi_upgrade/openwrt_vlan13.png' | prepend: site.baseurl }})

这里的接口 eth0.2 代表的就是在 eth0 上创建一个 VLAN ID =2 的虚拟接口，如果你想在多个接口上都创建，那么就可以写

```
eth0.2 eth1.2 eth2.2 eth3.2 eth4.2
eth0.10 eth1.10 eth2.10 eth3.10 eth4.10
```

根据你自己的物理接口编号来调整。 先不要选择在多个接口上创建桥接，填好名字后直接点击创建。

创建好之后，选择刚刚创建的 VLAN 接口（一定不能选择原始的 eth0 eth2 这种物理接口，只能选择 VLAN 接口！）：

![/styles/images/wifi_upgrade/openwrt_vlan14.png]({{ '/styles/images/wifi_upgrade/openwrt_vlan14.png' | prepend: site.baseurl }})

设置好桥接接口，同时再设置好对应的 DHCP ，DHCP 上的 DHCP选项建议配置一下，如：

```
6,8.8.8.8, 114.114.114.114 
```
表示 DHCP 分配内网地址的时候，直接让客户端配置如上的 DNS 服务器。 这个可以根据自己需要修改， 因为我这里都是 IoT 设备，且与我主要的网段隔开，所以 DNS 就配置公共的 DNS 就行了。


#### 4.2 openwrt 配置防火墙

接口配置好之后，还需要再配置一下防火墙，不然是不能直接到 wan 接口的。

这个比较简单，直接在 网络 -> 防火墙中新增，选择刚刚创建的 LAN ，允许转发到 wan 就行了。

如果你还想让主网络单向可以访问到 VLAN 网络，那么可以在防火墙中配置允许源区域 LAN 访问 VLAN。 不配置的话默认的 LAN 和 VLAN 是完全不互通的。


#### 4.3 Unifi AC 中配置

在 Unifi AC 的web界面中，还需要创建一个单独的网络来匹配这个 VLAN 。 在设置 -> 网络中，选择新建虚拟网络，填写好 VLAN ID和网络名称，如图：

![/styles/images/wifi_upgrade/unifi_ac_vlan15.png]({{ '/styles/images/wifi_upgrade/unifi_ac_vlan15.png' | prepend: site.baseurl }})

然后创建一个新的 WI-FI，匹配刚刚创建的网络即可：

![/styles/images/wifi_upgrade/unifi_ac_vlan16.png]({{ '/styles/images/wifi_upgrade/unifi_ac_vlan16.png' | prepend: site.baseurl }})


这样，连接这个 WI-FI 下的所有客户端，都跟主网络 LAN 隔开了，只能跟互联网相连。 如果你还嫌不够的话，还可以开启**客户端设备隔离**，实现客户端和客户端之间不能互相访问。



### 0x05 总结
--------------------

Unifi 的 AP 确实做得很好，如果你有钱，那么上 Unifi 全家桶可能是更好的选择，如果你没钱也没有折腾的需求，那么普通的 TPlink，小米，华为等等路由器都是挺不错的选择，笔者只是写出自身使用 TPlink 遇到的坑，网上也有很多说这些路由器不错的。 总之，适合自己需求，网络和性能都够稳定，就是好选择。


### 0xff 参考
--------------------

[UniFi 控制器设备采用丢失问题排查](https://blog.yiguochen.com/unifi-adoption.html)
[OpenWRT设置VLAN](https://www.wyr.me/post/708)
[ 十八聊智能 篇一百一十八：AC+AP全屋WiFi方案改造实战，UniFi评测：全套UBNT，应付全屋智能家居+手机无缝漫游 ](https://post.smzdm.com/p/adwgzdqp/)
[ 老房子WiFi布网攻略 篇三十九：openwrt vlan详解与WAN口、LAN口任意设定指南 ](https://post.smzdm.com/p/az6p9ggr/)

