---
layout: post
title:  "某门罗币挖矿木马分析及溯源"
date:   2017-09-17 23:41:01 +0800
categories: Reverse 
tag: malware 
---

* content
{:toc}



0x00 前言
------------------

在五月份的时候，大概就是wannacry出来的那个月，ms17-010滥用最多的时候，我在公司内部捕获到一个挖矿的样本。  

之后一直忙着做其他的，大概上个月，也就是八月份才想起来该分析一下这个样本，抽了工作之余的时间分析和溯源了一下这个挖矿病毒，博客记录一下分析的过程。

0x01 样本概述
------------------

样本名称 | MD5 
----|------
Ru1732312203.exe | 73fee66665fc878d819c21b4d5c21c88 
nthost.exe | 79c4d033d16a3c09e8d1f5d91a0a0426 
tasklsv.exe | bf1ee70b332b6cdd9a5f40a7017c002c 
winhost.exe | 93d48870d5b55c1611f01261a37f25ed 



恶意程序的主体为上面四个程序，其他还有一些tmp和dat等临时文件不一一列出。


0x02 Ru1732312203.exe样本分析  
------------------

该程序为恶意程序downloader，从地址http://123.126.106.27:280 处下载恶意程序，并更改防火墙设置，修改访问权限的策略，然后执行恶意文件。

**详细分析**  

先杀掉自己进程，确保不会重复，然后关闭防火墙。
 
![/styles/images/monero/ru_1.png]({{ '/styles/images/monero/ru_1.png' | prepend: site.baseurl }})

通过urldownloadtofile从地址http://123.126.106.27:280 ，下载程序nthost.exe tasklsv.exe winhost.exe，把ip地址硬编码写到源码里也是心大。

![/styles/images/monero/ru_2.png]({{ '/styles/images/monero/ru_2.png' | prepend: site.baseurl }})

然后创建服务nthost

![/styles/images/monero/ru_3.png]({{ '/styles/images/monero/ru_3.png' | prepend: site.baseurl }})


该程序还会通过调用wmic执行一堆查找进程然后删除的命令，这些进程看起来都是用近似系统文件名来伪装的恶意软件，通过该方式来避免和目前的程序冲突，也可能是干掉竞争对手啥的。

![/styles/images/monero/ru_4.png]({{ '/styles/images/monero/ru_4.png' | prepend: site.baseurl }})

添加防火墙规则，包括开启3389，web服务端口，关闭445端口，自己进来之后就把别人关在门外

![/styles/images/monero/ru_5.png]({{ '/styles/images/monero/ru_5.png' | prepend: site.baseurl }})

截止到9.12号，下载已经有1万多次

![/styles/images/monero/ru_6.png]({{ '/styles/images/monero/ru_6.png' | prepend: site.baseurl }})


0x03 nthost.exe样本分析  
------------------

该程序为前一个程序Ru1732312203.exe启动的服务程序，以服务的形式进一步启动挖矿主程序tasklsv.exe和winhost.exe。

启动tasklsv.exe 和winhost.exe 

![/styles/images/monero/nt_1.png]({{ '/styles/images/monero/nt_1.png' | prepend: site.baseurl }})


0x04 tasklsv.exe样本分析  
------------------

该程序为门罗币挖矿主程序，具有签名信息Promoir B.V，挖矿地址有usa.cetipool.com:8050 monero.cetipool.com:443 

work.cetipool.com:443等，通过与该域名进行数据交互来实现门罗币挖矿。

**域名相关信息**  

whois相关信息：  

2017-05-09 在godaday注册的域名，程序编译的时间为2017年5月25日  

由于隐藏了私人相关信息，看不到真实的注册人  

![/styles/images/monero/ta_1.png]({{ '/styles/images/monero/ta_1.png' | prepend: site.baseurl }})


该挖矿服务器的域名及ip相关信息，ip在近一段时间内连续多次更换过，捕获样本时指向的ip地址为47.94.136.87，现在9.17指向的ip地址为98.126.8.106

![/styles/images/monero/ta_2.png]({{ '/styles/images/monero/ta_2.png' | prepend: site.baseurl }})


**详细分析**  


整体流程为:

1.收集机器信息，初始化配置

