title: ubuntu 使用 GoAgent
date: 2014-05-13 19:31 
tags:
- GoAgent
- Ubuntu
categories:
- GoAgent
---
使用 GoAgent 可以无障碍地访问国外的网站，如Facebook、Twitter、YouTube等。
<!--more-->
# I、安装 GoAgent 
- 下载 [GoAgent](https://code.google.com/p/goagent/)
- 安装
 解压之后，进入 local 目录
 ```
 cd local
 sudo apt-get install addto-startup.py
 sudo apt-get install goagent-gtk.py
 ```
期间系统会提醒安装两个软件，使用以下两个命令安装
```
sudo apt-get install python-appindicator
sudo apt-get install python-vte
```

## II、配置 chrome 浏览器 
1. 进入 local 文件夹，将 SwitchySharp_1_9_52.crx  拖入扩展程序安装插件
 ![](https://img.blog.csdn.net/20140513192203093)
2. 此时浏览器右上角 就有goagent 图标 ，右击图标，单击选项
 ![](https://img.blog.csdn.net/20140513192344937)
3. 点击导入/导出 -> 从文件恢复   将SwitchyOptions.bak导入（此为配置文件）
![](https://img.blog.csdn.net/20140513192447578)
4. 点击浏览器 设置 -> 高级设置，选择 HTTPS/SSL 证书管理  将local下的CA.crt 导入
![](https://img.blog.csdn.net/20140513192546937)

          
 
