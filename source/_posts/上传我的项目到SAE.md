title: 上传我的项目到SAE
date: 2014-06-06 13:41
tags:
- SAE
categories:
- SAE
---

# I、创建新应用
登录SAE，可通过微博账号登录，进入首页，点击创建新应用
![](https://img.blog.csdn.net/20140605200206765)
新建成功后，回到首页，就可以看到新建的应用了
![](https://img.blog.csdn.net/20140605201219765)

# II、上传代码
1. 将代码按照项目结构打包，不要放在一个文件夹里，即解压后就能看到项目结构，而不是一个文件夹
提示：只支持上传zip、 gz、tar.gz三种代码包，文件大小不能超过20MB
如果压缩包中包含有中文文件名的文件，请使用utf8编码，否则会上传失败。最好不要有中文文件名
2. 点击新建的应用，即wustSurvey
![](https://img.blog.csdn.net/20140605202206031)
2. 首次进入，系统会提示新建一个版本，选择新建
右边会出现版本1，点击操作->上传代码，将刚打包好的代码上传
![](file:///C:\Users\LukyStar\AppData\Roaming\Tencent\Users\837410145\QQ\WinTemp\RichOle\WFTZA3B99M}[1%K1YEVGGQE.jpg)
![](https://img.blog.csdn.net/20140605202622953)

# III、配置数据库
1. 我用的是phpMyadmin管理数据库，可以将本地数据库导出成 .sql 形式
2. 回到SAE，点击左边导航条 服务管理-> MySql，点击右边的蓝色按钮，初始化完成后，点击管理MySql
![](https://img.blog.csdn.net/20140606123545734)
3. 将刚从本地到处的数据库上传
![](https://img.blog.csdn.net/20140606130016109)

# IV、部署代码
回到SAE管理MySql的页面，点击文档，看以看到如下信息
![](https://img.blog.csdn.net/20140606130453734)
下一步就是修改上传代码的数据库联系信息
![](https://img.blog.csdn.net/20140606130736625)
可以看到项目的目录结构，按照文档，更改代码里的数据库配置信息，每个人的情况不一样，我就不多说了
点击首页网址，就可以成功访问你的项目了。。。。。。
![](https://img.blog.csdn.net/20140606135101078)

恭喜！！！你的项目已经成功部署到SAE上了，赶快邀请好友去访问吧
               

