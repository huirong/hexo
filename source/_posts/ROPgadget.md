title: ROPgadget安装
date: 2015-06-12 15:04:02
tags: 
- 缓冲区溢出 
- ROP
categories: ROP
---
ROPgadget是一个自动化的搜索工具，找到指定二进制文件中的gadgets，来帮助我们实现ROP攻击。支持x86、x64、ARM、ARM64、PowerPC、SPARC和MIPS架构。
<!-- more -->
下载网址：<https://github.com/JonathanSalwan/ROPgadget>
# I、安装
## ① 安装Capstone
Capstone是一个轻量级的多平台架构支持的反汇编架构，支持包括ARM\ARM64、MIPC和x64/x86平台。
```
sudo apt-get install python-capstone
```
## ② 下载ROPgadget
下载ROPgadget并解压，就可以使用ROPgadget了。
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1et1cc51hfnj30oo0240tj.jpg)
也可以上诉命令使用别名
```
alias ropgadget="/home/star/ROP/ROPgadget-master/ROPgadget.py"
```
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1et1ccc944xj30p3014q3k.jpg)
# II、参考文献
[ROPgadget - Gadgets finder and auto-roper](http://shell-storm.org/project/ROPgadget/)
<https://github.com/JonathanSalwan/ROPgadget>

