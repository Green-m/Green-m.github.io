---
layout: post
title:  "Metasploit 开发指北"
date:   2018-07-27 17:41:01 +0800
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

- Metasploit API，能够查询大量的类、模块、函数定义等等，[参考链接](https://rapid7.github.io/Metasploit-framework/api/)  

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

0x03 开发库
------------------

对库函数进行开发就没有开发模块那么多文档了，这个难度也会比之前的难度高一点，没有那么多可以借鉴的代码，而且经常陷入无尽查找函数定义的循环中。 

### pry-byebug 

这是一个 ruby 的库文件，在 pry 的基础上添加了命令行下的单步调试功能。

pry 本来就很好用了，而 pry-byebug 最关键的是可以下断点，我自己开发遇到不认识的类或者函数，就靠这个库来解决。[官方链接](https://github.com/deivid-rodriguez/pry-byebug)

简单介绍一下使用方法（安装过程略过）：

- 在需要下断点的地方写入 `binding.pry`

```
  # Function to install payload as a service
  #-------------------------------------------------------------------------------
  def install_as_service(script_on_target)
    if  is_system? || is_admin?
      binding.pry
      print_status("Installing as service..")

```

- 通过指令来控制代码执行流

```
step: 单步执行，类似于OD里的 F7

next: 单步执行，类似于OD里的 F8

finish: 执行到当前函数返回，类似于 Ctrl + F9

continue: 继续执行，类似于F9  

backtrace： 查看堆栈
```

还有两个比较实用的函数， `show-doc` 和 `show-method`，这两个函数可以在 pry 界面查询对应函数的文档和源码。

Metasploit 项目包含了很多别的库函数，在官方文档内查不到的话用这个方法可以很顺利的找到对应的函数，就不用走太多弯路了。

掌握这些基本的操作就可以了，想深入了解的话多阅读官方文档和代码。


### Bug fixer

虽然 Metasploit 社区非常活跃，开发者也很积极的在更新，但 Bug 怎么都是避免不了的东西，尤其是在各种平台下，每添加一个 feature 就有可能引入新的 bug。 

笔者最开始最开始接触库函数相关的代码就是先找到了一个bug，想着提个 issue 吧，官方解决 issue 的速度真的是随缘，还不是靠自己来，于是就卷起袖子开始撸。 

接下来会用一个非常简单的例子给大家讲解。（原始 PR 链接： https://github.com/rapid7/metasploit-framework/pull/10151）  

在 meterpreter session 回连成功后，如果没有设置自动加载 stdapi 扩展或者因为网络原因没有加载成功的话，使用 load 命令自动补全待加载的扩展时，meterpreter会直接崩溃。

报错如下：

```
load [-] Session manipulation failed: undefined method `config' for nil:NilClass ["/root/Desktop/msf_dev/metasploit-framework/lib/rex/post/meterpreter/ui/console/command_dispatcher/core.rb:1220:in `cmd_load_tabs'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:409:in `tab_complete_helper'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:369:in `block in tab_complete_stub'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:358:in `each'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:358:in `tab_complete_stub'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:343:in `tab_complete'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/shell.rb:62:in `block in init_tab_complete'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/readline.rb:136:in `readline_attempted_completion_function'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:6329:in `gen_completion_matches'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:6813:in `rl_complete_internal'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:6903:in `rl_complete'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:4374:in `_rl_dispatch_subseq'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:4363:in `_rl_dispatch'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:4779:in `readline_internal_charloop'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:4853:in `readline_internal'", "/usr/local/rvm/gems/ruby-2.5.1@metasploit-framework/gems/rb-readline-0.5.5/lib/rbreadline.rb:4875:in `readline'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/input/readline.rb:162:in `readline_with_output'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/input/readline.rb:100:in `pgets'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/shell.rb:375:in `get_input_line'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/shell.rb:191:in `run'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/post/meterpreter/ui/console.rb:66:in `interact'", "/root/Desktop/msf_dev/metasploit-framework/lib/msf/base/sessions/meterpreter.rb:570:in `_interact'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/interactive.rb:49:in `interact'", "lib/msf/ui/console/command_dispatcher/core.rb:1386:in `cmd_sessions'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:546:in `run_command'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:508:in `block in run_single'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:502:in `each'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/dispatcher_shell.rb:502:in `run_single'", "/root/Desktop/msf_dev/metasploit-framework/lib/rex/ui/text/shell.rb:208:in `run'", "/root/Desktop/msf_dev/metasploit-framework/lib/metasploit/framework/command/console.rb:48:in `start'", "/root/Desktop/msf_dev/metasploit-framework/lib/metasploit/framework/command/base.rb:82:in `start'", "./msfconsole:49:in `<main>'"]
```

错误比较长，但比较有用的主要是第一行，也就是最底层的堆栈调用，会直接反应出问题，仔细看这一句

```
Session manipulation failed: undefined method `config' for nil:NilClass ["/root/Desktop/msf_dev/metasploit-framework/lib/rex/post/meterpreter/ui/console/command_dispatcher/core.rb:1220:in `cmd_load_tabs'
```

查看对应文件的代码 

```
# lib/rex/post/meterpreter/ui/console/command_dispatcher/core.rb
  def cmd_load_tabs(str, words)
    tabs = SortedSet.new
    if !client.sys.config.sysinfo['BuildTuple'].blank?
      # Use API to get list of extensions from the gem
      MetasploitPayloads::Mettle.available_extensions(client.sys.config.sysinfo['BuildTuple']).each { |f|
        if !extensions.include?(f.split('.').first)
          tabs.add(f)
        end
      }
```

`client.sys.config.sysinfo` 函数是在 stdapi 扩展中实现的，因此没有成功加载 stdapi 时调用此函数肯定会失败。  

这部分代码的作者在实现的时候未考虑该情况，才导致该bug的出现，我们只需要判断一下是否加载了该扩展就修复了该bug。

修复后的代码

```
# lib/rex/post/meterpreter/ui/console/command_dispatcher/core.rb 
  def cmd_load_tabs(str, words)
    tabs = SortedSet.new
    if extensions.include?('stdapi') && !client.sys.config.sysinfo['BuildTuple'].blank?
      # Use API to get list of extensions from the gem
      MetasploitPayloads::Mettle.available_extensions(client.sys.config.sysinfo['BuildTuple']).each { |f|
        if !extensions.include?(f.split('.').first)
          tabs.add(f)
        end
      }
```

更多适合新手的 Pull Request 可以参考[链接](https://github.com/rapid7/metasploit-framework/pulls?q=is%3Apr+label%3Anewbie-friendly+is%3Aclosed)

### feature develop 

当修复比较多的bug后，对 Metasploit 了解也越来越深入，尤其是一些运行加载的库和对应的函数，这个时候想实现一些自己想要的功能就得心应手。  

其实这个部分主要不是局限于对 Metasploit 的不熟悉，更多的是对编程能力的考验。

这部分不太好介绍如何开发，就放上一个我最近给 meterpreter 添加的补全功能吧，有兴趣的多参考一下。 

PR 链接见 https://github.com/rapid7/metasploit-framework/pull/10379  


0x04 结论
------------------

笔者最开始了解到 Metasploit 的时候就感受到了它的强大，是让我真正感觉到敲敲命令就能渗透的工具，慢慢的就成为了一个 Metasploit 的爱好者，当一个快乐的脚本小子。

希望本篇文章能给各位新手大佬提供一点帮助，也欢迎大家能够更多的参与到维护 Metasploit 社区的工作中。

