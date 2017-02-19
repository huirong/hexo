title: scrapy 数据库插入
date: 2014-05-22 22:30
tags:
- 爬虫
- python
categories:
- 爬虫
---

# I、安装mysql
- 安装mysql 
 ```
sudo apt-get install mysql
 ```
- 安装python-mysql
 ```
sudo apt-get install python-mysqldb
 ```
- 安装python支持mysql的驱动
 ```
sudo pip install pymysql
 ```
<font color="red">Tips：</font>安装时密码不要为空

# II、新建数据库
- 以root身份进入mysql
 ```
mysql -u root
 ```
- 新建数据库(数据库名 db)
 ```html
create database db
 ```
- 分配权限
 ```
GRANT ALL PRIVILEGES ON db.\* TO star@localhost IDENTIFIED BY "123456";
 ```
 <font color="red">Tips：</font>
 db 是刚建的数据库
 star 是新建的数据库用户
 12345 是密码
- 新建table
 ```
use db
 ```
- 设置编码
 ```
alter database mydb character set utf8
 ```
- 检查数据库编码是否设置成功
 ```python
show variables like 'character_set_%';
 ```
![](http://img.blog.csdn.net/20140522222746375)

然后自己去新建table(这里就不多说了)

# III、scrapy插入数据
- setting.py添加如下代码
 ```
ITEM_PIPELINES = ['wooyun.pipelines.WooyunPipeline']
 ```
<font color="red">Tips：</font>
wooyun  是scrapy项目名 
WooyunPipeline  是Pipeline 名
- pipeline.py 
 ```
# Define your item pipelines here
#
# Don't forget to add your pipeline to the ITEM_PIPELINES setting
# See: http://doc.scrapy.org/topics/item-pipeline.html
#

from scrapy import log
from twisted.enterprise import adbapi
from scrapy.http import Request
from scrapy.exceptions import DropItem
from scrapy.contrib.pipeline.images import ImagesPipeline
import time
import MySQLdb
import MySQLdb.cursors
import socket
import select
import sys
import os
import errno

class WooyunPipeline(object):
    def __init__(self):
        self.dbpool = adbapi.ConnectionPool('MySQLdb', db='db',
                user='star', passwd='cc', cursorclass=MySQLdb.cursors.DictCursor,
                charset='utf8', use_unicode=True)

    def process_item(self, item, spider):
        # run db query in thread pool
        query = self.dbpool.runInteraction(self._conditional_insert, item)
        query.addErrback(self.handle_error)

        return item

    def _conditional_insert(self, tx, item):
        # create record if doesn't exist.
        # all this block run on it's own thread
        tx.execute("select \* from wooyun where name = %s", (item['name'][0]))
        result = tx.fetchone()
        if result:
            log.msg("Item already stored in db: %s" % item, level=log.DEBUG)
        else:
            tx.execute(\
                "insert into wooyun (name,time,url) "
                "values (%s, %s,%s)",
                (item['name'][0],
                 item['time'][0],
                 item['url'][0])
            )
            log.msg("Item stored in db: %s" % item, level=log.DEBUG)

    def handle_error(self, e):
        log.err(e)

 ```










