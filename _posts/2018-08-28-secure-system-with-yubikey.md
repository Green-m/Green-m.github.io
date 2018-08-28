---
layout: post
title:  "使用 Yubikey 加固你的系统"
date:   2018-08-28 16:20:01 +0800
categories: PGP
tag: yubikey
---

* content
{:toc}


0x00 前言
------------------
最开始了解到 yubikey 是因为和一个朋友聊到 PGP 的问题，我觉得 PGP 保存私钥很麻烦，换一个环境或者电脑被搬走的话还是存在一些风险的，放云上就更加不用说了。然后他就说你可以考虑一下 yubikey。 

其实我之前也听某个群里的大佬说到过 yubikey，初听时就觉得这个东西应该很不错，安全级别应该很高，因为那会还没有开始使用 PGP，只是把 yubikey 当作一个开机的智能卡，所以也就没有深入去了解这个东西了。

现在重新了解一下 yubikey 发现这个玩意儿功能真的齐全，而且很难得的是兼容性很好，几乎兼容市面上所有主流操作系统，商业公司值得信赖。

下面就介绍一下 yubikey 在我这里的使用场景，以及踩的一些坑，希望大家看了能有一些收获。

0x01 选购 yubikey
------------------

Yubikey 4 (Nano) 这是最常见的，也是最受欢迎的型号，这个型号提供了几乎所有的功能，除了NFC，Nano版本和普通版本的区别在于Nano版本比较小，适合一直插在电脑USB口上；而普通版本可以穿在钥匙扣上。

Yubikey 4C (Nano) 和上面的版本功能一模一样，除了USB口是 Type C

YubiKey NEO  有NFC功能，但是NFC不支持 IOS。而且有一个比较坑的地方，NEO 和普通版本的 Yubikey 4 相比，密钥长度最高只支持到 2048，这跟我已经有的PGP密钥已经不匹配了。

