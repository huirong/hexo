---
title: INFORMATION_SCHEMA 数据库
date: 2016-08-24 20:12:38
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
INFORMATION_SCHEMA数据库作为MySQL数据库元数据的一个中央集中仓库存在。
<!-- more -->
# 1 INFORMATION_SCHEMA 数据库
INFORMATION_SCHEMA 是一个“虚拟的数据库”，因为它不存放在磁盘任何位置。但它和其他数据库一样含有表，且表中的内容可以通过使用select语句和其它数据库一样查询访问。
此外，你还可以使用select来获取关于INFORMATION_SCHEMA其本身的信息。
# 2 information_schema 数据库表说明
表名 | 说明
-- | -- 
<font color="orange">SCHEMATA</font> | 提供了当前mysql实例中所有数据库的信息。是show databases的结果取之此表。
<font color="orange">TABLES</font> | 提供了关于数据库中的表的信息（包括视图）。详细表述了某个表属于哪个schema，表类型，表引擎，创建时间等信息。是show tables from schemaname的结果取之此表。
<font color="orange">COLUMNS</font> | 提供了表中的列信息。详细表述了某张表的所有列以及每个列的信息。是show columns from schemaname.tablename的结果取之此表。
<font color="orange">STATISTICS</font> | 提供了表中的列信息。详细表述了某张表的所有列以及每个列的信息。是show columns from schemaname.tablename的结果取之此表。
<font color="orange">USER_PRIVILEGES（用户权限）</font> | 给出了关于全程权限的信息。该信息源自mysql.user授权表。是非标准表。
<font color="orange">SCHEMA_PRIVILEGES（方案权限）</font> | 给出了关于方案（数据库）权限的信息。该信息来自mysql.db授权表。是非标准表。
<font color="orange">TABLE_PRIVILEGES（表权限）</font> | 给出了关于表权限的信息。该信息源自mysql.tables_priv授权表。是非标准表。
<font color="orange">COLUMN_PRIVILEGES（列权限）</font> | 给出了关于列权限的信息。该信息源自mysql.columns_priv授权表。是非标准表。
<font color="orange">CHARACTER_SETS（字符集）</font> | 提供了mysql实例可用字符集的信息。是SHOW CHARACTER SET结果集取之此表。
<font color="orange">COLLATIONS</font> | 提供了关于各字符集的对照信息。
<font color="orange">TABLE_CONSTRAINTS</font> | 描述了存在约束的表。以及表的约束类型。
<font color="orange">KEY_COLUMN_USAGE</font> | 描述了具有约束的键列。
<font color="orange">ROUTINES</font> | 提供了关于存储子程序（存储程序和函数）的信息。此时，ROUTINES表不包含自定义函数（UDF）。名为“mysql.proc name”的列指明了对应于INFORMATION_SCHEMA.ROUTINES表的mysql.proc表列。
<font color="orange">VIEWS</font> | 给出了关于数据库中的视图的信息。需要有show views权限，否则无法查看视图信息。
<font color="orange">TRIGGERS</font> | 提供了关于触发程序的信息。必须有super权限才能查看该表。