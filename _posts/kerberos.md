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

通过对上面原理的介绍，我们发现，即使不能获取到某个账号的明文密码，仅仅获取NTLM Hash也是大有可为的。一旦获取了某个账号的NTLM Hash，我们就能够进行 Pass The Hash 攻击（PTH）。

**Pass The Hash 攻击** 

- Metasploit ( exploit/windows/smb/psexec, tools/exploit/psexec.rb, )
- Psexec.exe
- mimikatz

![](图片地址)

## 0x03 Kerberos 协议简介

Kerberos 协议认证的原理和上述提到的 NTLM 有很多类似的地方，下面简单介绍一下Kerberos 的认证流程。

1. （AS-REQ）Client 发送用户名 Tom 到 KDC （Key Distribution Center）以向 AS （Authentication Service）请求 TGT 票据等信息。
2. （AS-REP）收到请求后，AS 生成随机字符串 Session Key，（？？？加上时间戳 timestamp）使用 Tom 的 NTLM Hash 对 Session Key 加密得到密文 A，再使用账号 krbtgt 的 NTLM Hash 对 Session Key 、 Client Info和 timestamp 加密得到 TGT，A 和 TGT 一起返回给 Client。
3. （TGS-REQ） Client 收到请求后，使用自身的 NTLM Hash 解密 A 就能得到 Session Key，然后使用 Session Key 对 Client Info 和 timestamp 加密得到 B，加上 TGT ，发送给 KDC中的 TGS。
4. （TGS-REP）TGS  收到请求后，使用 **krbtgt** 的 NTLM Hash 解密 TGT，得到 Session Key 和 timestamp 以及 Client Info，同时，使用 TGT 解密出的 Session Key 解密密文B，得到Client Info 和 timestamp。 比对这两部分解密得到的内容以验证是否通过。通过后，生成一个新的随机数 Session Key2，并用它加密 client info 和 timestamp 得到密文 enc-part；使用服务器计算机的NTLM Hash 对 session key2 和 client info 以及 timestamp 加密得到最终的 Ticket，返回给 Client。
5. （AP-REQ）Client 使用 Ticket 和 enc-part 直接请求某服务。
6. （AP-REP） 对Ticket 和 enc-part 解密后进行验证授权。

**注意**：

- Kerberos 协议设计的思路就是用来在不受信的环境下进行认证的协议。

-  krbtgt 账号的 NTLM Hash 理论上只存在于 KDC 中。这意味着 TGT 只能由 KDC 来解密。如果krbtgt 账号的NTLM Hash泄露了，那么 TGT 就能被解密甚至伪造。伪造的 TGT 叫做黄金票据。
- Ticket 是由服务器计算机本身的 NTLM Hash 加密的，Client 不能解密。如果该Hash 泄露，那么就可以解密甚至伪造 Ticket。伪造的 Ticket 叫做白银票据。
- 在上述的流程中，涉及到时间戳 timestamp，由于它的存在，才使得被第三方获取了加密信息 A 、B、TGT不会在短时间内被暴力破解。timestamp 一般时间为8小时。
- Kerberos 协议和 NTLM 协议都会使用 NTLM Hash 对生成的任意随机数加密，然后比对结果。 Kerberos 的主要区别在于添加了第三方-------KDC参与到认证过程中。
- Client info 中包含域名信息、Client 名称等



## 0x04 Pass The Ticket

### 黄金票据（Gold Ticket）

攻击者在获取了 krbtgt 账号的 ntlm hash 后，就能自己任意伪造 TGT（包括session key，client info等内容），然后直接发送给 kdc，实现任意权限伪造。	

**黄金票据常用攻击方法和工具**

- mimikatz
- Metasploit
- 



**伪造黄金票据所需信息**

- 目标的域名信息（如：vln2012.local)
- 目标域名的 SID （可通过 `lsadump::lsa` 获取）
- 伪造的账号名称（如： Administrator）
- 伪造账号的 RID （RID 是 SID 最右边的一部分，默认的 Administrator 账号的 RID 是 500）
- 伪造账号所属组的 RIDs （如 `Domain Users` 和 `Domain Admins` 的RIDs 分别是 512 和 513）
- 一个或者多个 krbtgt 账号的 Hash （如： RC4 加密的 NTLM Hash，AES-128 HMAC 、AES-256 HMAC）



**黄金票据的特点**

- 黄金票据可以冒充域内任何用户生成对应的 Kerberos TGT 票证。因此，它可以用来任何合法的用户，虽然一般都会选择域管理员帐户；

- 可以离线创建。一旦收集了创建黄金票据所需的所有数据，就不需要连接到域（参见上面的内容）;
- 在有限期内都有效。Mimikatz默认值为10年，也提供了参数可以创建任意有效期的黄金票据。但如果域管理员修改了 krbtgt 账户的 Hash， 黄金票据就会立即失效；
- 域组策略中修改 Kerberos Policy 中的生存周期（lifetime）不会影响到黄金票据（默认更新时间为10小时，总有效时间为7天）。KDC 会验证 TGT 中的存活时间戳，而不是域控上的组策略设置。攻击者生成任何生存周期的黄金票据;
- 被冒充的帐户重置密码不会影响到黄金票据；
- 重置 krbtgt 账号的密钥会使所有的黄金票据失效。 
- Windows 事件日志不区分合法TGT与黄金票据。 因此目前没有通用的规则来检测黄金票据使用。



### 白银票据 （Silver Ticket）

















## 0x08 PAC



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

https://ldapwiki.com/wiki/Golden%20Ticket