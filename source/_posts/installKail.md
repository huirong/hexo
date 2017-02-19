title: win8下硬盘安装Kali双系统
date: 2015-03-09 14:43:39
tags: kali
categories: kali
---

只有一个常用U盘，不想用来装系统，只好使用硬盘安装，听说比较麻烦，确实比较麻烦，出了好多问题
<!-- more -->
# I、准备工作
##  ① 下载 [kali镜像](https://www.kali.org/downloads/)
我下载的是64位版本的  <http://cdimage.kali.org/kali-1.1.0/kali-linux-1.1.0-amd64.iso>，各位根据自己的实际情况下载
## ② 解压 kali-linux-1.0.9a-amd64iso
解压 kali-linux-1.0.9a-amd64iso 到某个盘根目录。（只需解压后的文件，不再需要iso）
<font color="red">Tips：</font>解压后，里面的文件全部要放到根目录下，而不是在一个文件夹里面
## ③ 分区
windows下压缩一部分未使用的磁盘空间，到后面给kali linux使用
我用分区助手进行分区，一般20G左右就可以了，可根据实际情况选择大小，如下图
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eq0gae0r45j30hq080wfl.jpg)
## ④ 添加引导项
安装EasyBCD,添加引导项
打开 -> 条目 -> NeoGrub -> 安装 -> 配置
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1eq0gcir4w3j30g70dljum.jpg)
在弹出的配置窗口粘贴下面代码
```
title Install kali　　
root (hd0,X)! ]9 A- B0 K, ~8 ~: s- a
kernel (hd0,X)/live/vmlinuz boot=live noconfig=sudo username=root hostname=kali　
initrd (hd0,X)/live/initrd.img
boot
```
    
其中“X”替换为你的iso解压目录，想了解（hd0,X）格式，可以百度一下，这里就不再多说了。

# II、安装kali
1. 重启，进入live模式，点击左上角的appication -> System tools -> install kali,
开始安装kali
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1eq0gdi4rzzj31kw16odxp.jpg)
2. 选择语言和地区
语言：Chinese（Simplified）
地区：中国 
3. 配置网络和域名
我的配置网络就是连接现有的有线网，然后分配IP什么的
domain留空不管他，下一步
4. 设置root密码
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eq0geixwv6j31kw16ono6.jpg)
5. 磁盘分区
这里我们手动进行分区，一共分三个区，一个300M的/boot分区，一个2048M的swap分区，其他的分为一个/（你也可以把/home单独分区出来），因为分区方法类似，所以我只讲一个/boot分区的步骤
首先选择最下面的手动分区
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1eq0gfmfg0mj31kw16ob29.jpg)

**创建boot分区**
- 在选择之前分出来的空闲分区
- 创建一个300M 的分区 
创建好的分区如下：
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1eq0giglz4uj31kw16ou0x.jpg)
需要双击修改相关参数，修改后的结果如下，然后选择“分区设定结束并将修改写入磁盘”
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eq0gk0vtgnj31kw16ox6p.jpg)

**用同样的方法创建swap分区和其他分区**
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1eq0glb9zghj31kw16oqv5.jpg)
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1eq0glpnt2lj31kw16ou0x.jpg)
区分结束后，结果如下：
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1eq0gmllt1yj31kw16o1ky.jpg)
6. 坐等安装系统
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1eq0gnfi171j31kw16o1kx.jpg)
一段时间后，终于见到了熟悉的画面
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eq0go0rny6j31kw16odzj.jpg)

