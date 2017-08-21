---
layout: post
title:  "DNS日志记录方法"
date:   2017-08-21 13:05:01 +0800
categories: Audit
tag: DNS Log 
---

* content
{:toc}



经过yuange大佬的提示，快速定位后门的方法之一，检查dns记录，但是yuange博客的命令试了一下不起效果。

于是自己搜索了一下，把结果记录在这。



Windows开启dns cache log
-------------

直接上命令：


```
net stop dnscache

type nul > %systemroot%\system32\dnsrsvlr.log
type nul > %systemroot%\system32\dnsrslvr.log
type nul > %systemroot%\system32\asyncreg.log

cacls %systemroot%\system32\dnsrsvlr.log /E /G "NETWORK SERVICE":W
cacls %systemroot%\system32\dnsrslvr.log /E /G "NETWORK SERVICE":W
cacls %systemroot%\system32\asyncreg.log /E /G "NETWORK SERVICE":W

net start dnscache
```

这里有个小问题，这个dnsrslvr.log文件只会记录一小段时间的dns记录，过一段时间会刷新重新记录，目前还没找到不刷新的方法。

参考链接：  

[http://blog.nsfocus.net/open-dns-client-service-log/](http://blog.nsfocus.net/open-dns-client-service-log/)




解析DNS log
--------------

去查看dns日志文件dnsrslvr.log会发现比较乱比较杂，我写了一个简单的py脚本去解析这个日志文件。

脚本功能很简单，就是正则取域名的那一项，当然你还可以添加其他的操作，如解析dns记录，查询dns whois信息，查询归属地之类的。

目前过滤了isatap。




文件重定向的坑
-------------------

很早就踩过这个重定向的坑无数次了，没想到用python的时候又踩了一次，还搞了很久，最后Stack Overflow解决的，不得不说Stack Overflow真是神器。

具体的说就是32位python在c:\\windows\\system32\\ 目录下时会自动跳转到sysWOW64 目录下，解决方法是调用kernel.dll里Wow64DisableWow64FsRedirection

和Wow64RevertWow64FsRedirection来禁用掉windows的文件重定向。然后直接访问就行了。


最终脚本见： 

[https://github.com/Green-m/Demo/blob/master/dns_log_parse.py](https://github.com/Green-m/Demo/blob/master/dns_log_parse.py)


windows的ETW方式
------------------

windows 8.1和 windows server 2012 R2及以上版本的操作系统，可以下载补丁直接以标准的windows日志格式记录dns log，windows server 2016可以直接开启。

由于时间关系这里我就没测试了，官网的文档写的非常详细。

微软官方文档：

https://technet.microsoft.com/en-us/library/dn800669(v=ws.11).aspx


