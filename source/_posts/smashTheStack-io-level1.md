title: SmashTheStack IO level1
date: 2015-05-18 10:58:31
tags: 
- 缓冲区溢出
- Smash the Stack
categories: Smash the Stack
---
现在正式开始练习，进入第一关
<!-- more -->
# I、进入系统
1、在终端输入：
```
ssh -l level1 io.smashthestack.org -p2224
```
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1es88zxmuwxj30k305pwfv.jpg)

2、输入密码
第一关的密码是level1 （只有第一关的密码才知道，以后每一关的密码需要自己取得）
看到以下信息，说明已经进来了，每一关的权限有限，这个可以自己测试一下就知道了。
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1es890ccegkj30k00cvq7i.jpg)

# II、熟悉环境
## ① 首先阅读README文件
我是不会给你翻译的，自己慢慢阅读
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1es891dmcsaj30k10220tk.jpg)
## ② 查看levels
每一关都有一个可执行文件或对应的C程序，可通过gdb调试可执行文件，所有这些文件存放在 /levels 目录中
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1es891i9mhjj30k00a479u.jpg)
# III、找漏洞
## ① 执行level01
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1es891o9ucfj30k0012t8w.jpg)
随便输入三个数字，查看效果
![](http://ww3.sinaimg.cn/large/005CA6ZCjw1es891rv5uij30k001hglw.jpg)
没有任何反应
## ② gdb调试
如果不会gdb的，可以查看我的博客：[GDB常用命令]()，当然网上也有很多教程
```
gdb level01
```
我试过了，不能使用list命令查看源代码，那就老老实实看汇编代码吧
```
disassembel main
```
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1es891xfak8j30k10ak0uc.jpg)
## ③ 分析汇编指令
第一关，应该不会太难的，果真如此，就连我这个汇编菜鸟都能看懂
输入一个数，存入eax寄存器；与0x10f进行比较，如果相等，跳转到YouWin，否则退出
那么结果显而易见，应该输入0x10f，不过要转化为十进制：271
## ④ 测试
重新运行level01
```
./level01
```
输入 271
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1es89254g8dj30k102174y.jpg)
OK，成功破解第一关，是不是很easy
# IV、查看下一关密码
按照上面的提示，密码放在/home/level2/.pass 中
那还等什么，赶紧查看密码吧！！！！
```
cat /home/level2/.pass
```
密码get：3ywr07ZFw5IsdKzU
![](http://ww1.sinaimg.cn/large/005CA6ZCjw1es892c8l9wj30jz01imx9.jpg)

由此密码可以顺利进入下一关。。。。。。。

