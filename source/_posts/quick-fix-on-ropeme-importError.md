title: 快速修复 ropeme ImportError：No module named distorm
date: 2016-03-11 13:44:13
tags: 
- exploit 编写
- 缓冲区溢出
- ROP
categories: exploit 编写
---
首次使用 [ropeme](https://github.com/packz/ropeme) 过程中，遇到 ImportError 问题，解决方案如下。
<!--more-->
# Ⅰ、问题
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1f36pui72zij30kd03o762.jpg)
```
star@ubuntu:~/glibc/CVE-2015-7547-master$ ../ropeme-master/ropeme/ropshell.py 
Traceback (most recent call last):
  File "../ropeme-master/ropeme/ropshell.py", line 24, in <module>
    import gadgets
  File "/home/star/glibc/ropeme-master/ropeme/gadgets.py", line 21, in <module>
    import distorm
ImportError: No module named distorm
```
# Ⅱ、解决方法
- 打开 gadgets.py
- 将所有的 distorm 替换成 distorm3
- 保存并退出
- 然后重新运行 ropshell.py
 
<font color="red">Tips：</font> 前提是 Linux 中已经安装了 distorm3
安装 distorm3
```
sudo apt-get install python-distorm3
```
# Ⅲ、参考文献
[Quick fix on ROPeme's ImportError: No module named distorm](http://breaktoprotect.blogspot.hk/2014/03/quick-fix-on-ropemes-importerror-no.html)
