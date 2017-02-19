title: 在Ubuntu上安装LAMP服务器并简单配置
date: 2015-04-01 20:05:07
tags: 
- Ubuntu
- LAMP
categories: LAMP
---
我在DigitalOcean上有个Linux服务器，想把下载的项目挂在上面，就得安装一个LAMP服务器，一个个安装Appach、MySql、PHP太麻烦了，就安装个集成的。
<!-- more -->
# I、在Ubuntu上安装LAMP
```
apt-get install lamp-server^
```

最后那个符号^不要少，不然不能正确执行

![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqrk4ytavkj30ir0btwf0.jpg)

在安装的过程中，系统会提示你为MySQL的根用户输入密码
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqrk5ocd04j30ir0btjtb.jpg)
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eqrk5wgf77j30ir0bt0ts.jpg)
密码设置成功后，会继续安装一部分文件，等待一会，就OK了

# II、修改 Apache 根目录
Ubuntu安装Apache后，默认的根目录是 /var/www/html，让我很不习惯，处女座的毛病犯了，就想修改配置文件
Apache有多个配置文件，设置根目录的配置文件，位于 /etc/apache2/sites-available下的 000-default.conf
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eqrk8zquefj30i6038q3w.jpg)

将 DocumentRoot的值 由 /var/www/html  改为 /var/www/
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqrk7ntvbgj30ir0btq6u.jpg)

<font color="red">最后将/var/www/html/下的index.php文件移动到 /var/www/，这样才能正常显示主页了,因为根目录已经变了</font>

# III、测试
## ① 测试Apache
打开浏览器，输入网址http://localhost/或者http://YOUR_IP，会看到如下提示
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqrkadel6gj30nx08btcs.jpg)
OK，Apache正常！！！！
## ② 测试php
在/var/www中创建一个testing.php文件，内容可以自定义，我使用的终端命令：
```
echo "<?php phpinfo(); ?>" | sudo tee /var/www/testing.php
```
然后重启Apache服务 
```
service apache2 restart
```
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqrk655v8wj30il04bwfm.jpg)
在浏览器中，输入网址http://localhost/testing.php或http://YOUR_IP/testing.php   可以看到安装的PHP信息
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqrk9sw2r9j30jc097myl.jpg)
## ③ 配置MySQL
需要将MySQL绑定到本地主机IP地址，默认情况下，这个地址应该是 127.0.0.1，为了以防万一，还是确认一下
```
cat /etc/hosts | grep localhost
```
会看到如下内容
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqrkar0vq0j30b501fjri.jpg)
然后确定MySQL的conf文件
```
cat /etc/mysql/my.cnf | grep bind-address
```
会看到如下内容
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqrkb4yn7qj30du00ydfz.jpg)
MySQL配置OK了。。。
# IV、安装phpMyAdmin
phpMyAdmin图形化的界面，帮助我们方便的管理数据库。
```
apt-get install libapache2-mod-auth-mysql phpmyadmin
```
接下来，系统会提示你选择为phpMyAdmin配置的Web服务器，使用键盘上的箭头键。高亮apache2，然后回车
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqrkbcj30zj30ir0bttaa.jpg)
询问是否为phpMyAdmin配置一个名为dbconfig-common的数据库，选择 yes，回车 
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1eqrkbjqo3vj30ir0btn0o.jpg)
然后输入之前设置的数据库密码，并确认密码
# V、配置Apache服务器
如果此时在浏览器中输入http://localhost/phpmyadmin，会提示404错误
此时应该进行简单的配置，将phpMyAdmin的配置文件，复制到Apache2下
```
cp /etc/phpmyadmin/apache.conf /etc/apache2/conf-enabled/phpmyadmin.conf
```
然后重启Apache服务器
```
service apache2 restart
```
现在可以正常访问http://localhost/phpmyadmin 页面了
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1eqrkbxvqrnj30hm0fk0tk.jpg)
输入 root和之前设置的数据库密码进行登录
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1eqrkc938j9j30o20ckwgf.jpg)

恭喜你，成功完成Ubuntu下安装和配置LAMP和phpMyAdmin了，现在可以在上面搭建自己的网站了。。。。。。

# Ⅵ、参考文献
 - [如何在Ubuntu上安装LAMP服务器系统？](http://os.51cto.com/art/201307/405333_all.htm)
 - [Ubuntu Apache 根目录的更改方法](http://blog.csdn.net/fengguowuhen7871/article/details/8843241)
 - [I cannot access phpMyAdmin on Ubuntu 14.04](https://www.digitalocean.com/community/questions/i-cannot-access-phpmyadmin-on-ubuntu-14-04)