2.创建workio线程，通过json格式提交信息

3.创建stratum_thread线程，用来和矿池相连，切换相关挖矿模式等

4.创建timer线程，根据条件判断连接等

5.创建miner线程，进行挖矿工作


程序运行截图

![/styles/images/monero/ta_3.png]({{ '/styles/images/monero/ta_3.png' | prepend: site.baseurl }})

初始化程序需要的变量等信息

![/styles/images/monero/ta_4.png]({{ '/styles/images/monero/ta_4.png' | prepend: site.baseurl }})

Workio_thread线程

![/styles/images/monero/ta_5.png]({{ '/styles/images/monero/ta_5.png' | prepend: site.baseurl }})

通过stratum协议，向门罗币地址usa.cetipool.com:8050，monero.cetipool.com:443发起请求

![/styles/images/monero/ta_6.png]({{ '/styles/images/monero/ta_6.png' | prepend: site.baseurl }})

发送登录信息，"login":"4AGF7tHM5ZpR7FRndiPMk6MxR1nSA8HnAVd5pTxspDEcCRPknAnFAMAYpw8NmtbGFAeiakw6rxNpabQ2MyBKstBY8sTXaHr","pass":"x"

![/styles/images/monero/ta_7.png]({{ '/styles/images/monero/ta_7.png' | prepend: site.baseurl }})

通过server端分配的任务ID，提交挖矿的result

![/styles/images/monero/ta_8.png]({{ '/styles/images/monero/ta_8.png' | prepend: site.baseurl }})


挖矿主线程，开始挖矿

![/styles/images/monero/ta_9.png]({{ '/styles/images/monero/ta_9.png' | prepend: site.baseurl }})


0x05 winhost.exe样本分析  
------------------

该程序为木马后门的主程序，由Themida加壳，该程序编译时间较早，为15年5月。

太久没摸过这种强壳了，没有勇气去手动拖，简单的基于行为分析一下。

exeinfo

![/styles/images/monero/win_1.png]({{ '/styles/images/monero/win_1.png' | prepend: site.baseurl }})

创建服务RASMAN，经典远控

![/styles/images/monero/win_2.png]({{ '/styles/images/monero/win_2.png' | prepend: site.baseurl }})

该样本与如下地址有网络连接

```
http://down.51-cs.cn/protect.dll
http://user.qzone.qq.com/12345678
http://qzone.qq.com/?s_url=http://user.qzone.qq.com/12345678  
```

dns请求

```
down.51-cs.cn  162.159.211.78
dos.51-cs.cn   121.41.102.66
user.qzone.qq.com 2.21.243.64
qzone.qq.com  2.21.243.59
```

目前该qq空间地址已经无法访问，推测通过QQ空间作为C&C服务端。

51-cs.cn的whois信息

![/styles/images/monero/win_3.png]({{ '/styles/images/monero/win_3.png' | prepend: site.baseurl }})

真名叫黄xx，注册邮箱为xxxxxxxxx@qq.com


该QQ号信息

![/styles/images/monero/win_4.png]({{ '/styles/images/monero/win_4.png' | prepend: site.baseurl }})

黄xx从15年开始就用该域名作为C&C服务端，并持续更换对应的ip，最近一次更新是在2017-09-17，其QQ个人说明也像是C&C服务端命令。

![/styles/images/monero/win_4.1.png]({{ '/styles/images/monero/win_4.1.png' | prepend: site.baseurl }})

![/styles/images/monero/win_4.2.png]({{ '/styles/images/monero/win_4.2.png' | prepend: site.baseurl }})

通过在其QQ空间找到他手机号

![/styles/images/monero/win_5.png]({{ '/styles/images/monero/win_5.png' | prepend: site.baseurl }})

支付宝账号也是该邮箱注册的

![/styles/images/monero/win_6.png]({{ '/styles/images/monero/win_6.png' | prepend: site.baseurl }})

0x06 总结  
------------------

该挖矿木马很可能是通过MS17-010传播，然后通过Downloader下载木马和挖矿程序。

万幸的是不像wannacry一样在内网通过smb传播，不然后果有点严重。

至于黄xx同学，做黑的也要先把屁股擦干净，不然小心警察叔叔找你喝茶。
