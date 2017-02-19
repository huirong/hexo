title: wampserver2.5-apache2.4.9允许外部访问的配置
date: 2014-10-21 10:15
tags: 
- apache
categories:
- apache
---

打开..\wamp\bin\apache\apache2.4.9\conf\httpd.conf配置文件，
```
<Directory "c:/wamp/www/">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options Indexes FollowSymLinks

    #
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride all
    Require all granted   #添加允许外部访问
 
    #
    # Controls who can get stuff from this server.
    #
 
    #   onlineoffline tag - don't remove
    # Require local  #注释请求本机访问


</Directory>
```

<font color="red">按代码 22 29 行修改</font>
重启服务，再访问下就可以了。
