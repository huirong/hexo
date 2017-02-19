title: Linux 下安装 edb debugger
date: 2016-04-28 15:12:10
tags:
- 调试器
- Linux
categories: 调试器
---
edb 是个跨平台的 x86/x86-64 的可视化调试器，和 Ollydbg 类似。
Linux 是目前唯一正式支持的平台，但在 FreeBSD, OpenBSD, OSX 和 Windows 可使用部分功能。
<!-- more -->
# Ⅰ、下载
下载地址 [edb debugger](https://github.com/eteran)
# Ⅱ、安装步骤
希望大家养成阅读 readme 文档的好习惯，里面有安装教程。
## ① 安装依赖包
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1f3cgden37vj30a104qdg5.jpg)
```
sudo apt-get install qt4-dev-tools
sudo apt-get install libbost-dev

```
