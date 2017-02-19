title: Immunity Debugger 插件 mona 使用
date: 2015-12-18 19:45:32
tags: 
- 缓冲区溢出
- 调试器
categories: 调试器
---
本人最近写学shellcode编写，mona插件不可或缺，在此介绍mona插件用法
<!-- more -->

# Ⅰ、<font color="blue">查找 pop pop ret</font>
用Immunity Debugger附加上待调试的程序，这样的地址更具有通用性，不依赖操作系统
在命令行输入
```
!mona rop
```
生成结果在 C:\D\mona-master\output\c ，log data中有显示

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ez422rhcvwj30ho05gt9v.jpg)

以下是rop.txt中的内容

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1ez424wnmi4j30vb048myt.jpg)

# Ⅱ、<font color="blue">查找JMP ESP,CALL ESP, push esp;ret</font>
命令：

```
!mona jmp -r esp
```
结果：

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1ez4293rxj5j30i70by782.jpg)

# Ⅲ、<font color="blue">计算SEH溢出长度</font>
## ① 生成溢出字符串
命令：

```
!mona pattern_create 5000
```
5000为字符串长度
结果：

![](http://ww2.sinaimg.cn/large/005CA6ZCjw1ez42ed5d1jj30k404ugne.jpg)

使用该字符串尝试溢出，得到seh的值为:35744134
## ② 计算溢出长度
首先查找查找nseh偏移，命令
```
!mona pattern_offset 35744134
```
![](http://ww1.sinaimg.cn/large/005CA6ZCjw1ez42k4ke5jj30ef04wwfs.jpg)
nseh的偏移为：584，则seh handler的偏移：584 + 4 =588

# Ⅳ、<font color="blue">汇编指令转机器码</font>
命令：
```
!mona assemble/asm –s 汇编指令（多个指令用#分隔）
```
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ez42uxxyxfj30c303yjrw.jpg)

# V、参考文献
[Immunity Debugger-mona插件使用](http://www.hack80.com/thread-21042-1-1.html)
