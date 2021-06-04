---
layout: post
title:  "使用轻量级工具Sysmon监视你的系统"
date:   2016-12-16 12:04:01 +0800
categories: tools
tag: Sysmon
---

* content
{:toc}


## 0x00 前言

本文于2016年12月16日首发于 freebuf 。 因未知原因，现在(2021年6月)已无法访问2018年以前发布于 freebuf 的文章，故重新搬运到本人博客，仅做存档作用。 

**注意**： 本文发布时间较早，部分内容可能已经失效或不正确，请读者阅读时甄别。

本文原始链接：https://www.freebuf.com/sectool/122779.html



## 0x01 sysmon介绍

**sysmon是由Windows Sysinternals出品的一款Sysinternals系列中的工具。它以系统服务和设备驱动程序的方法安装在系统上，并保持常驻性。sysmon用来监视和记录系统活动，并记录到windows事件日志，可以提供有关进程创建，网络链接和文件创建时间更改的详细信息。**

通过收集使用Windows事件集合或SIEM代理生成的事件，然后分析它们，您可以识别恶意或异常活动，并了解入侵者和恶意软件在您的网络上如何操作。

## 0x02 sysmon特点

用完整的命令行记录下子进程和父进程的创建行为。

使用sha1（默认），MD5，SHA256或IMPHASH记录进程镜像文件的hash值。可以同时使用多个hash，包括进程创建过程中的进程GUID。

每个事件中包含session的GUID。

> 1.记录驱动程序或者加载的DLL镜像的签名及其hash。
>
> 2.记录磁盘和卷的原始数据的读取请求。
>
> 3.记录网络连接，包括每个连接的源进程，IP地址，端口号，主机名和端口名（可选）
>
> 4.如果更改注册表则自动重新加载配置。
>
> 5.具有规则过滤，以便动态包括或排除某些事件。
>
> 6.在加载进程的初期生成事件，能记录在复杂的内核模式运行的恶意软件。

## 0x03 安装

```
    Install:    Sysmon.exe -i <configfile>
    [-h <[sha1|md5|sha256|imphash|*],...>] [-n [<process,...>]]
    [-l (<process,...>)]

    Configure:  Sysmon.exe -c <configfile>
                  [--|[-h <[sha1|md5|sha256|imphash|*],...>] [-n [<process,...>]]
                   [-l [<process,...>]]]

    Uninstall:  Sysmon.exe -u
```

-c 更新或显示配置

-h 指定hash记录的算法

-i 安装，可用xml文件来更新配置文件

-l 记录加载模块，可指定进程

-m 安装事件清单

-n 记录网络链接

-r 检测证书是否撤销

-u 卸载服务和驱动

**一键安装：**

```
 sysmon -accepteula  –i -n
```

**指定配置文件（安装时请用-i）**

```
sysmon -c xxx.xml
```

注：安装需要管理员权限并重启，windows 7 或者以上，服务器系统windows server 2012 及以上。

## 0x04 配置文件

在实际环境中，往往生产环境和使用需求不尽相同，而记录所有的日志又显得繁琐和庞大，这时候自定义规则显得尤为重要。

sysmon提供了xml格式的配置文件来让用户自定义过滤规则，配置文件的东西比较多，把个人总结的一些东西贴出来方便大家交流。

一个xml配置文件的示例（xml大小写敏感）

```
  <Sysmon schemaversion="3.20">
<!-- Capture all hashes -->
<HashAlgorithms>*</HashAlgorithms>
<EventFiltering>
<!-- Log all drivers except if the signature -->
 <!-- contains Microsoft or Windows -->
 <DriverLoad onmatch="exclude">
<Signature condition="contains">microsoft</Signature>
 <Signature condition="contains">windows</Signature>
</DriverLoad>
 <!-- Do not log process termination -->
<ProcessTerminate onmatch="include" />
 <!-- Log network connection if the destination port equal 443 -->
<!-- or 80, and process isn't InternetExplorer -->
<NetworkConnect onmatch="include">
<DestinationPort>443</DestinationPort>
<DestinationPort>80</DestinationPort>
</NetworkConnect>
<NetworkConnect onmatch="exclude">
<Image condition="end with">iexplore.exe</Image>
 </NetworkConnect>
 </EventFiltering>
</Sysmon>
```

