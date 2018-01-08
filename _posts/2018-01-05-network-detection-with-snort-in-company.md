---
layout: post
title:  "使用Snort检测企业流量"
date:   2018-01-05 16:10:01 +0800
categories: IDS 
tag: Snort 
---

* content
{:toc}




0x00 前言
------------------

虽然自己已经从红队换到了蓝队阵营，但是发了几次文章，大部分还是红队方向的东西。刚好新年伊始，开个好头，18年第一篇文章写一下上个季度从事的Snort相关的研究，供大家参考。   

当然，这里的主要适用范围是在中小型互联网企业，尤其是人员较少，然后又不怎么重视安全（不愿意花钱）的那种，大公司、财力雄厚的公司可以直接忽略我这篇博客。  


0x01 Snort相关介绍
------------------

Snort IDS（入侵检测系统）是一个强大的网络入侵检测系统。它具有实时数据流量分析和记录IP网络数据包的能力，能够进行协议分析，对网络数据包内容进行搜索/匹配。它能够检测各种不同的攻击方式，对攻击进行实时报警。此外，Snort是开源的入侵检测系统，并具有很好的扩展性和可移植性。  


乌云drops有一篇文章讲解得很详细，包括检测原理、配置文件、规则语法等。如果你对Snort了解不多，没有实际踩过坑的话，建议去阅读一下相关资料，方便对下文有更深刻的认识。  

原文地址：  
(已无法访问，可以在乌云镜像里看到)  
SNORT入侵检测系统
http://drops.wooyun.org/%e8%bf%90%e7%bb%b4%e5%ae%89%e5%85%a8/9232  


0x02 选择Snort的原因  
------------------

最开始说要做流量分析，就知道Snort是业内比较老牌的产品，然后去调研了一下相关产品，最后着重考虑Snort和Bro两者之间选择。  

Snort和Bro的比较：

![/styles/images/snort/p1.png]({{ '/styles/images/snort/p1.png' | prepend: site.baseurl }})


总的来说，选中Snort的原因是更方便上手，社区活跃，资料拓展很多。  

Bro的资料远远不如Snort，而且需要花时间研究怎么写脚本。这对于一个人手不够的企业来说，花费的精力与回报比例太低，肯定是不划算的。  

Bro主要的优点在于高速网络中大流量的分析能力比Snort强，这一点在后面使用Snort的过程中已经完美解决。

综上所述，最适合我们的还是Snort。


0x03 具有特色的Snort架构
------------------

对于企业来说，流量都是不小的，特别是某些大型业务，带宽都是万兆网络，对于这样的大型流量，直接上Snort肯定是没用的，根本抗不住，特别是我司这样多个机房遍布各地，想用常规方法不太现实。  

特色的Snort架构如图：

![/styles/images/snort/p2.png]({{ '/styles/images/snort/p2.png' | prepend: site.baseurl }})


用的工具有

- 流量抓取dumpcap  

- 流量分析Snort  

- 数据库mysql  

- 数据转换并导入数据库插件barnyard2

- 前端Snorby


大体流程为：  

外网与内网进出口的流量，通过交换机旁路镜像到服务器上，在服务器上通过Snort进行分析，将Snort分析的结果导入数据库中，然后将流量源文件保存一段时间。 

为啥说这是具有特色的Snort结构呢，因为特色在于我们将Snort的流量嗅探和流量分析分离了开来，只用Snort做流量分析，流量抓取这部分我们自己来实现和控制。  

这样做的好处在于：  

1. 自己实现流量抓取能够将抓取到的流量用来做其他的分析，也可以保存起来方便溯源。  

1. 解决性能问题。这样分开实现，可以通过高性能的流量抓取，和多进程的Snort实例来做到不会因为流量数据过大导致漏报。  



0x04 安装及配置
------------------

### 安装Snort、barnyard2 

