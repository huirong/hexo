title: 使用kali破解无线网
date: 2015-03-11 16:10:55
tags: 渗透测试
---

之前也用kali破解无线网，但kali装在虚拟机上，虚拟机不支持无线网卡，即使使用小米随身wifi做适配器还是各种问题，前两天刚在硬盘上安装kali，就迫不及待的想试试了
<!-- more -->
我使用的是 aircrack-ng 来破解，无线网加密方式是WPA2，现在一般的无线网都是这种加密方式
#关于Aircrack-ng 
Aircrack-ng是一个与802.11标准的无线网络分析有关的安全软件，主要功能有：网络侦测，数据包嗅探。WEP和WPA/WPA2-PSK破解。Aircrack-ng可以工作在任何支持监听模式的无线网卡上并嗅探802.11a,802.11b,802.11g的数据。
Aircrack-ng是一个包含了多款工具的无线攻击审计套装，这里面很多工具在后面的内容中都会用到，具体见下表：
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1eq1wj6klytj30ff0cw76v.jpg)
#使用Aircrack-ng破解WPA2加密的无线网
##1.查看无线网卡 
```
ifconfig
```
![](http://ww2.sinaimg.cn/large/005CA6ZCgw1eq1wis1p6yj30k40dljuy.jpg)
wlan0即为我的无线网卡
##2.使无线网卡处于监听模式
用于嗅探的无线网卡要处于monitor监听模式（若不会，自行百度）
```
airmon-ng start wlan0
```
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1eq1zm1ixgsj30ki0dtq5c.jpg)
当看到驱动下面显示有monitor mode enabled on mon0，即已启动监听模式，监听模式下适配器名称变更为mon0
##3.探测无线网络，抓取无线数据包
激活无线网卡后就可以进行抓包了，是用airodump-ng工具实现，我这里分为两步来实现
（1）查看bssid channel
```
airodump-ng mon0
```
![](http://ww2.sinaimg.cn/large/005CA6ZCgw1eq1wkkjzb3j30p10i8jxw.jpg)
第一个就是我要破解的无线网，记住他的bssid channel号
（2）抓包并保存
```
airodump-ng --ivs --ignore-negative-one --bssid  A8:57:4E:77:D0:8C mon0
```
参数解释：
--ivs 这里的设置是通过设置过滤，不再将所有无线数据保存，而只是保存可用于破解的IVS数据报文，这样可以有效地缩减保存的数据包大小
--ignore-negative-one 没有使用这个参数的时候，一直fixed channel -1，导致后面的步骤一直出错，可能因为我的Aircrack-ng不是最新的版本，这个根据大家的实际情况来，不一定非得加上
--bssid 待破解的无线网的MAC地址
-c 无线网的工作频道
-w 后跟要保存的文件名，这里w就是“write写”的意思，所以输入自己希望保持的文件名，我的文件名为loanga。**大家一定要注意的是：** 这里我们虽然设置保存的文件名是loanga，但是生成的文件却不是loanga.ivs，而是loanga-01.ivs。
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1eq1zssu6rtj311y0lcdj0.jpg)
##4.进行DeAuth攻击
为了获得破解所需的WPA2握手验证的整个完整数据包，将会发送一种称之为“DeAuth”的数据包来将已经连接至无线路由器的合法无线客户端强制断开，此时，客户端就会自动重新连接无线路由器，大家也就有机会捕获到包含WPA2握手验证的完整数据包了
重新打开一个终端
```
aireplay-ng -0 30 -a A8:57:4E:77:D0:8C mon0
```
参数解释：
-0 采用DeAuth攻击模式，后面跟攻击次数
-a 待破解的MAC地址
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1eq1wmh2w48j30kf0don2o.jpg)
当抓包的界面出现 WPA handshake 时，即成功获取到无线WPA-PSK验证数据报文，就可以进行暴力破解了
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1eq1wm1m8j0j30jp0d7tb6.jpg)
##5.开始暴力破解
```
aircrack-ng -w /usr/share/wordlists/dictionary/wordlist1.txt loanga-01.ivs
```
参数解释：
-w 后跟预先下载的字典，我的字典是在网上下载的，大家也可以自己下载生成字典的软件，自己制作字典
如果破解不成功，大家可以换一个字典
![](http://ww2.sinaimg.cn/large/005CA6ZCjw1eqh3oj2cdcj30kc0domzg.jpg)
恭喜你，破解成功！！！！
#参考文献
 - 完全教程 Aircrack-ng破解WEP、WPA-PSK加密利器<http://netsecurity.51cto.com/art/201105/264844_all.htm>
 - <https://www.youtube.com/watch?v=zAh0yQdLXDc>

**本教程纯属学习。。。。。。**