可选择的**事件过滤器**有

> ProcessCreate  进程创建
>
> FileCreateTime 进程创建时间
>
> NetworkConnect 网络链接
>
> ProcessTermina 进程结束
>
> DriverLoad    驱动加载
>
> ImageLoad     镜像加载
>
> CreateRemoteTh 远程线程创建
>
> RawAccessRead  驱动器读取
>
> ProcessAccess  进程访问
>
> FileCreate    文件创建
>
> RegistryEvent  注册表事件
>
> FileCreateStre 文件流创建

**过滤器事件**的选项：

**ProcessCreate**

```
UtcTime, ProcessGuid, ProcessId, Image, CommandLine, CurrentDirectory, User, LogonGuid, LogonId, TerminalSessionId, IntegrityLevel, Hashes, ParentProcessGuid, ParentProcessId, ParentImage, ParentCommandLine
```

**FileCreateTime**

```
UtcTime, ProcessGuid, ProcessId, Image, TargetFilename, CreationUtcTime, PreviousCreationUtcTime
```

**NetworkConnect**

```
UtcTime, ProcessGuid, ProcessId, Image, User, Protocol, Initiated, SourceIsIpv6, SourceIp, SourceHostname, SourcePort, SourcePortName, DestinationIsIpv6, DestinationIp, DestinationHostname, DestinationPort, DestinationPortName
```

**ProcessTerminate**

```
UtcTime, ProcessGuid, ProcessId, Image
```

**DriverLoad**

```
UtcTime, ImageLoaded, Hashes, Signed, Signature
```

**ImageLoad**

```
UtcTime, ProcessGuid, ProcessId, Image, ImageLoaded, Hashes, Signed, Signature
```

**CreateRemoteThread**

```
UtcTime, SourceProcessGuid, SourceProcessId, SourceImage, TargetProcessGuid, TargetProcessId, TargetImage, NewThreadId, StartAddress, StartModule, StartFunction
```

**RawAccessRead**

```
UtcTime, ProcessGuid, ProcessId, Image, Device
```

**ProcessAccess**

```
UtcTime, SourceProcessGUID, SourceProcessId, SourceThreadId, SourceImage, TargetProcessGUID, TargetProcessId, TargetImage, GrantedAccess, CallTrace
```

**FileCreate**

```
UtcTime, ProcessGuid, ProcessId, Image, TargetFilename, CreationUtcTime
```

**RegistryEvent**

```
UtcTime, ProcessGuid, ProcessId, Image, EventType, TargetObject
```

**FileCreateStreamHash**

```
UtcTime, ProcessGuid, ProcessId, Image, TargetFilename, CreationUtcTime, Hash
```

