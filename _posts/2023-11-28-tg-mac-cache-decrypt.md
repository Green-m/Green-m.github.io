---
layout: post
title:  "Telegram mac 客户端本地缓存解析"
date:   2023-11-28 14:49:01 +0800
categories: Misc
---

* content
{:toc}




0x00 前言
------------------

最近接到一个需求，要求对 telegram 客户端的本地缓存进行解密和取证，研究了一下，把能解密的部分操作记录下来。

本来原始需求想研究的是 windows 平台上的 telegram 客户端，但搜索半天最后发现 windows 上的客户端使用的是 tdesktop 项目，不缓存数据，只存储凭证。而 mac 上的 telegram 会将数据缓存本地，基于 tidb 存储，所以本文是以解密和解析 tidb 结构的一种方法， 理论上来说不局限于平台，只要底层是 tidb 存储的都可以，我仅测试了 mac 的客户端。


测试使用的环境是： 10.2.4.255298 Stable(Mac telegram)，M1 Pro Venture 13.2 


0x01 获取本地的用户加密的 encryptkey
------------------

macos 的官方客户端，默认在 `~/Library/Group\ Containers/6N38VWS5BX.ru.keepcoder.Telegram/stable/`

该目录下有一个 `.tempkeyEncrypted` 文件，这个文件就是用来对本地缓存进行加密的密钥文件。

```
$ ls -la
total 32
drwxr-xr-x@ 12 green  staff   384 Nov 27 16:48 .
drwxr-xr-x@  5 green  staff   160 Aug 22  2022 ..
-rw-r--r--@  1 green  staff    64 Jul 30  2022 .tempkeyEncrypted
drwxr-xr-x@ 37 green  staff  1184 Jul 24 09:19 Wallpapers
drwxr-xr-x@  6 green  staff   192 Jan  4  2022 account-xxx
drwxr-xr-x@  5 green  staff   160 Sep  6  2022 account-xxxx
drwxr-xr-x@  6 green  staff   192 Nov 16 14:36 accounts-metadata
-rw-r--r--@  1 green  staff  4878 Nov 16 14:36 accounts-shared-data
-rw-r--r--@  1 green  staff    21 Nov 27 16:48 crashhandler
drwxr-xr-x@ 22 green  staff   704 Nov 27 16:39 logs
drwxr-xr-x@  3 green  staff    96 Jul 18  2022 temp
drwxr-xr-x@ 12 green  staff   384 Nov 27 16:40 trlottie-animations
```

我参考网上的资料，整理了一个脚本，可以解析这个 `.tempkeyEncrypted`，脚本地址： https://gist.github.com/Green-m/6e3a6d2ffbb1b669d37b756572ca232f

需要修改的地方主要是：

1. 文件路径， mac 上主要是用户名

```
/Users/yourname/Library/Group Containers/6N38VWS5BX.ru.keepcoder.Telegram/stable/.tempkeyEncrypted
```

2. 默认密码，如果 TG 客户端设置了 passcode，则需要把这个密码修改成对应的 passcode，如果没有则不用改。

```
DEFAULT_PASSWORD = 'no-matter-key'
```

如果提示hash 不匹配则说明密码错误：    

```
    raise Exception(f'hash mismatch: {dbHash} != {calcHash}')
Exception: hash mismatch: 430097447 != -1438537127
```

如果正确的话，会将 `.tempkeyEncrypted` 转成这样的格式：

```
PRAGMA key="x'427a6970d894722d0795446d098xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxcfced7ea2e6edf8bbd4cebef5e358'"
```


拿到这个 key 之后，我们就可以正确解密缓存了。


可以从脚本中看出来，加密的算法是对 password 做 sha512 的 hash，得到 AES 中的 KEY 和 IV，然后再对 tempkey 解密，如果要爆破的话反过来就行了。



0x02 使用 sqlcipher 解密 sqlite
------------------

使用 sqlcipher 工具对加密的 db_sqlite 文件进行解密， key 就是上一步获取的key。 

```
sqlcipher ~/Library/Group\ Containers/6N38VWS5BX.ru.keepcoder.Telegram/stable/account-*/postbox/db/db_sqlite

PRAGMA user_version;
PRAGMA cipher_plaintext_header_size=32;
PRAGMA cipher_default_plaintext_header_size=32;
PRAGMA key="x'*****************************'";
ATTACH DATABASE 'plaintext.db' AS plaintext KEY '';
DETACH DATABASE plaintext;
```

解密完成后在当前目录下生成文件 `plaintext.db` ， 该文件就是解密后的明文 sqlite 数据库。 但是如果此时查看该数据库，会发现数据都是碎片化的，不能直接肉眼看出来东西。


0x03 解析明文 sqlite 数据库
------------------


脚本地址：https://gist.github.com/Green-m/5f845f52af08cb53b4804ede198fc4f1

根据你的需求，修改或者添加不同的函数：

```
list_all_peers() # 列举所有的 peers，包括 channel、group、chat 等 

get_messages_from_peer(9596437714) # 根据 peer id 获取对话内容 

get_messages_from_peer_timestamp(9596437714, 1687224400) # 根据 peer id 和 时间戳获取对话内容
```

解析效果：

![/styles/images/tg_decrypt/tg1.png]({{ '/styles/images/tg_decrypt/tg1.png' | prepend: site.baseurl }})

![/styles/images/tg_decrypt/tg2.png]({{ '/styles/images/tg_decrypt/tg2.png' | prepend: site.baseurl }})



0xff 参考
------------------

https://gist.github.com/stek29/8a7ac0e673818917525ec4031d77a713


