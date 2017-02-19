title: metasploit 连接数据库
date: 2014-10-29 21:38
tags:
- kail
- metasploit
categories:
- metasploit
---

# I、查看数据库基本信息
```
cd /opt/metasploit
cat config/database.yml
```
结果如下图所示：
![](http://img.blog.csdn.net/20141029212002844)

# II、连接数据库
```
db_connect msf3:4bfedfc2@localhost:7337/msf3dev
```
查看数据库是否连接成功
```
db_status
```
结果如图：
![](http://img.blog.csdn.net/20141029212419589)

<font color="red">Tips：</font>我在执行db_connect命令时，遇到过一个错误
```
No database drive installed.Try  "gem install pg"
```
解决方法如下：
查看ruby版本
```cd /opt/metasploit/msf3
 /opt/metasploit/ruby/bin/ruby -v
/usr/bin/ruby -v
vi /opt/metasploit/msf3/msfconsole  
```

![](http://img.blog.csdn.net/20141029213346728)

然后将第一行的 !/usr/bin/env ruby 改为　!/opt/meatsploit/ruby/bin/ruby
![](http://img.blog.csdn.net/20141029213350550)

