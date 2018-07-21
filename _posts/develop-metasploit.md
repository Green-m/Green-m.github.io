---
layout: post
title:  "metasploit 开发指北"
date:   2018-06-8 18:05:01 +0800
categories: MSF
tag: ruby
---

* content
{:toc}




0x00 前言
------------------

本文将介绍如何通过ruby库`pry-byebug`来对 metasploit framework 进行开发调试。

0x01 准备工作  
------------------

metasploit 官方 wiki 就提供了非常好的文档，能解决你遇到的大部分问题。[链接](https://github.com/rapid7/metasploit-framework/wiki) 

下面列出一些比较重要的文档供参考。

- 搭建 Metasploit 开发环境，这会创建一个专门用来开发的 metasploit 版本msf5。[官方文档](https://github.com/rapid7/metasploit-framework/wiki/Setting-Up-a-Metasploit-Development-Environment)（如果你是一个英语苦手，那么王一航个人提供了一个中文翻译版，见 [中文文档](https://www.jianshu.com/p/326e1034a7e4)）。

- 社区贡献指南，请参照官方给出的代码规范进行开发，[官方链接](https://github.com/rapid7/metasploit-framework/blob/master/CONTRIBUTING.md)，[中文链接](https://www.jianshu.com/p/3c8b41f2b2f4)

- 一些其他的开发技能，包括但不局限于 git使用、ruby 语法、Metasploit 使用了解等。

如果你想参与到社区中，并给 Metasploit 提交 PR，请一定好好参考上述的文档和规范。
当然如果你仅仅是想自己随便开发一个功能或者进行本地调试，完全不必遵守如此多的限制，怎么开心怎么来。毕竟 ruby 就非常自由，do whatever you want。

0x02 编写一个 exploit 模块  
------------------


0x0A 结论
------------------

笔者最开始了解到 metasploit 的时候就感受到了它的强大，是从我真正感觉到敲敲命令就能渗透的工具，慢慢的就成为了一个 metasploit 的爱好者，当一个快乐的脚本小子。

希望本篇文章能给各位新手大佬提供一点帮助，也欢迎大家能够更多的参与到维护 metasploit 社区的工作中。

