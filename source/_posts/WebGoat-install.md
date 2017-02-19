---
title: WebGoat 安装
date: 2016-09-07 15:55:51
tags:
- 渗透测试
- WebGoat
categories: WebGoat
---
WebGoat 是 OWASP 组织研制出的用于进行 web 漏洞实验的应用平台，用来说明 web 应用中存在的安全漏洞。
<!-- more -->
# 1 WebGoat 简介
WebGoat 运行在带有java 虚拟机的平台之上，当前提供的训练课程有30 多个，其中包括：跨站点脚本攻击（XSS）、访问控制、线程安全、操作隐藏字段、操纵参数、弱会话cookie、SQL 盲注、数字型SQL 注入、字符串型SQL 注入、web 服务、Open Authentication 失效、危险的HTML 注释等等。
# 2 OWASP 简介
OWASP 是一个开放式Web 应用程序安全项目（OWASP，Open Web Application Security Project）组织，它提供有关计算机和互联网应用程序的公正、实际、有成本效益的信息。其目的是协助个人、企业和机构来发现和使用可信赖的软件。

开放式Web 应用程序安全项目（OWASP）是一个非营利组织，不附属于任何企业或财团。因此，由OWASP 提供和开发的所有设施和文件都不受商业因素的影响。

# 3 安装/升级 JDK
WebGoat 运行在带有java 虚拟机的平台上，因此需要安装 JDK。
而且如果版本过低，会出错，最好还是升级 JDK 到最新版。升级 JDK 步骤和安装相同，不用卸载原来的 JDK。

<font color="red"> 如果是在 Windows 下安装，可自行绕过此步骤，毕竟双击 exe 谁都会~~~~~ </font>

我原装 JDK 如下：
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7l54687c4j30j5025wez.jpg)

JDK 版本太低，在后续运行 WebGoat 出错，所以需要升级。

## 3.1 下载
下载最新版 JDK，网址 <http://www.oracle.com/technetwork/java/index.html>
根据自己的机型选择对应的版本。

将安装包移动到 `/usr/local/java` 目录下，并解压。
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7l4tebd2qj30gt02mq3e.jpg)

## 3.2 修改环境变量
修改 `/etc/profile` 配置文件，在文件末尾添加如下代码，不要修改其他部分。
```
JAVA_HOME="/usr/local/java/jdk1.8.0_101"
CLASSPATH="./:/usr/local/java/jdk1.8.0_101/lib"
PATH=$PATH:$JAVA_HOME/bin:$CLASSPATH
export PATH
```

可能大家下载的 jdk 版本和我的不同，根据自己的修改jdk版本号。

保存并退出，`source /etc/profile`

## 3.3 设置成系统默认的jdk
```
update-alternatives --install /usr/bin/javac javac /usr/local/java/jdk1.8.0_101/bin/javac 1071
update-alternatives --install /usr/bin/java java /usr/local/java/jdk1.8.0_101/bin/java 1071
update-alternatives --config java
```

最后一个 --config java 命令，shell会提示你选择哪个jdk作为默认的jdk，第一个带 * ，就是刚才设置的，直接enter就可以了。

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7l5azp2crj30s0045abh.jpg)

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7l5b917d5j30s00633zw.jpg)

## 3.4 查看是否安装/升级成功
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7l5citwzzj30gy02lwes.jpg)
说明安装/升级成功！！！！！

# 4 安装 WebGoat
下载简易安装 jar 可执行文件
<https://s3.amazonaws.com/webgoat-war/webgoat-container-7.0-SNAPSHOT-war-exec.jar>

在终端运行如下命令：
```
java -jar webgoat-container-7.0.1-war-exec.jar
```

在浏览器访问 <http://localhost:8080/WebGoat>
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7l5i4d4kij30rz0ir405.jpg)

可根据页面下方的用户名和密码登录 O(∩_∩)O~~

# 5 参考文献
[WebGoat: A deliberately insecure Web Application](https://github.com/WebGoat/WebGoat/blob/master/README.MD)
[debian 6 安装 JDK 、eclipse、 SDK 笔记](http://www.cnblogs.com/chineseboy/archive/2013/05/07/3064873.html)
