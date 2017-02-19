title: edb 调试带参数的应用程序
date: 2016-03-24 10:50:28
tags: 调试器
---
edg debugger 是Linux下图形化的调试器，和 ollydbg 一样强大。
<!-- more -->

## 安装及使用方法
我使用的 kali 系统，集成了 edb debugger，安装教程请参考 [Linxu edb安装](http://blog.csdn.net/ustczwc/article/details/8736271)

图形化的界面，使用方法基本是傻瓜式的。
但是调试带参数的运行程序，特别是参数是脚本的运行结果，需在命令行设置

## 带参数运行
假设 gdb 的命令为：
```
./uppercaser > in.txt < out.txt
```

则使用命令启动 edb
```
edb --run ./uppercaser > in.txt < out.txt
```

如果参数是脚本运行结果
```
edb --run ./uppercaser  "`./exploit.py`"
```

# 参考文献
[EDB - How do I debug a program with i/o redirection as the arguments?](http://stackoverflow.com/questions/27911456/edb-how-do-i-debug-a-program-with-i-o-redirection-as-the-arguments)
