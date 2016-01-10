title: ARP欺骗原理
date: 2015-03-25 14:09:50
tags: 渗透测试
---
详细讲解ARP欺骗的原理，实验在下篇博客
<!-- more -->
#ARP
地址解析协议，即ARP（Address Resolution Protocol），是根据IP地址获取物理地址的一个TCP/IP协议。主机发送信息时将包含目标IP地址的ARP请求广播到网络上的所有主机，并接收返回消息，以此确定目标的物理地址；收到返回消息后将该IP地址和物理地址存入本机ARP缓存中并保留一定时间，下次请求时直接查询ARP缓存以节约资源。
#ARP欺骗
地址解析协议是建立在网络中各个主机互相信任的基础上的，网络上的主机可以自主发送ARP应答消息，其他主机收到应答报文时不会检测该报文的真实性就会将其记入本机ARP缓存；由此攻击者就可以向某一主机发送伪ARP应答报文，使其发送的信息无法到达预期的主机或到达错误的主机，这就构成了一个ARP欺骗。
局域网的网络流通不是根据IP地址进行，而是根据MAC地址进行传输。
#ARP欺骗工作原理
假设一个网络环境中，网内有2台主机，分别为主机A、B，网关。主机详细信息如下描述：
网关的地址为：IP：192.168.1.1 MAC: 01-01-01-01-01-01
A的地址为：IP：192.168.1.2 MAC: 02-02-02-02-02-02
B的地址为：IP：192.168.1.3 MAC: 03-03-03-03-03-03
正常情况下A和网关之间进行通讯。
1. B向A发送一个自己伪造的ARP应答，而这个应答中的数据为发送方IP地址是192.168.1.1（网关的IP地址），MAC地址是03-03-03-03-03-03（网关的MAC地址本来应该是01-01-01-01-01-01，这里被伪造了）
2. 当A接收到B伪造的ARP应答，就会更新本地的ARP缓存（A被欺骗了），这时B就伪装成网关了。
3. B同样向网关发送一个ARP应答，应答包中发送方IP地址四192.168.10.2（A的IP地址），MAC地址是03-03-03-03-03-03（A的MAC地址本来应该是 02-02-02-02-02-02，这里被伪造了）。
4. 当网关收到B伪造的ARP应答，也会更新本地ARP缓存（网关也被欺骗了），这时B就伪装成了A。
5. 样主机A和网关都被主机B欺骗，A和网关之间通讯的数据都经过了B。主机B完全可以知道他们之间说的什么：）。
![](http://ww3.sinaimg.cn/large/005CA6ZCjw1eqhy7qjsuzj30fb09r0t9.jpg)
就是，A向网关发送数据包，先会发给B，在由B转发给网关
网关向A发送数据包，也先会发给B，在由B转发给A
#参考文献
<http://baike.baidu.com/subview/32698/16532303.htm>
<http://baike.baidu.com/view/155386.htm>