安装的时候踩过不少坑，一步一步找教程来安装的。后来整理了一个自动安装脚本，在其他机器上安装的时候方便多了。  

安装脚本见 [https://github.com/Green-m/Demo/blob/master/snort/snort_install.sh ](https://github.com/Green-m/Demo/blob/master/snort/snort_install.sh)  

`注：` 这个一键安装脚本是以`Centos 7 `为基础的，其他发行版请自己修改脚本。  

在官网（https://www.snort.org/downloads#rules ）推荐注册用户，可以下载更完整的规则库。  

同时在 https://rules.emergingthreats.net/open/snort-2.9.0/ 也能下载到更多的规则库，不过有的会比较冗余，自己取舍。  

安装mysql数据库这里就不说了，比较简单，网上搜一下就有。  

### 配置Snort  

安装snort结束后，直接运行可能会报一些错，根据报错去snort.conf 注释掉一些规则文件，或者去规则文件里注释相关规则。

直到`snort -c /etc/snort.conf -T ` 提示没有报错代表配置成功。  

说一下几个关键的配置： 

设置 HOME_NET 变量，如果不知道怎么修改建议设置成any  
```
ipvar HOME_NET any 
```

如果想将日志导入到数据库中的话，需要配置输出日志格式为unified2。  
```
output unified2: filename snort.log, limit 128
```

如果你的流量很大，建议将memcap设置成大一点的值，最大的值为1073741824，可以充分利用内存。  
```
preprocessor stream5_global: track_tcp yes, \
   track_udp yes, \
   track_icmp no, \
   max_tcp 262144, \
   memcap 1073741824, \
   max_udp 131072, \
   max_active_responses 2, \
   min_response_seconds 5
```


### 配置barnyard2


打开配置文件/etc/snort/barnyard.conf ，修改如下设置，其中最后一行是数据库的配置，根据实际情况修改
```
#config logdir:/var/log/barnyard2 
#config hostname: localhost
#config interface: eth0
#config waldo_file:/var/log/snort/barnyard2.waldo
#config dump_payload
#config dump_payload_verbose
#output database: log,mysql,user=user password=xxxxx dbname=snort host=x.x.x.x
```

配置完成后执行命令，直到不报错。
```
barnyard2 -c /etc/snort/barnyard.conf -T
```

### 安装snorby  

snorby项目地址https://github.com/Snorby/snorby ，安装前需要安装ruby, rails, rake等环境，具体安装步骤就不提供了，根据官方文档操作吧，坑有点多。  

安装完成后需要修改config目录下的一些配置来访问数据库，会自己初始化数据库的。  

当然你也可以选择其他前端，snorby坑有点多，不怎么推荐。主要是也没有别的太好的选择，如果有更好的前端，欢迎推荐。  



### 常见问题FAQ 

**我遇到的一些问题及解决方法：**  

- snort报警只有sid没有名称  

类似这样的报警Snort Alert [1:2100376:8]，主要是因为/etc/snort/sid-msg.map 文件没有映射该事件ID与名字导致的。  

这种情况多出现在有第三方规则添加进来，但是没有添加映射导致的。需要将第三方的sid-msg.map与原来的sid-msg.map合并到一个文件。  


0x05 运行及调度
------------------

在安装和配置完成后，准备工作完成，下面讲一下我是如何实现大流量的流量抓取及分析的。

### dumpcap 抓流量 

选择wireshark的dumpcap的原因是为了跨平台，保证windows和linux下都能正常运行，虽然大部分情况下都是linux。  

这个其实影响不大，可以选择其他的比如PF_RING、tcpdump、libpcap等都行。  

```
dumpcap -i eth0 -B 1024 -b duration:60 -b files:1000 -w /pcap/test.pcap
```

这里的命令是一分钟保存一个文件，一共1000个文件，免得磁盘爆掉。具体数量需要根据你服务器的硬件配置和流量大小来决定。  

然后通过supervisor保持运行，一直抓取。

其中一个机房的流量大概是每分钟1.4G，dumpcap占单核CPU 60%的性能，完全吃得住。  

如果流量过大可以换成PF_RING抓取，也可以考虑从交换机那将流量分光，导入到多台服务器上处理。 优化的方式很多。  

### Snort 分析流量文件  

最开始在流量比较小的时候，单个Snort实例是完全能够支撑下来的。  后面到了其他流量巨大的机器的时候，Snort分析一个pcap可能就需要3分钟，而dumpcap会每分钟都产生一个pcap包。这样生产者的速度比消费者的速度快，肯定是不行的。   

为了解决这样的问题，我写了一个调度脚本来控制Snort分析，同时启动多个Snort进程来分析多个pcap文件。 

关键代码如下：  

```
	# pcaplist_total 表示目录下所有的pcap文件，以集合的方式表示
        pcaplist_total = list(set(pcaplist_total) - set(opening_files))
        # pcaplist_tohandle  表示未分析的pcap文件  
        pcaplist_tohandle = list(set(pcaplist_total) - set(pcaplist_handled))

        # 当未处理的pcap文件数小于三个，继续等待，可以根据情况自己设置
        if len(pcaplist_tohandle) < 3:
            logging.info("Waiting for more pcap files for {} times...".format(cycletime))
            sleeptime = 10*cycletime
            if sleeptime > 600:
                sleep(600)
            else:
                sleep(sleeptime)
            cycletime +=1
            continue

        cycletime=1
        snort_option = snort_command + '"{}"'.format(" ".join(pcaplist_tohandle))

        logging.info("Snort is starting to analyze {} pcap files...".format(len(pcaplist_tohandle)))

        # 有几个pcap文件就分成几份，最多分成8份，最多启动8个进程
        pcaplist_tohandle_chunked = chunks(pcaplist_tohandle,8)

        start_time = timeit.default_timer()
        pool = threadpool.ThreadPool(8) # most 8 processes 

        # 通过snort -c /etc/snort/snort.conf -q -l /var/log/snort --pcap-list 来调用snort  
        reqs = threadpool.makeRequests(call_snort, pcaplist_tohandle_chunked)
        [pool.putRequest(req) for req in reqs]
        pool.wait()
```


某些参数如进程数量，等待时间，文件个数可以根据自己的情况来调整。
完整代码见 [https://github.com/Green-m/Demo/blob/master/snort/snort_multiprocess.py](https://github.com/Green-m/Demo/blob/master/snort/snort_multiprocess.py) 

由于我们的流量还是属于比较大的，单机房每分钟1到2G，用这种方式来处理流量，是完全能够处理过来的，最多可能会有一到两分钟的延迟。 

当然我这里选用的是物理机，128核，性能过剩。 如果流量不够大的话完全可以用虚拟机，8核以上应该都完全够用。

和dumpcap类似，为了保证该python脚本能够持续运行，用supervisor作为守护进程。  



### barnyard2 

barnyard没啥好说的，就是一个将snort分析过后的日志导入数据库的插件，其支持多种数据库。  
执行
```
barnyard2 -c /etc/snort/barnyard.conf -d /var/log/snort -f snort.log -l /var/log/barnyard2/ -D
```
放入后台运行。  

不过有一点需要注意，barnyard2在没接收到数据的时候可能会自己退出，最开始想用supervisor去守护的，发现不行，于是自己写了一个crontab定时任务去检测，脚本如下  
```
#!/bin/sh
ps -ef |grep barnyard2|grep -v grep
if [ $? -ne 0 ]
then
barnyard2 -c /etc/snort/barnyard.conf -d /var/log/snort -f snort.log -l /var/log/barnyard2/ -D
echo `date` "Start bardyard2" >> /var/log/barnyard2/barnyard.log
fi
```

保存，并设置定时任务启动即可。 


### Snorby 

安装过程略过。  

启动用thin 加 nginx做反向代理，可以解决数据量过大的时候无响应。  

用snorby默认的发送邮件的功能，是一个大坑。  经常发不出来，也不能修改时间，默认每天晚上12点给你发送邮件，太尼玛坑爹了。  

虽然可以用  
```
ReportMailer.daily_report.deliver, ReportMailer.weekly_report.deliver, ReportMailer.monthly_report.deliver
```
来发送邮件，但是发送的时候还是会把昨天0点到24点的报告发出来，并不会从当前的时间往前推24个小时。  

需要修改源码```.app/mailers/report_mailer.rb```里 ```report = Snorby::Report.build_report('yesterday', timezone)```将yesterday修改为last_24，邮件就可以发送最近24小时了。  

定时发送邮件的脚本如下，保存为snorby-report.sh  
```
if [ "$#" -ne 1 ]; then exit 1; fi
job=$1
cd /home/Green-m/snorby-master
RAILS_ENV=production bundle exec rails r ReportMailer.${job}_report.deliver
```

crontab 设置
```
10 10 * * * /home/Green-m/snorby-report.sh daily
15 10 * * 1 /home/Green-m/snorby-report.sh weekly
20 10 1 * * /home/Green-m/snorby-report.sh monthly
```

如果想修改收件人，也到app/mailers/report_mailer.rb修改源码，改成想要的收件人即可。   


整体效果如图 

![/styles/images/snort/p3.png]({{ '/styles/images/snort/p3.png' | prepend: site.baseurl }})



0x06 优化及监控  
------------------

### snorby优化 

有人肯定要问，说了这么多，好像一个性能优化的点都没说到，到底这一套能不能跑起来啊。  

其实我最开始也想了很多优化的点，比如流量太大，dumpcap抓流量会不会抓不过来丢包， snort分析不过来怎么办，全部导进一个数据库数据库崩掉了怎么办。  

后面搭好了发现之前这些根本不是问题，或者说我准备了一些优化的方式，但是根本没用上。  

兴高采烈的用了好几天，日志量好几百万条的时候，打开snorby前端，前端直接卡死了，真TM尴尬。  

进后台看了下rails的日志，发现一条查询直接花了100多秒，rails默认超时时间60S，直接就报错了。  

```
SQL (37692.992ms)  select count(*) from aggregated_events;
SQL (67081.531ms)
 select e.sid, e.cid, e.signature,
 e.classification_id, e.users_count,
 e.notes_count, e.timestamp, e.user_id,
 a.number_of_events from aggregated_events a
 inner join event e on a.event_id = e.id
 order by e.timestamp desc limit 45 offset 0
```

aggregated_events表是一个视图view，view定义如下 
```
select `iphdr`.`ip_src` AS `ip_src`,`iphdr`.`ip_dst` AS `ip_dst`,`event`.`signature` AS `signature`,max(`event`.`id`) AS `event_id`,count(0) AS `number_of_events` from (`event` join `iphdr` on(((`event`.`sid` = `iphdr`.`sid`) and (`event`.`cid` = `iphdr`.`cid`)))) where isnull(`event`.`classification_id`) group by `iphdr`.`ip_src`,`iphdr`.`ip_dst`,`event`.`signature` limit 10;
```

最后决定直接加个时间限制，默认查询七天内，简单粗暴
```
select `iphdr`.`ip_src` AS `ip_src`,`iphdr`.`ip_dst` AS `ip_dst`,`event`.`signature` AS `signature`,max(`event`.`id`) AS `event_id`,count(0) AS `number_of_events` from (`event` join `iphdr` on(((`event`.`sid` = `iphdr`.`sid`) and (`event`.`cid` = `iphdr`.`cid`)))) where isnull(`event`.`classification_id`) and `event`.`timestamp` >= NOW() - INTERVAL 7 DAY  group by `iphdr`.`ip_src`,`iphdr`.`ip_dst`,`event`.`signature` limit 10;
```

查询时间缩短到十来秒，虽然不够快，但是也勉强够用了，以后如果再有更高的性能需求，可能真的需要自己开发前端了。 

### 设置白名单  

公司是有自动巡检程序的，每次定时启动都会给snort带来很多不必要的负担，这部分的数据我们是要过滤的，没有任何意义。  

最开始以为white_list.rules添加ip就能被snort过滤了，用了一下发现没效果，最后还是搜到了配置在snort.conf 里的
```
config bpf_file: /etc/snort/ignore.bpf
```

编辑/etc/snort/ignore.bpf，语法有点类似于tcpdump，多个ip可以通过or链接
```
not (host 192.168.1.105 or host 192.168.1.106 or host 192.168.1.107)
```

也可以添加网段
```
not net 192.168.1.0/24
```

甚至还支持端口过滤。

根据情况，将自己需要过滤的ip添加进去即可。


### 监控高危事件  

在使用中会发现，有时候高危事件snort检测到了，但是并没有及时的提醒。如果要一直看着的话，这太浪费时间和精力了，不geek。  

想着elasticsearch都有报警插件elastalert，搜了一圈，mysql并没有找到，于是只能自己造了，不过工作量也不大，一会就写好了。  

关键的数据库查询如下  

```
def snort_query(sid_list):
    query_result = []
    connection = pymysql.connect(host='xxxx',
                             user='snorby',
                             password='xxxxx',
                             db='snorby',
                             charset='utf8mb4',
                             cursorclass=pymysql.cursors.DictCursor)

    try:
        with connection.cursor() as cursor:

            for sid in sid_list:
                sql = """
                        select `signature`.sig_name AS `signature`, inet_ntoa(`iphdr`.`ip_src`) AS `ip_src`, inet_ntoa(`iphdr`.`ip_dst`) AS `ip_dst`, `event`.`timestamp` AS `timestamp`, count(0) AS `number_of_events`, `signature`.`sig_sid` AS `SID` ,
                        (SELECT name from asset_names WHERE asset_names.ip_address = `iphdr`.`ip_src`) AS `IP_SRC_INFO`,
                        (SELECT name from asset_names WHERE asset_names.ip_address = `iphdr`.`ip_dst`) AS `IP_DST_INFO` 
                        from ((`event` join `signature` on (`event`.`signature` = `signature`.`sig_id`)) join `iphdr` on (((`event`.`sid` = `iphdr`.`sid`) and (`event`.`cid` = `iphdr`.`cid`))) ) 
                        where isnull(`event`.`classification_id`) and `signature`.`sig_sid`={} and `event`.`timestamp` >= NOW() - INTERVAL 5 MINUTE  group by `iphdr`.`ip_src`,`iphdr`.`ip_dst`,`event`.`signature`;
                        """.format(sid)

                cursor.execute(sql)
                results = cursor.fetchall()
                if results:
                    query_result.append(results)
                    


    finally:
        connection.close()

    return query_result

if __name__ == '__main__':


    sid_list = [12345,678910] # SID
    query_result = snort_query(sid_list)
    if query_result :
        mailbody = info_to_mailbody(query_result)
        send_email(mailbody,"Greenm@xxx.com")
        logging.info("Sended email")
```


通过crontab每5分钟执行一次，每次查询近五分钟的事件，如果有满足的sid，就直接发邮件报警。   

这里脚本我没有放出完整的，发送邮件的部分各位自己实现吧。  

0x07 总结  
------------------

在搞这一套流量分析的时候，还是踩了很多坑的，不过还好最后也终于是搞定了。  

其他的地方我觉得都OK，唯一差劲的是前端，毕竟是开源的，也就这个样子了，有时间的话一定自己去开发一套。  

有问题欢迎联系我，微博私信或者github提Issue都可以。

2018年开个好头，祝大家新年开心。
