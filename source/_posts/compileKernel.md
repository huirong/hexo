title: Linux编译内核步骤
date: 2015-04-21 15:58:08
tags: Linux
categories: Linux
---

题记：如果你想了解关于编译内核的基础知识：内核的定义、内核编译、内核编译的目的、内核版本的选择，请参考我的另一篇博客 [编译内核基础知识](http://localhost:4000/2015/04/22/Kernel/)
<!-- more -->
# I、准备工作
由于系统中没有图形界面配置工具ncurses，因此首先下载此工具安装包，然后在终端打开工具所在目录，切换到root用户下，输入以下命令：
```
tar zxvf ncurses-5.9.tar.gz
cd ncurses-5.9
./configure
make
make install
```
这样就安装好了ncurses，可以使用了
# II、下载内核
 - 到官网下载内核版本 <http://www.kernel.org>，我下载的是 linux-3.14.39
 - 打开终端，切换到root用户，输入 su ，然后输入密码即可
 - 将下载的linux-3.14.39.tar.xz 移动到/usr/src/ 目录下
 ```
 mv linux-3.14.39.tar.xz /usr/src
 ```
 - 进入/usr/src/目录下，解压缩内核压缩包
```
xz -d linux-3.14.39.tar.xz
tar -xvf linux-3.14.39.tar
```

# III、编译内核
## ① 清理内核中的残渣
```
cd /usr/src/linux-3.14.39
make mrproper
```
## ② 配置内核
```
make menuconfig
```
出现图形化界面
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1ereo9ppwjcj30ke0dmwg9.jpg)
说一下配置：
对每一个配置选项，用户有三种选择，它们分别代表的含义如下：
<*>或[*]——将该功能编译进内核
[]——不将该功能编译进内核
[M]——将该功能编译成可以在需要时动态插入到内核中的代码

使用空格键进行切换
配置完之后，保存退出
## ③ 配置完之后，开始编译内核
```
make
```
这一步需要很长时间，请耐心等待。。。。。。。。
## ④ 编译内核模块
```
make modules_install
make install
```

# IV、修改启动程序配置
1. 将生成的bzImage文件和System.map文件拷贝到/boot/目录下
 ```
cp /usr/src/linux-3.12.6/arch/x86/boot/bzImage /boot/
cp /usr/src/linux-3.12.6/System.map /boot/
 ```
2. 查看启动项
 - 更新系统引导配置，不同系统命令不一样，大家自行google
 - 查看配置文件
 配置文件在/boot/grub2/grub.cfg
 gedit /boot/grub2/grub.cfg
 看到配置文件中有如下内容，说明内核已经添加到启动项了
 ![](https://ww2.sinaimg.cn/large/005CA6ZCjw1ereo9y7ugkj30i70es760.jpg)

 OK，编译内核已经全部完成，可以重启电脑了。。。。。
 如果没有必要不要编译内核，需要很久啊O(∩_∩)O

# V、参考文献
[linux内核编译步骤（详细全过程）](http://mzqthu.iteye.com/blog/2001167)

 
