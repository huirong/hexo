title: 在 Kali 中安装 chrome
date: 2015-03-24 18:51:06
tags: kali
categories: kali
---

在Win下习惯了chrome，安装kali之后就想换个浏览器
我使用如下命令安装chrome时，apt-get install google-chrome-stable，一直报错，找不到源，只好到官网上下载.deb安装包
<!-- more -->
# I、下载chrome安装包
到官网上下载 [chrome安装包](https://www.google.com/intl/zh-CN/chrome/browser/desktop/index.html)
选择合适的安装包
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eqh1mga2b3j30lo0gm0uj.jpg)
# II、安装
 - 进入到.deb的目录，使用命令行安装
 ```
 dpkg -i google-chrome-stable_current_i386.deb
 ```
 可以看到 应用程序 -> 互联网 出现了久违的Google Chrome
 - 修改相关配置
 点击chrome图片之后，会提示“chrome不能作为跟用户运行”
 不要紧，so easy，添加一条语句
 ```
 cd /opt/google/chrome
 vim google-chrome
 ```
 在文件末尾的代码处添加  
 	--user-data-dir
 ![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqh1q3bqmzj30kf029aaa.jpg)
ok了，可以正常使用了
# III、参考文献
[Kali linux 安装chrome](http://blog.csdn.net/change518/article/details/18556625)

# IV、友情链接
使用chrome，不翻墙还不如不用，我用的是shadowsocks，参考如下网址[shadowsocks的安装和配置--在ubuntu和ubuntu中](http://jackroyal.github.io/2015/03/09/use-ss/)



