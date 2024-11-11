---
layout: post
title:  "Linux 服务器短信接收同步"
date:   2023-04-06 11:32:01 +0800
categories: Misc
---

* content
{:toc}



### 0x00 前言
--------------------

最近有多张电话卡的需求，记录一下我的解决方案，目前认为是比较完美的，已经稳定运行快一个月。


### 0x01 手机之间同步的困扰
--------------------

我的 iphone 不支持双卡，带两个手机又比较麻烦，所以最开始就想利用一下闲置在家里的旧手机，iOS 支持 imessage 收到信息自动同步到新手机。 但中间遇到两个新问题： 

1）老手机长期开机待机电量消耗很快，特别是 iphone 电池容量也不大，如果一直插着电对手机电池也不是特别好；
2）我的旧手机是第一代 iphone SE，它不支持升级到 IOS16 了，而登陆新手机的 icloud 账号居然提示我不能登陆，请先升级到 IOS16（Only apple can do!）。  

我尝试了多种解决方案，均登陆失败。我不太懂苹果的逻辑，但我的旧手机看来是永远都登陆不了我主用的 icloud 账号了。

第一个问题还好，可以考虑用智能插座等解决方案去定时开关解决，但第二个账号问题就彻底堵死了想用手机同步短信的路子。 没办法，只能考虑其他路线了。


### ~~0x02 SIM800C USB 模块~~
----------

网上搜索时，无意中发现了有开发板可以做到接收短信的功能，同时还有支持 USB 接口的模块。 马上在淘宝上下单了一个，还带天线支持全网通，才50多块钱。 唯一的缺点可能就是该模块没有外包装，不防猫。
图片大概长这样：

![/styles/images/gammu_sms/sim800c.jpg]({{ '/styles/images/gammu_sms/sim800c.jpg' | prepend: site.baseurl }})

把这个东西直接插到电脑上，Linux 或者 windows 都行，我是直接插到家里的 Linux 服务器上的，插上去之后还不能直接用，需要先安装CH340的串口驱动程序，我这里是店家直接给的驱动，直接 `make install` 安装就行，也可以自己参考网上的资料安装。 安装后就能通过 USB 接口检测到设备了。


```bash
❯ ls /dev/ttyUSB*
/dev/ttyUSB0
❯ lsusb
Bus 001 Device 008: ID 1a86:7523 QinHeng Electronics HL-340 USB-Serial adapter
...
```

### 0x02.5 A7670G 模块（2024年11月6日更新）

伴随着长时间的使用，可能是因为它只支持 2G 信号，而随着现在 2G 逐渐被淘汰，SIM800C 的信号变得越来越不稳定了，经常收不到短信甚至掉线。

于是我开始考虑重新购入一个新设备，最终挑中了 A7670G 模块，它支持 4G、3G、2G 各种频段，接口也支持串口和 USB 口连接，很方便。 但该设备不如 SIM800C 轻便，还需要额外供电，也贵了不少，成本来到了150到200块钱。

大概长这样(外壳可以选配)：

![/styles/images/gammu_sms/sim800c.jpg]({{ '/styles/images/gammu_sms/A7670G.jpg' | prepend: site.baseurl }})


后续过程和上面 SIM800C 一般无异，就不赘述了。 

切换成这个后发现稳定了很多，虽然短信接收因为漫游的原因还是有延迟，但起码不再掉线不漏接短信了，非常满意。



### 0x03 gammu 接收短信与推送
----------------------------
该模块原生是通过串口去控制，通过 AT 命令收发短信和打电话等操作的，我在测试的时候也用过这个命令去控制。 但我其实不需要过于复杂的功能，仅仅需要收发短信就可以了，问了下 chatGPT，它推荐我用 gammu 这个工具去控制，我才发现有这样一个工具，而且效果确实也挺好。 

gammu 的介绍和基本操作我这里就不详细介绍了，感兴趣的可以自己看官方文档或者其他入门资料介绍。 我是通过 `gammu-smsd` 这个常驻进程去实现的：


