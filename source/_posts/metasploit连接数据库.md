title: 'metasploit连接数据库'
date: 2014-10-29 21:38
tags: metasploit
---


#（1）查看数据库基本信息
<!--more-->
       命令： cd /opt/metasploit
   cat config/database.yml
      结果如下图所示
![](http://img.blog.csdn.net/20141029212002844)






#（2）连接数据库

命令：db_connect msf3:4bfedfc2@localhost:7337/msf3dev
查看数据库是否连接成功    db_status
结果如图
![](http://img.blog.csdn.net/20141029212419589)





补充说明：
我在执行db_connect命令时，遇到过一个错误  No database drive installed.Try  "gem
install pg"
解决方法如下
 查看ruby版本  命令  cd /opt/metasploit/msf3
 /opt/metasploit/ruby/bin/ruby -v
/usr/bin/ruby -v
vi /opt/metasploit/msf3/msfconsole  



![](http://img.blog.csdn.net/20141029213346728)





然后将第一行的 !/usr/bin/env ruby 改为　!/opt/meatsploit/ruby/bin/ruby
![](http://img.blog.csdn.net/20141029213350550)


