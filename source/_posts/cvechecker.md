---
title:  cvechecker 安装和使用教程
date: 2016-06-17 22:10:30
tags: 渗透测试
---
cvechecker 通过扫描系统中已安装的软件列表，并将结果和CVE数据库匹配，报告系统中可能存在的漏洞。

# 1 安装
- 下载 [cvechecker](https://github.com/sjvermeu/cvechecker) 安装包
首先运行
```
autoreconf --force --install
```

- 运行标准配置命令（根据不同的环境，添加 --enable-*  选项）
 ```
 ./configure --enable-sqlite3
 ```
<font color="orange">如果不加 --enable-sqlite3 选项，安装好之后，初始化数据库的时候会出问题</font>
# 2 待解决问题
## 2.1 安装 libconfig
安装过程会出现如下问题：
![](http://ww2.sinaimg.cn/large/005CA6ZCjw1f5582ge5icj30k107qadf.jpg)

需要安装 libconfig，下载 [libconfig](http://www.hyperrealm.com/libconfig/libconfig-1.5.tar.gz
) 安装包

```
cd libconfig-1.5/
./configure
make
sudo make install
```
## 2.2 安装 sqlite3
安装完 libconfig 之后，又会出现如下问题，缺少 sqlite3 包
![](http://ww2.sinaimg.cn/large/005CA6ZCgw1f558grmtpsj30k305ognw.jpg)

安装 sqlite3 
```
sudo apt-get install sqlite3
sudo apt-get install libsqlite3-dev
apt-get install libsqlite3-tcl
```

# 3 编译安装
./configure 运行成功后，编译安装

```
make
sudo make install
```

# 4 使用
- 初始化 SQLite3 数据库
 ```
 cvechecker -i
 ```
- 加载 CVE 和版本匹配规则
 ```
 sudo apt-get install xsltproc
 pullcves pull
 ```
- 生成需要扫描的文件
 ```
 sudo find / -type f -perm -o+x > scanlist.txt
 echo "/proc/version" >> scanlist.txt
 ```
- 收集已经安装的软件列表
 ```
 sudo cvechecker -b scanlist.txt
 ```
- 输出匹配的 CVE 列表
 ```
 sudo cvechecker -r
 ```
部分结果截图如下：
![](http://ww1.sinaimg.cn/large/005CA6ZCjw1f55g7yrl64j30i80a8432.jpg)

# 5 参数解释
```
cvechecker --help
```
![](http://ww3.sinaimg.cn/large/005CA6ZCjw1f55gdhdxjnj30jx0ce44b.jpg)
# 6 参考文献
[cvechecker](https://github.com/sjvermeu/cvechecker)
[cvechecker Installation](https://github.com/sjvermeu/cvechecker/wiki/Installation)
