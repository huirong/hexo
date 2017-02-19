title: DigitalOcean初始化--使用SSH_Keys登录（Windows篇）
date: 2014-12-04 21:54
tags: VPN
categories: VPN
---

# I、SSH密钥

虽然可以使用基于密码的登录管理服务器，使用SSH密钥对会更好。 SSH密钥比密码更安全，并可以帮助您登录，而无需记住长密码。

用PuTTYgen来生成SSH密钥，PuTTY创建SSH会话，连接服务器。SSH密钥对分为公钥和密钥，公钥可以对外公布，密钥自己保管，用来连接服务器。

PuTTY、PuTTYgen [下载地址](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html)

# II、创建SSH密钥对
打开PuTTYgen，界面如下,点击 Generate 按钮
![](http://img.blog.csdn.net/20141204210557253)

由于SSH密钥使用的是信息安全的随机块创建，需要点击对话框的空白区域随机生成数据，要一直点击，直到完成
![](http://img.blog.csdn.net/20141204210627283)

生成之后的界面如下
![](http://img.blog.csdn.net/20141204210820820)

最好对密钥加密，在Key passphrase 输入框输入密钥密码，然后保存密钥和公钥 点击Save public key 和 Save private key，保存后的文件是.ppk格式

# III、上传公钥到DigitalOcean
打开DigitalOcean Control界面，点击SSH Keys -> Add SSH Key
![](http://img.blog.csdn.net/20141204212033771)

然后随便写一个公钥的名字，把刚才的公钥填写到下面，点击CREATE SSH KEY
![](http://img.blog.csdn.net/20141204212313703)

# IV、用刚才的公钥新建一个VPS服务器
![](http://img.blog.csdn.net/20141204212624890)

根据自己的情况填写，只是最后一次选择刚才新建的SSH Key
![](http://img.blog.csdn.net/20141204212738700)

# V、使用PuTTY创建SSH会话，连接服务器
打开PuTTY，选择windows，输入刚开新建的虚拟机的IP，在DO（DigitalOcean）的控制面板中可以看到，使用默认端口号22
![](http://img.blog.csdn.net/20141204213236848)

然后选择Data ，输入服务器的用户名，习惯root，是系统管理员
![](http://img.blog.csdn.net/20141204213529656)       ![](http://img.blog.csdn.net/20141204213619906)

SSH --> Auth   点击Browser（预览） 选择之前保存的 .ppk 密钥文件（第二步，Save privacy key保存的文件）
![](http://img.blog.csdn.net/20141204213157375)

最后创建session，点击Session。为Session取一个方便记忆的名字，然后保存（点击Save），然后可以关掉PuTTY了，以后可以用这个Session连接服务器
![](http://img.blog.csdn.net/20141204214035753)

# Ⅵ、用PuTTY保存的Session连接服务器
再次打开PuTTY，点击Session --> Load  双击刚才保存的Session 即DO_DROPLET_SESSION,然后点击下面的open，会出现一个警告框，选择yes

恭喜你，连上服务器了，输入你的密钥密码就行了（第二步设置的key passphrase）
![](http://img.blog.csdn.net/20141204215237738)

















