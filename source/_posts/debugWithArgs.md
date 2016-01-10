title: gdb和insight debugger调试带参数程序
date: 2015-05-17 15:54:46
tags: 调试器
---
gdb --args ./testprg arg1 arg2 
<!-- more -->
另一种方式：
正常启动  gdb testprg
带参数运行 run arg1 arg2


insight是gdb的图形化界面，基本命令都一样
insight --args ./testprg arg1 arg2