详细的各版本功能比较可见官网，[点击链接](https://www.yubico.com/products/yubikey-hardware/compare-yubikeys/)

最后还是选择了大众版本的 Yubikey 4。

购买途径的话，淘宝上有不少，理论上来说随便选一家价格差不多的也没差，毕竟这个东西造假成本非常高。

另外如果人在国外的话，可以考虑使用Github 的学生优惠去官网买，能便宜不少，寄回国内的话邮费就好像挺贵的。


0x02 PGP smartcard
------------------

作为我的主要使用场景，这部分会详细的介绍将 Yubikey 作为PGP 智能卡使用的流程。

注意：强烈建议用系统自带的GnuPG生成密钥，不要使用 Yubikey 自带的生成密钥的功能，自带的生成了之后私钥就永远都导不出来了，一旦丢失之前的密钥对全部失效。

这里就以我 Kali 下的环境作为示范，介绍如何生成及导入密钥，其他的系统环境请参考 (Yubikey 官方英文文档)[https://support.yubico.com/support/solutions/articles/15000006420-using-your-yubikey-with-openpgp]

### 生成主密钥

安装gpg，一般系统都自带了

`apt-get install gnupg`

生成密钥
`gpg --gen-key`

然后会提示你选择密钥的种类，选择第一种 RSA and RSA (default)

然后选择密钥长度，越长越难爆破，我选择 4096

然后选择过期时间，默认是永不过期，妥善保管好的话永不过期也没啥问题。

然后输入y确认信息。 

然后是输入个人信息，包括姓名和电子邮件地址，电子邮件地址蛮重要的，很多地方会校验这个。

确认后会要求你输入一个密码来保护私钥，这个密码就是你私钥的最后保护了，不要输入弱口令，尽量强一点。

等待一段时间，就会生成一个密钥。

这里主密钥和公钥就生成完成了，已经能满足基本使用需求了，但是为了能够能充分的使用 Yubikey ，我们还可以添加其他的两个子密钥，用来认证和签名，认证的密钥可以用来对 SSH 登陆进行认证，建议添加。


### 添加子密钥

编辑你的密钥，1234ABC是刚才生成的密钥ID，用`gpg --list-keys` 就可以查看。

`gpg --expert --edit-key 1234ABC`

然后输入
`addkey`

```
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection?

```

输入密码后，提示选择类型，输入8 选择RSA

```
Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection?
```
添加认证类型，默认情况是只有 Sign Encrypt ，我们通过反复选择 S E A来更改这个密钥的性质，成为你所需要的密钥，
最后选择Q结束。
然后是一些过期时间之类的选择，和之前一样，结束后save一下就可以了。

### 导出与导入

将本地的密钥导出到硬盘，用作备份，以防 Yubikey 丢失。

私钥
`gpg --export-secret-key --armor 1234ABC > public.key`

公钥
`gpg --export --armor 1234ABC > secret.key`

导出后建议加密保存在本地。

**将私钥装载到 Yubikey** 

插入yubikey，可以先查看一下状态. 

`gpg --card-status`

可以在bashrc或者zshrc下取个别名，以后经常会用到这条命令的。

`alias cs='gpg --card-status' `

插入 Yubikey 

`gpg --expert --edit-key 1234ABC`

会显示出你的主密钥和子密钥。

切换一下
`gpg> toggle`

选择签名密钥，然后导入密钥：
```
gpg> key 1
gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

```
选择加密密钥，然后导入:
```
gpg> key 1
gpg> key 2
gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2

```

最后保存
```
gpg> save
```
就一切成功了，最后`quit`退出即可，可以用`gpg --card-status`看一下状态。

在这之后，你的本地保存的私钥就不是真正的私钥了，虽然看起来还是，没多大区别，但实际上保存的只是一个虚拟地址，这个地址指向你的 Yubikey 保存密钥的位置。 

如果你像我一样有多台电脑，家庭和单位都需要，你可以将现在的密钥导出（而不是真正的私钥），方法和之前一样，然后再导入到其他电脑上，这样多台电脑都可以使用yubikey来进行PGP认证了。

这就是基本的导入教程，可能没有其他教程详细，网上这类教程很多，有的就没有讲得太细。

### vmware虚拟机问题

由于我的 Kali是在vmware虚拟机环境下，直接插入 Yubikey 是不识别的，需要修改对应虚拟机的相关vmx 文件。

打开对应虚拟机的vmx配置文件，在末尾添加，
```
usb.generic.allowHID = "TRUE"
usb.generic.allowLastHID = "TRUE"
```
然后重启 vmware，选择对应的可移动设备连接就可以了。如果出现了两个类似的，都有yubikey开头，优先选择不带shared的，如果无效的话，就选择带shared的尝试。

我的工作电脑就只能用共享的yubikey进行连接，还好使用起来没有什么问题。


0x03  系统锁屏解锁 
------------------

用 Yubikey 来解锁屏幕也是逼格满满的，厂商的兼容性真的是做的相当的好，几乎支持了全部主流PC系统，具体可以见[官方连接](https://www.yubico.com/why-yubico/for-business/computer-login/)

目前我的宿主机是windows，就介绍一下 windows 上登陆的使用，

下面简单介绍一下流程和注意事项，如果有一些其他的问题，建议还是以[官方文档](https://www.yubico.com/wp-content/uploads/2016/06/Windows-Login-YubiKey-Configuration_en.pdf)为准。

由于 windows 10 新加入的 windows hello，Yubikey 也集成了这个功能，可以和hello 搭配使用，使得操作更加简单方便，可以直接在微软商店中下载，但我的系统是win7不支持这种方式，下面就介绍win7下安装的要点。

### 准备工作

- 官方推荐需要两个yubikey，一个用来备份，一个用来平时使用，这种方式更安全，一个的话也能用。

- 下载 YubiKey Personalization Tool  [下载链接](https://www.yubico.com/support/downloads/)

- 本地管理员权限账户

- 下载 YubiKey Windows Login software [下载链接](https://www.yubico.com/support/downloads/)

- 已安装Microsoft .NET Framework 4.0 

在安装过程中会需要重启。

注意：如果没有两个 yubikey 的话，加密性没有那么强，如果不慎丢失，可以进安全模式禁用该功能就好了。 
如果对系统加密性要求比较高的同学，可以考虑购入两个 Yubikey。

### 安装和配置  

按照文档安装上述两个工具就好了，一个工具是安装本地登陆验证流程相关的驱动，另外一个主要是配置Yubikey使用的。

一般中间不会出什么问题，按照示意图进行配置就好，由于软件界面也都是英文的，直接看英文文档反而更加清晰，就不翻译了。

需要注意的就是，配置好以后记得备份一下配置文件，尤其是使用两个Yubikey，选择安全模式下也加密的情况。

重启后就可以开心的使用了，每次锁屏后都需要插入yubikey才能成功登陆。

如果有机会的话我会更新其他系统安装的场景。


0x04  SSH登陆
------------------

我们可以直接使用GPG套件内的 gpg-agent 来对SSH连接进行认证，非常方便。

当然这里就体现出来咱们前面为啥要生成多个子密钥的作用了，如果没有生成认证用的子密钥的话，光靠主密钥是不能对SSH进行认证的。

下面介绍一下通过配置自动使用SSH连接时，优先使用 gpg认证密钥来认证的方法。

### zsh下的配置(Kali下测试)  

zsh 下可以直接使用插件 gpg-agent ，然后输入就可以完成 

`start_agent_withssh`

如果一直出现如下报错
`gpg-agent: a gpg-agent is already running - not starting a new on`

将如下命令添加到 `~/.zshrc` 即可
```
# Set SSH to use gpg-agent
unset SSH_AGENT_PID
if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"
fi
```
然后 `source ~/.zshrc` 就可解决错误。

### Mac 下的 zsh配置

Macos 下和 Kali 下的配置不同，直接使用该插件会失败，配置方法如下：

安装`pinentry-mac`

`brew install pinentry-mac`

在文件`~/.gnugp/gpg-agent.conf`中添加
```
pinentry-program /usr/local/bin/pinentry-mac
enable-ssh-support
```

然后在 `~/.zshrc`中添加 
```
export "GPG_TTY=$(tty)"
export "SSH_AUTH_SOCK=${HOME}/.gnupg/S.gpg-agent.ssh"
```

最后重启gpg-agent
```
gpg-connect-agent killagent /bye
gig-connect-agent /bye
```

无论是啥环境和shell，修改后都要记得 source 一下使之生效。

### 将公钥导出  

用下列命令导出本地 SSH 公钥
`ssh-add -L`

如果有公钥出现，而且结尾字符串中是这样的形式 `cardno:000607848047`，那说明就成功了。

接下来就把刚才的 SSH 公钥，然后添加到对应主机的文件 `~/.ssh/authorized_keys` 中，以后就可以插上yubikey 直接使用 ssh 连接了。

0x05 其他使用
------------------

其他的使用场景还有很多，如

- Github, Google等等账户的二次认证设备
- 静态密码生成器

这里就不介绍了，有兴趣的话搜一下相关资料。

0x06 总结
------------------

从开始使用 PGP 做认证开始，就越来越觉得密钥的保管是个大问题，尤其是 Metasploit 项目要求部分人员使用PGP进行签名，有时候需要在家里或者公司，如果都保存的话感觉泄露的风险就更大。

从这个角度来讲，Yubikey 完美解决了我的保管密钥的痛点，感觉很棒。

如果你也有和我类似的困扰，欢迎尝试yubikey。