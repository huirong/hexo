---
title: WebGoat 学习 -- Access Control Flaws（访问控制缺陷）
date: 2016-09-08 10:29:39
tags:
- 渗透测试
- WebGoat
categories: WebGoat
---
访问控制知识点可参考 [第六章 访问控制](http://www.cec-ceda.org.cn/information/book/info_6.htm)
<!-- more -->
# 1 Using an Access Control Matrix（使用访问控制模型）
在基于角色的访问控制方案中，一个角色表示一组访问权限和特权，一个用户可以分配一个或多个角色。基于角色的访问控制方案通常由两部分组成：角色权限管理和角色分配。
一个被破坏的方案控制方案可能允许用户执行不在他/她分配的角色范围内的访问，或以某种方式允许特权升级到未经授权的角色。

## 1.1 General Goal(s) 目标
每个用户都是角色的成员，每个角色允许访问特定资源。
本课程的目标是探索管理此网站使用的访问控制规则，只有 "Admin" 组才有 "Account Manager(账户管理)" 资源。

## 1.2 探索
在上一节 General 中，设置了代理，本小节不需要设置代理，记得取消代理。

- 先选择一个用户，在选择一个资源，点击【Check Access】按钮
    ![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f7m0zycfgoj30ir046aae.jpg)

    `* User Moe [Public] was allowed to access resource Public Share`
    共有用户 Moe 对资源 Public Share 有访问权限

- 继续测试用户的访问控制权限，选择用户和资源，点击【Check Access】
    ![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7m2rfvuzuj30ir047aaj.jpg)

    共有用户 Moe 没有访问资源 Performance Review 的权限。

- 继续测试
    ![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7m2rxfw6lj30ir04ujs8.jpg)

    恭喜你，成功完成本课程。
    用户 Larry 对资源Account Manager 具有访问权限时。
    同时左侧菜单栏看到醒目的绿色图标——过关标志。

# 2 Bypass a Path Based Access Control Scheme(绕过基于路径的访问控制方案)
在一个基于路径的访问控制方案中，攻击者可以通过提供相对路径信息遍历路径。因此，攻击者可以使用相对路径访问那些通常任何人都不能直接访问或直接请求就会被拒绝的文件。

## 2.1 General Goal(s) 目标
用户 'guest' 具有访问 lessonPlans/en 目录下所有文件的权限。我们需要尝试突破访问控制侧路，访问不在下列清单中的文件。
选中一个文件，点击【View File】按钮，WebCoat会提示是否具有访问该文件的权限，可是试着访问“WEB-INF/spring-security.xml”文件。注意：该文件的位置取决于您正在使用的WebGoat 版本和环境。

## 2.2 绕过实验
### 2.2.1 测试
随便打击框框中的一个文件，然后【View File】，页面上方红色的有提示，下面显示页面的具体内容

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7m5qdo181j30iu0g3tbd.jpg)

### 2.2.2 使用 WebScarab 拦截并修改
在使用 WebScarab 前，记得设置浏览器的代理。

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7m6fggkpgj30l90e5n03.jpg)

URL 为 `http://localhost:8080/WebGoat/attack?Screen=320&menu=200&File=DOMXSS.html&SUBMIT=View+File`  

File 的值即为想要访问的文件名，修改这个文件为 WEB-INF/spring-security.xml 的路径，`http://localhost:8080/WebGoat/attack?Screen=320&menu=200&File=../../../../../WEB-INF/spring-security.xml&SUBMIT=View+File`

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7m6n08zdtj30l90e5juh.jpg)

然后点击 【Accept changes】之后查看浏览器内容

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7m78tp2u1j30qy0ar78d.jpg)

- 页面上显示 spring-security.xml 的绝对路径，还有恭喜通过本课程的提示！！！！
- 下方是 spring-security.xml 的内容。
- 左侧菜单栏醒目的绿色图标——过关标志。

<font color="red"> 不同机器和环境，spring-security.xml的目录不同 </font>

# 3 LAB:Role Based Access Control
很多网站都尝试使用基于角色的方式严格限制资源访问，但开发人员在实现这类解决方案时容易出现疏忽。
## 3.1 Stage 1: Bypass Presentational Layer Access Control
### 3.1.1 General Goal（目标）
即：绕过表示层访问控制。

Tom 是一名普通员工，利用脆弱的访问控制策略在员工列表页面中执行 Delete（删除）功能。验证 Tom 的 profile（个人档案）可以被删除。
每个用户的密码是该用户名的小写字母（Tom Cat 的密码为 tom）。

该公司内部员工层次图：

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f7mi0pmp1pj30h40c675u.jpg)

访问控制矩阵如下：

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7mi185ydjj30n40770tj.jpg)

### 3.1.2 初探
通过员工层次图和访问控制矩阵可以发现，员工的所有上司（直系或非直系）都具有删除其 profile 的权限。以 Tom 的直系上司 Jerry Mouse 的身份登录：

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7mhne9n7lj30h80br75b.jpg)

设置浏览器代理，然后开启 WebScarab，选中 Joanne McDougal ，点击 【DeleteProfile】，观察 WebScarab 拦截到的信息：

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7mhtgltbwj30l10du41m.jpg)

然后点击【Abort request】终止请求，回到浏览器点击【ViewProfile】，观察 WebScarab 拦截到的信息：

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7mhy5p43dj30l10dun0a.jpg)

多次实验，发现同一个用户，点击不同的操作，只有 action 参数不同。

### 3.1.3 验证
为了验证 Tom 的 profile（个人档案）可以被删除，以 Tom 身份登录，点击【ViewProfile】，尝试使用 WebScarab 拦截请求，修改 action 参数。

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1f7mi3ucyovj30h80b0mxy.jpg)

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f7mi4liz17j30l10duad6.jpg)

修改 action 参数，然后点击【Accept changes】

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7mid0kzjcj30l10du0vu.jpg)

返回到浏览器，发现通关了 \(^o^)/YES!

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7mi9xyk63j30mt0e3diu.jpg)

## 3.2 stage2：Add Business Layer Access Control
添加业务层访问控制。
<font color="red"> 本课程只适用于 WebGoat 开发版。</font>

本课程是修改 stage 1 的 bug，拒绝未经授权的用户使用 Delete 功能，为此一定要修改 WebGoat 源码。
修复成功后，可以返回 stage 1 验证是否拒绝未经授权的使用 delete 功能。

由于 WebGoat 源码是用 Java 写的，我是 Java 小白，so 本教程暂且搁置。

## 3.3 stage3: Bypass Data Layer Access Control
绕过数据层访问控制

使用 Tom 身份登录进系统，点击【ViewProfile】，看到 Tom 的个人信息：
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1f7yzr03cl0j30go0b5gn7.jpg)

WebScarab 抓包查看数据：
![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f7yzukia87j30l10du41m.jpg)

修改 employee_id 的值，110（随机选取）
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1f7yzxvcch5j30go0b7wgi.jpg)

OK，页面上成功显示编号为 110 的员工个人信息。
观察页面，有通关提醒 Y(^o^)Y

## 3.4 stage2：Add Data Layer Access Control
添加业务层访问控制。
<font color="red"> 本课程只适用于 WebGoat 开发版。</font>

本课程是修改 stage 3 的 bug，我是 Java 小白，so 本教程暂且搁置。



