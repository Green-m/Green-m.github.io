# kerberos 是如何工作的？

## 0x00 前言

本文主要介绍一些 Kerberos 协议的原理和围绕其展开的攻防研究，针对黄金/白银票据如何工作的细节进行了一些研究。

## 0x01 NTLM 简介

在介绍Kerberos协议之前，先简单介绍一下NTLM的相关知识。

在 Windows中， 存储的密码Hash就叫做 NTLM Hash，其中NTLM 全称是 “NT LAN Manager”，NTLM Hash 由hash 算法（[HMAC](https://en.wikipedia.org/wiki/HMAC) - [MD5](https://en.wikipedia.org/wiki/MD5)）对密码运算后得到。

在局域网或者域中，经常会使用NTLM 协议来进行认证，大概流程如下：

1. Client 向 Server端发送请求，要求使用 Tom 用户进行认证。
2. Server 端收到后，生成一个随机字符串C，使用 Tom 账号的 NTLM Hash 对C加密。将未加密的C发送给 Client。
3. Client 收到C后，用 Tom 的NTLM Hash 对C 加密，然后发送给 Server。
4. Server 收到后，验证两边加密的结果是否相等，相等就表示通过。



通过这样的认证流程，就能够在不发送用户密码明文和Hash的情况下进行认证了。

## 0x02 Pass The Hash

通过对上面原理的介绍，我们能够知道NTLM Hash的关键性。一旦获取了某个账号的NTLM Hash，我们就能够进行 Pass The Hash 攻击（PTH）。

**Pass The Hash 攻击** 

- Metasploit ( exploit/windows/smb/psexec, tools/exploit/psexec.rb, )
- Psexec.exe
- 





- kuhl_m_pac_validationInfo_to_PAC   生成 PAC 信息
- kuhl_m_pac_signature 对生成的pac信息签名（通常，PAC验证信息会包含在KRB_AP_REQ中发送出去）
- kuhl_m_kerberos_ticket_createAppEncTicketPart 创建 EncTicketPart ，（加密数据，由EncASRepPart or EncTGSRepPart组成）



## 0x09 参考链接

https://www.secpulse.com/archives/94848.html

http://cert.europa.eu/static/WhitePapers/CERT-EU-SWP_14_07_PassTheGolden_Ticket_v1_1.pdf

https://tools.ietf.org/html/rfc4120#section-5.8

http://www.securityandit.com/network/kerberos-protocol-understanding/

https://github.com/gentilkiwi/mimikatz

https://en.wikipedia.org/wiki/NT_LAN_Manager

https://blogs.msdn.microsoft.com/openspecification/2009/04/24/understanding-microsoft-kerberos-pac-validation/

