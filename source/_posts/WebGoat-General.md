---
title: WebGoat 学习 -- General 
date: 2016-09-07 21:03:17
tags:
- 渗透测试
- WebGoat
categories: WebGoat
---
# 1 工具与插件
## 1.1 WebScarab
### 1.1.1 简介
WebScarab 是一个用来分析由浏览器提交到服务器请求，以及服务器对浏览器做出的响应的应用服务框架，也可以当做一个代理工具，或者说就是一个代理。使用者可以利用 WebScarab 看、分析、修改、创建所截取的浏览器与服务器之间的请求与响应，也可以用来分析HTTP 与HTTPS 协议。

网页的对话输入框中可能存在某些限制，比如长度，格式等等。现在使用者可以在 WebScarab 截获请求对话框中对其进行修改；还可以利用WebScarab 对网站进行注入型攻击，以测试网站的安全性。

WebScarab 也是WebGoat 的一个辅助工具，在WebGoat 进行某些漏洞攻击时，可以利用 WebScarab 分析，修改所提交的请求，以及返回的响应。

但是 WebScarab 工具好像在七八年前就停止更新了，大家可以选择更好的工具。。。。

### 1.1.2 安装
- 下载  <https://sourceforge.net/projects/owasp/files/WebScarab/>
根据系统选择对应的版本，我安装在Linux系统下，下载的是 webscarab-selfcontained-20070504-1631.jar
- 安装 首先要有 JRE 环境，上一节已经安装了 JDK，所以可以使用命令直接运行 `java -jar webscarab-selfcontained-20070504-1631.jar`

## 1.2 Firebug
相信对 Web 安全有一定了解的人，对 Firebug 一定不陌生。

Firebug 是Firefox 下的一个插件,能够调试所有网站语言,如HTML,CSS 等，但FireBug 最吸引人的就是 JavaScript 调试功能，使用起来非常方便，而且在各种浏览器下都能使用（IE,Firefox,Opera, Safari）。除此之外，还有其他强大的功能，比如HTML,CSS,DOM 的察看与调试，网站整体分析等等。总之，就是一套完整而强大的 WEB 开发工具。再有就是其为开源的软件。

# 2 General（Http Basic（Http 基础知识））
## 2.1 总体目标

在页面下方的输入框中输入你的名字，点击【Go!】按钮提交。服务器接收请求，并逆置你的输入，并将结果返回给客户端，显示在页面的输入框中。该过程主要是展示 HTTP 请求的基本操作原理。

点击页面上方的按钮可以查看：Java 源代码、提
示信息、HTTP 请求参数、HTTP 请求 Cookies 。您也可以在首次使用时尝
试使用工具WebScarab。

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7lcb90i5fj30gr040q3a.jpg)

## 2.2 Http 工作原理
所有的HTTP 传输遵循同样的通用格式，每个客户端的请求和服务端的响应都有三个部分:  <font color="red"> 请
求或响应行、一个报头部分、实体部分 </font> 。客户端以如下方式启动一个交互：
- 客户端连接服务器并发送一个文件请求  
    `GET /index.html?param=value HTTP/1.0`
- 客户端发送可选头信息，告知接收服务器其配置和文件格式。
    `User-Agent: Mozilla/4.06 Accept: image/gif, image/jpeg, */*`
- 发送请求和报头之后，客户端可以发送更多的数据。该数据主要用于使用POST 方法的
CGI 程序。

## 2.3 WebScarab 拦截并分析
- 在浏览器中手动配置代理，然后启动 WebScarab：
    <font color="red"> 代理端口为 8008，我刚开始写错了，一直拦截不到信息(╯﹏╰) </font>

    ![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7lx3ageg0j30f50hcmzc.jpg)
    
- 然后在 WebScarab 的 Intercept 中选取 Intercept requests ，设置拦截请求：
    
    ![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7lxbj6sfqj30hz0eoq4g.jpg)

- 在输入框中输入你的名字，点击【Go!】按钮。然后观察 WebScarab 的新窗口中。
    
    ![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7lx9mqoe4j30i20eqq5r.jpg)

- 回到浏览器，页面提示成功完成本课程，左侧菜单栏看到醒目的绿色图标——过关标志。
    
    ![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f7lxqx5enhj30lz0bf77d.jpg)

OK，果然第一个课程很简单，感觉主要是为了熟悉 WebGoat 环境 o(〃'▽'〃)o





