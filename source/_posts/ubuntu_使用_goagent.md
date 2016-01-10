title: 'ubuntu_使用_goagent'
date: 2014-05-13 19:31
tags: goagent
---

1)  下载goagent  https://code.google.com/p/goagent/
<!--more-->
2)  解压  cd local   进入local文件夹
3)  在终端运行 sudo apt-get install addto-startup.py
                        sudo apt-get install goagent-gtk.py
     期间系统会提醒安装两个软件   使用以下两个命令安装
             sudo apt-get install python-appindicator
             sudo apt-get install python-vte


 4)   进入浏览器扩展程序
       然后进入local文件夹，将SwitchySharp_1_9_52.crx  拖入扩展程序安装插件![](http://img.blog.csdn.net/20140513192203093)
5)  此时浏览器右上角 就有goagent 图标 ，右击图标，单击选项
       ![](http://img.blog.csdn.net/20140513192344937)
       点击导入/导出 -> 从文件恢复   将SwitchyOptions.bak导入（此为配置文件）
       ![](http://img.blog.csdn.net/20140513192447578)
6)   点击浏览器 设置 -> 高级设置   
       选择 HTTPS/SSL 证书管理  将local下的CA.crt 导入
     ![](http://img.blog.csdn.net/20140513192546937)

          
 
