---
title: sqli labs 安装
date: 2016-08-24 21:22:18
tags:
- 渗透测试
- SQL 注入
categories: SQL 注入
---
好久没学习 Web 安全了，都去搞逆向去了，心血来潮，重温下Web 安全，就从最简单的sql注入开始
<!-- more -->
# 1 sqli labs 安装
- 搭建 wamp 集成环境，教程网上都有。
- 从 <https://github.com/Audi-1/sqli-labs> 下载源码
- 将加压后的源码放在 www 目录下
- 打开sql-connections文件夹下的“db-creds.inc”文件 ，修改用户名密码为自己的
- 打开浏览器，输入网址 <http://localhost/sqli-labs-master>
- 点击setup/resetDB 链接在你的mysql中创造数据库
- 开始学习

# 2 基础知识
## 2.1 [infromation_schema 数据库](http://huirong.github.io/2016/08/24/information-schema/)
## 2.2 注释
1. #(请用%23来使用，一般会被转码)
2. -- (--后面加个空格，用%20或者+表示空格)
3. /*  */

# 3 参考文献
[安全科普：SQLi Labs 指南 Part 1](http://www.freebuf.com/articles/web/34619.html)
