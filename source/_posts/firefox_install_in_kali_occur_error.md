title: firefox install in kali occur error
date: 2014-09-02 10:44
tags:
- firefox
- kali
categories:
- kali
---

在kali linux 中安装firefox 遇到如下错误
现在尚不能配置软件包 firefox-mozilla-build

不能配置(目前状态为 half-installed )

在处理时有错误发生：
```
firefox-mozilla-build

E: Sub-process /usr/bin/dpkg returned an error code (1)
```

解决方法
```
sudo apt-get install --reinstall firefox-mozilla-build
```

参见链接  [firefox install in kali occur error](https://forums.kali.org/archive/index.php/t-6852.html)
