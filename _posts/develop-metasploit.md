---
layout: post
title:  "Metasploit 开发指北"
date:   2018-06-8 18:05:01 +0800
categories: MSF
tag: ruby
---

* content
{:toc}




0x00 前言
------------------

本文将通过一些简单的例子说明如何开始学习对 Metasploit Framework 进行开发。

0x01 准备工作  
------------------

Metasploit 官方 wiki 就提供了非常好的文档，能解决你遇到的大部分问题。[链接](https://github.com/rapid7/Metasploit-framework/wiki) 

下面列出一些比较重要的文档供参考。

- 搭建 Metasploit 开发环境，这会创建一个专门用来开发的 Metasploit 版本msf5。[官方文档](https://github.com/rapid7/Metasploit-framework/wiki/Setting-Up-a-Metasploit-Development-Environment)（如果你是一个英语苦手，那么王一航个人提供了一个中文翻译版，见 [中文文档](https://www.jianshu.com/p/326e1034a7e4)）。

- 社区贡献指南，请参照官方给出的代码规范进行开发，[官方链接](https://github.com/rapid7/Metasploit-framework/blob/master/CONTRIBUTING.md)，[中文链接](https://www.jianshu.com/p/3c8b41f2b2f4)

- Metasploit API，能够查询大量的类、模块、函数定义等等，[链接](https://rapid7.github.io/Metasploit-framework/api/)  

- 一些其他的开发技能，包括但不局限于 git使用、ruby 语法、Metasploit 使用了解等。

如果你想参与到社区中，并给 Metasploit 提交 PR，请一定好好参考上述的文档和规范。
当然如果你仅仅是想自己随便开发一个功能或者进行本地调试，完全不必遵守如此多的限制，怎么开心怎么来。毕竟 ruby 就非常自由，do whatever you want。

0x02 编写模块  
------------------

Metaploit 官方的 wiki 里对于如何开发 auxiliary, exploit, post 甚至是 meterpreter 脚本都有非常详细的文档，参照这些文档就能够让你对如何开发这些模块有相当的了解(即使有少部分已经有点过时)。

这里笔者就不详细的一一翻译文档了，主要介绍一些自己开发过程中踩过的坑和需要额外注意的地方。  


### 模仿其他模块  

当你看完了上面所有的文档和 wiki，如果还是对开发一个全新的模块毫无思绪的话，那么模仿其他人的模块是最好的方法。

在同类型的模块下面，你总能找到和你想开发的类似的模块，仔细阅读一遍，最理想的情况可以直接复制过来，然后修改`check` 和 `exploit` 函数就大功告成。  

### 查询 API 文档 

Metasploit 项目比较庞大，而且由于 ruby 的灵活和抽象，整体代码封装得相当的好，这导致阅读代码的时候经常找不着北，不清楚函数具体的定义，这时就需要查询 API 文档。

这里以我开发 Hadoop 未授权漏洞代码为示例进行说明。完整源代码见 https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/http/hadoop_unauth_exec.rb  

这个模块的`exploit` 函数只有两句:

```
# File 'modules/exploits/linux/http/hadoop_unauth_exec.rb', line 62
  def exploit
    print_status('Sending Command')
    execute_cmdstager
  end
```

主要是由 `execute_cmdstager` 函数触发漏洞利用代码，但代码上下文中根本找不到该函数的定义，在文档中查询该函数源码([链接](https://rapid7.github.io/metasploit-framework/api/Msf/Exploit/CmdStager.html#execute_cmdstager-instance_method))：

```
def execute_cmdstager(opts = {})
  self.cmd_list = generate_cmdstager(opts)

  stager_instance.setup(self)

  begin
    execute_cmdstager_begin(opts)

    sent = 0
    total_bytes = 0
    cmd_list.each { |cmd| total_bytes += cmd.length }

    delay = opts[:delay]
    delay ||= 0.25

    cmd_list.each do |cmd|
      execute_command(cmd, opts)
      sent += cmd.length

      # In cases where a server has multiple threads, we want to be sure that
      # commands we execute happen in the correct (serial) order.
      ::IO.select(nil, nil, nil, delay)

      progress(total_bytes, sent)
    end

    execute_cmdstager_end(opts)
  ensure
    stager_instance.teardown(self)
  end
end
```

这里其他的细节不用管，主要可以看到执行了两个函数，`generate_cmdstager` 和 `execute_command`，分别是用来生成 payload 和执行漏洞利用代码的，前者是 MSF 封装好的函数，后者需要自己在代码中实现。（有兴趣可以继续深入源码学习 `generate_cmdstager` 是如何实现的，笔者这里限于篇幅就不过多介绍了。）

0x03 编写库
------------------


0x0A 结论
------------------

笔者最开始了解到 Metasploit 的时候就感受到了它的强大，是让我真正感觉到敲敲命令就能渗透的工具，慢慢的就成为了一个 Metasploit 的爱好者，当一个快乐的脚本小子。

希望本篇文章能给各位新手大佬提供一点帮助，也欢迎大家能够更多的参与到维护 Metasploit 社区的工作中。
