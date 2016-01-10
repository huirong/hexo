title: Smash the Stack Wargaming Network介绍
date: 2015-05-18 10:07:01
tags: 缓冲区溢出
---
最近在学习缓冲区溢出，听说这个靶练场不错，就去学习学习。。。。。
<!-- more -->
#Smash the Stack简介
Smash the Stack Wargaming Network提供一系列的博弈赛。在本网站中，一场博弈赛相当与一个合法的黑客环境，再现了现实世界中软件漏洞的理论或概念，并允许我们合法的使用渗透测试技术。该网站提到的软件，可以是操作系统、网络协议或任何用户应用程序。
#如何进入博弈赛
为了进入博弈赛，你需要一个ssh客户端（openssh,PuTTy,SecureCRT）。每一关都有其独特的连接细节，你需要注意<font color="orange">端口</font>和<font color="orange">初始用户名</font>。
如果你使用Linux系统，只需在终端输入：
```
 user@box:$ ssh -l level1 io.smashthestack.org -p2224
```
 紧接着就要输入密码，第一关的密码是 level1
 这样就可以成功进入第一关
#如何获得下一关的密码
 你只有找到本关卡的漏洞，并成功利用，才能获得下一关的密码。每一关的密码放在不同的位置。但是它位于以下几个位置中的一个：
    ~/.pass
    ~/passwd
    /pass/

查看密码使用一下命令：
user@box:$ cat ~/.pass
user@box:$ cat /pass/level1
user@box:$ cat ~/passwd

本文章，我只是个翻译员，不过接下来的关卡，都是自己的亲身实践。。。。。。
#参考文献
[Smash the Stack Wargaming Network](http://smashthestack.org/faq.html#a3)
