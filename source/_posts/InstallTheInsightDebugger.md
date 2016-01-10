title: Linux下安装Insight Debugger
date: 2015-05-15 14:32:44
tags: 调试器
---
之前一直在用gdb调试器，查看内存、寄存器的值都觉得不方便，Insight Debugger相当与图形化的gdb
<!-- more -->
#添加源
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1es4yacbwwrj30i301u74j.jpg)
在 /etc/apt/sources.list 中添加源：
```
deb http://ppa.launchpad.net/sevenmachines/dev/ubuntu natty main 
deb-src http://ppa.launchpad.net/sevenmachines/dev/ubuntu natty main
```

更新源
```
 sudo apt-get update
```
#安装Insight Debugger
在终端输入：
```
sudo apt-get install insight
```
#验证安装
现在感受下Insight Debugger
启动：

![](http://ww1.sinaimg.cn/large/005CA6ZCjw1es4yarfwcej30db00jt8j.jpg)

图像化界面：
![](http://ww3.sinaimg.cn/large/005CA6ZCjw1es4yazd31yj30jg0f4djx.jpg)
OK！！ Insight Debugger已经安装成功了，大家慢慢享受其中的奥妙

#参考文献
[Install the Insight Debugger on Linux Mint (works for Ubuntu too)](http://baptiste-wicht.com/posts/2012/01/install-insight-debugger-linux-mint-ubuntu.html#)