注：更多过滤器的详细说明请参考[https://technet.microsoft.com/en-us/sysinternals/sysmon](https://technet.microsoft.com/en-us/sysinternals/sysmon)。

onmatch选项只能设置为include或exclude。

condition根据不同的需求可设置为如下值：

| **Condition** | **Description** |
| --------- | ----------- |
| **Is** | Default, values are equals |
| **is not** | Values are different |
| **Contains** | The field contains this value |
| **Excludes** | The field does not contain this value |
| **begin with** | The field begins with this value |
| **end with** | The field ends with this value |
| **less than** | Lexicographical comparison is less than zero |
| **more than** | Lexicographical comparison is more than zero |
| **Image** | Match an image path (full path or only image name). For example: lsass.exe will match c:\windows\system32\lsass.exe |



下述规则将记录所有创建tmp格式或exe格式文件的行为：

```
<FileCreate onmatch="include">
<TargetFilename condition="end with">.tmp</TargetFilename>
<TargetFilename condition="end with">.exe</TargetFilename>
</FileCreate>
```

下述规则将记录加载没有签名的镜像行为：

```
<ImageLoad onmatch="exclude">       <Signed condition="is">true</Signed>    </ImageLoad>
```

你可以为同一个事件同时指定include和exclude，但一个onmatch里只能赋值为exclude或include。exclude规则里覆盖include规则。同一规则里的过滤条件用逻辑或连接，不同规则里的过滤条件用逻辑与连接。

下述规则将记录除了chrome之外的所有访问80或443的网络记录

```
<NetworkConnect onmatch="exclude">
<Image condition="end with">chrome.exe</Image>
</NetworkConnect>
<NetworkConnect onmatch="include">
<DestinationPort condition="is">80</DestinationPort>
<DestinationPort condition="is">443</DestinationPort>
</NetworkConnect>
```

贴一个我PC机器上的xml配置文件，在实际生产中请根据实际情况调整

```

<Sysmon schemaversion="3.20">



  <!-- Capture all hashes -->



  <HashAlgorithms>*</HashAlgorithms>



  <EventFiltering>



    <!-- Log all drivers except if the signature -->



    <!-- contains Microsoft or Windows -->



    <DriverLoad onmatch="exclude">



      <Signature condition="contains">Microsoft</Signature>



      <Signature condition="contains">Windows</Signature>



    </DriverLoad>



    <ProcessTerminate onmatch="include" >



      <Image condition="end with">MsMpEng.exe</Image>



    </ProcessTerminate>



    <!-- Log network connection if the destination port equal 443 -->



    <!-- or 80, and process isn't InternetExplorer -->



    <!--NetworkConnect onmatch="include">



      <DestinationPort>443</DestinationPort>



      <DestinationPort>80</DestinationPort >



    </NetworkConnect -->



    <FileCreateTime onmatch="exclude" >



      <Image condition="end with">chrome.exe</Image>



    </FileCreateTime>



    <ImageLoad onmatch="include">



      <Signed condition="is">false</Signed>



    </ImageLoad>



    <!-- Log access rights for lsass.exe or winlogon.exe is not PROCESS_QUERY_INFORMATION -->



    <ProcessAccess onmatch="exclude">



      <GrantedAccess condition="is">0x1400</GrantedAccess>



    </ProcessAccess>



    <ProcessAccess onmatch="include">



      <TargetImage condition="end with">lsass.exe</TargetImage>



      <TargetImage condition="end with">winlogon.exe</TargetImage>



    </ProcessAccess>



    <NetworkConnect onmatch="exclude">



      <Image condition="end with">chrome.exe</Image>



      <SourcePort condition="is">137</SourcePort>



      <SourcePortName condition="is">llmnr</SourcePortName>



      <DestinationPortName condition="is">llmnr</DestinationPortName>



    </NetworkConnect>



    <CreateRemoteThread onmatch="include">



      <TargetImage condition="end with">explorer.exe</TargetImage>



      <TargetImage condition="end with">svchost.exe</TargetImage>



      <TargetImage condition="end with">winlogon.exe</TargetImage>



      <SourceImage condition="end with">powershell.exe</SourceImage>



    </CreateRemoteThread>



  </EventFiltering>



</Sysmon>
```

注： 有时候explorer.exe或者svchost会发出http请求，不用惊慌，仔细看下链接，可能只是访问Akamai或者Microsoft（刚刚测试的时候发现svchost连到新加坡111.221.29.254:443，查了一下才发现是微软新加坡分公司）。如果实在不放心，就在防火墙里配置规则把explorer的所有流量阻止，万无一失。

## 0x05 测试示例

### 记录dump hash

笔者之所以看中了这个软件，因为最近更新可以记录访问进程的信息，通过事件CreateRemoteThread和ProcessAccess可以记录特殊操作，如dump hash和线程注入等。下面用mimikatz来演示。

当我运行了mimikatz抓hash之后，默认在"Applications and Services Logs/Microsoft/Windows/Sysmon/Operational"里能找到日志记录。

如下图：

![mimi]({{'/styles/images/sysmon/mimi.png' | prepend: site.baseurl }})

记录下来mimikatz访问了进程lsass.exe，属于进程访问事件。那么从哪里能看出来是进行了dump hash操作呢，关键的一点就是 GrantedAccess的值为0x143A，这个值表示什么呢，我们查阅msdn可知：

```
0x1000  PROCESS_QUERY_LIMITED_INFORMATION 受限制的进程查询信息
0x0400  PROCESS_QUERY_INFORMATION   进程查询权限，包括token，退出代码，优先级，改权限自动继承PROCESS_QUERY_LIMITED_INFORMATION
0x0002  PROCESS_CREATE_THREAD 创建线程权限
0x0008  PROCESS_VM_OPERATION  需要对进程的地址空间执行操作，常用于VirtualProtectEx和WriteProcessMemory。
0x0010  PROCESS_VM_READ 读取进程中的内存
0x0020  PROCESS_VM_WRITE  使用WriteProcessMemory写进程内存，看见WriteProcessMemory，大概应该知道会有些什么操作了。
```

将上面的权限按位异或即得0x143A，及表示mimikatz对lsass拥有上述访问权限，包括写进程内存和读进程内存，这样就能获取到用户口令。而一般的进程访问只需要0x1400，也就是只有进程查询权限，这里mimikatz明显有恶意行为。

而普通的进程对于lsass.exe的访问如下图：

![ms]({{'/styles/images/sysmon/ms.png' | prepend: site.baseurl }})

这里记录下来的是windows defender对lsass.exe的访问，这里的GrantedAccess是属于普通的标准进程访问权限，calltrace也有详细的记录，属于正常的进程访问。

更多关于进程访问的权限可参考

[https://msdn.microsoft.com/en-us/library/windows/desktop/ms684880](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684880)

### **恶意网络连接和操作**

用meterpreter回连，可以看到连接的详细信息，包括ip，端口，pid等，如图![test]({{'/styles/images/sysmon/test.png' | prepend: site.baseurl }})

![msf]({{'/styles/images/sysmon/msf.png' | prepend: site.baseurl }})

执行migrate操作也会记录在CreateRemoteThread里，如图：

![migrate]({{'/styles/images/sysmon/migrate.png' | prepend: site.baseurl }})

这里就能很明显的看出来test.exe把恶意代码注入到explorer.exe，借explorer这个躯壳来隐藏自己。而整个过程中我的windows defender一个泡都没冒。

而filetime过滤器会将系统上所有文件的覆盖和创建时间修改时间等都记录下来：

![filetime]({{'/styles/images/sysmon/filetime.png' | prepend: site.baseurl }})



很多恶意程序留后门和dll劫持都会有替换文件和修改时间的操作（这里不是恶意行为，只是用来作一个演示）。

在本地加载了任何无签名的exe和dll等可执行文件：

![imageload]({{'/styles/images/sysmon/imageload.png' | prepend: site.baseurl }})



在面对dll注入，各种白加黑，利用dll劫持等操作的恶意软件都会因为这一项被记录下来（有签名的恶意软件极少）

## 0x06 日志记录

目前的恶意软件为了对抗检测很多都有日志删除功能，为了保存好系统日志，可以将日志定期保存在本地或远程服务器，日志的默认保存在%SystemRoot%\System32\Winevt\Logs\Microsoft-Windows-Sysmon%4Operational.evtx，可在事件查看器的日志属性设置保存在远程服务器，或通过其他工具或脚本保存。

## 总结

虽然目前有很多监视和分析软件行为的工具，如process explorer，process monitor，pstools，还有各种沙盒分析软件，反汇编调试工具，以及各大杀毒软件，但sysmon最为一个轻量级的监控软件，有它的亮眼的地方，结合其他工具能让监控系统变得更容易和更效率。如有兴趣进一步交流讨论，欢迎访问笔者[微博](http://weibo.com/u/2585957090)或[博客](https://green-m.me/)。