配置文件(有省略)：
```
#/etc/gammu-smsdrc
[gammu]
# Please configure this!
port = /dev/ttyUSB0
connection = at115200
# Debugging
logformat = textall
# SMSD configuration, see gammu-smsdrc(5)
[smsd]
service = files
RunOnReceive = /etc/sms_send_tg.sh
...
```

为了能够同步给远端的其他手机，我选择用 `telegram bot` 实现，当然也可以选择其他对网络更友好的方式，比如 `slack bot`，飞书等等。 脚本内容如下：

```
#!/bin/bash

# Replace YOUR_TOKEN and YOUR_CHAT_ID with your Telegram bot token and chat ID
TOKEN="xxxx"
CHAT_ID="-1xxxx"

echo "Received log from tg bash script: $SMS_1_NUMBER: $SMS_1_TEXT" >> /xxxxxx/sms_ganmu.log

# Send the message to the Telegram bot using cURL
curl -q -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
-d chat_id="$CHAT_ID" \
-d text="Received SMS from $SMS_1_NUMBER: $SMS_1_TEXT" && echo "Send message successful."
```

### 0x04 gammu 发送短信
-----------------------

因为我比较少用到主动发短信的功能，大部分时候是手动发短信，所以没有实现自动化，这里就简单介绍一下手动发短信的功能：

```
echo "hello,my friend" | su gammu -s /bin/bash -c 'gammu-smsd-inject TEXT +8613333333333'
```
主要是借助了 `gammu-smsd-inject` 程序实现的。


### 0x05 gammu 监控与通知
---------------------------

虽然 `gammu-smsd` 进程我是通过 `systemd` 控制自动启动的，但毕竟是外设硬件，存在着太多不可预测的意外情况（你也不知道什么时候猫猫发疯给你咬了），所以我们还需要有一个监控该进程用户通知提醒的进程。 

官方提供了一个程序 `gammu-smsd-monitor` 来监控健康状态，但该程序没有考虑后台持续运行的情况，也没有提供附带的报警功能，所以需要自己去改进实现。

我是通过 `systemd` 来实现的，当该服务失败或者异常时会调用 `bark-alert-user` 服务：

```
# /etc/systemd/system/gammu-smsd-monitor.service
[Unit]
Description=Gammu SMSD Monitor
After=network.target
OnFailure=bark-alert-user@%n.service

[Service]
Type=simple
ExecStart=/usr/bin/gammu-smsd-monitor -d 15
Restart=no
RestartSec=5
LimitNOFILE=65535
ExecStop=/usr/bin/killall  gammu-smsd-monitor

[Install]
WantedBy=multi-user.target
```


创建报警服务`bark-alert-user`，该服务触发时会执行 `/usr/bin/bark_alert.sh` ：

```
# /etc/systemd/system/bark-alert-user@.service
[Unit]
Description=Bark alert send to user for %i

[Service]
Type=oneshot
ExecStart=/usr/bin/bark_alert.sh %i
User=nobody
Group=systemd-journal
```

脚本内容是发送一个 http 请求到 bark，然后会将通知发送到我的手机上：
```
#!/bin/bash
curl "https://api.day.app/<token>/$1"
```

读者根据需求也可以使用其他的类似功能来实现。


### 0x06 gammu 的坑
-----------------
1. 使用的时候需要注意， `gammu` 控制是支持直接用 `gammu` 程序 和 `gammu-smsd`的，但这两个程序其实是互斥的，也就是说两者只能有一个同时占用你的 USB 设备。我当时在这里卡了很久，以为是我的设备出了问题 ，在我的命令行 `gammu` 直接测试设备成功时，`gammu-smsd` 启动就会报错。所以，请务必同时只使用 `gammu` 或者 `gammu-smsd` 来控制你的设备。

2. `gammu` 默认是通过文件存储短信的，所以如果以非 root 用户启动服务的话，需要注意服务用户对收件箱和发件箱的权限，否则可能会出现狂发短信的情况。


### 0xff 参考

gammu 官方文档： https://docs.gammu.org/  

自建短信中转服务：https://zhuanlan.zhihu.com/p/53387245  

get-notification-when-systemd-monitored-service-enters-failed-state： https://serverfault.com/questions/694818/get-notification-when-systemd-monitored-service-enters-failed-state  









