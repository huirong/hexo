---
title: 安装 LLVM
date: 2017-03-17 21:08:30
tags: LLVM
categories: LLVM
---
LLVM 相关概念请参考
- 英文原版 [The Architecture of Open Source Application:LLVM](http://www.aosabook.org/en/llvm.html)
- 中文翻译 [LLVM简介 - The Architecture of Open Source Application](http://www.chenhf.com/lang/llvm-intro)
本文仅记录 LLVM 的安装过程。
<!-- more -->
# 1 安装小提示
1. 最好<font color="red">不要</font> 在虚拟机上安装 LLVM，LLVM 比较吃内存，在虚拟机上很容易出现 <font color="red">“Memory exhausted”</font>  错误，若在物理机上遇到此错误，解决方法参见 [llvm/clang compile error with Memory exhausted](http://stackoverflow.com/questions/25197570/llvm-clang-compile-error-with-memory-exhausted)。
    
2. llvm为了防止编译的中间结果分布在码源目录中，影响码源的结构。因此不支持目录内编译。需要在码源目录外创建额外的编译目录（假设名字为build），build 至少 30G，**<font color="red">请预留 30G 磁盘空间</font>** ,编译需花费一个多小时，请严肃对待。
   笔者等待了一个多小时，最后内存不足，"mmap:failed to allocate 1660253772 bytes for output file:cannot allocate memory"。

3. llvm+clang 3.6以前的版本可是使用./configure来进行编译，3.6以后的版本，只能使用cmake进行编译。安装之前，使用 `sudo apt-get install cmake` 命令安装cmake。

安装之前，阅读以上要点。
安装之前，阅读以上要点。
安装之前，阅读以上要点。

# 2 LLVM 安装
LLVM 官方安装教程参见 [Getting Started with the LLVM System](http://llvm.org/docs/GettingStarted.html)

首先创建存在 LLVM 的文件夹，假设命令为 llvm。

## 2.1 下载源码
- (1)下载 LLVM 源码
```
cd llvm
svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
```
- (2)下载 clang
```
cd llvm/tools
svn co http://llvm.org/svn/llvm-project/cfe/trunk clang
```
- (3)下载 LLD linker [可选]
```
cd llvm/tools
svn co http://llvm.org/svn/llvm-project/lld/trunk lld
```
- (4)下载 Polly 循环优化器 [可选]:
```
cd llvm/tools
svn co http://llvm.org/svn/llvm-project/polly/trunk polly
```
- (5)下载 Compiler-RT (required to build the sanitizers) [可选]:
```
cd llvm/projects
svn co http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt
```
- (6)下载 Libomp (需 OpenMP 支持) [可选]:
```
cd llvm/projects
svn co http://llvm.org/svn/llvm-project/openmp/trunk openmp
```
- (7)下载 libcxx 和 libcxxabi [可选]:
```
cd llvm/projects
svn co http://llvm.org/svn/llvm-project/libcxx/trunk libcxx
svn co http://llvm.org/svn/llvm-project/libcxxabi/trunk libcxxabi
```
- (8)下载测试组件（Test Suite）源码[可选]
```
cd llvm/projects
svn co http://llvm.org/svn/llvm-project/test-suite/trunk test-suite
```

## 2.2 编译安装
- llvm+clang 3.6以前的版本可是使用./configure来进行编译，3.6以后的版本，只能使用cmake进行编译。
- llvm为了防止编译的中间结果分布在码源目录中，影响码源的结构。因此不支持目录内编译。需要在码源目录外创建额外的编译目录。

在 llvm 同级目录新建 build 文件夹，用于编译 llvm 源码，即该目录下，有两个文件夹：llvm(LLVM和clang等源码),build。

```
cd build
cmake -G "Unix Makefiles" ../llvm
make -j 2
sudo make install
```

cmake 命令格式：
`cmake -G <generator> [options] <path to llvm sources>`
常见的 generators 如下:
- Unix Makefiles — for generating make-compatible parallel makefiles.
- Ninja — for generating Ninja build files. Most llvm developers use Ninja.
- Visual Studio — for generating Visual Studio projects and solutions.
- Xcode — for generating Xcode projects.

# 3 参考文献
- [Getting Started with the LLVM System](http://llvm.org/docs/GettingStarted.html)
- [llvm+clang编译安装](http://www.cnblogs.com/Long-w/p/6345028.html)
- [llvm/clang compile error with Memory exhausted](http://stackoverflow.com/questions/25197570/llvm-clang-compile-error-with-memory-exhausted)