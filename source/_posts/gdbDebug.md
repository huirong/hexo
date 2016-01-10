title: GDB常用命令
date: 2015-04-30 13:54:06
tags: 调试器
---
调试器（比如GDB）能让你观察另一个程序在执行时的内部活动，或程序出错时发生了什么。
<!-- more -->
#GDB简介
GDB主要能为你做四件事(包括为了完成这些事而附加的功能)，帮助你找出程序中的错误。
 - 运行你的程序，设置所有的能影响程序运行的东西。
 - 保证你的程序在指定的条件下停止。
 - 当你程序停止时，让你检查发生了什么。
 - 改变你的程序。那样你可以试着修正某个bug引起的问题，然后继续查找另一 个bug。
你可以用GDB来调试C和C++写的程序。

**先说明一下**如何取得包括原代码符号的可执行代码。大家有心的话可以去看一下gcc的man文件在shell下打man gcc)。 gcc -g <原文件.c> -o <要生成的文件名> 
-g 的意思是生成带原代码调试符号的可执行文件。 
-o 的意思是指定可执行文件名。 
(gcc 的命令行参数有一大堆，有兴趣可以自己去看看。) 

#GDB命令
###<font color="orange">启动</font>
 - gdb $program
program就是你的执行文件，一般在当前目录下
 - gdb $program core
用gdb同时调试一个运行程序和core文件，core是程序非法执行后core dump后产生的文件。
 - gdb $program $pid
如果你的程序是一个服务程序，那么你可以指定这个服务程序运行时的进程ID。gdb会自动attach上去，并调试他。program应该在PATH环境变量中搜索得到。

###<font color="orange">运行</font>
 - r ：运行程序
 - continue/c ：继续运行
 - next/n ：下一行，但不进入函数调用
 - stop/s ：下一行，且进入函数调用
 - ni 或 si ：下一跳指令，ni与si的区别同n与s的区别
 - finish/fini ：继续运行至当前栈帧/函数刚刚退出
 - until/u ：  继续运行至某一行，在循环中，u可以实现运行至循环刚刚退出，但这取决于循环的实现
 
### <font color="orange">查看源码</font>
 - list/l linenum/function ：查看第linenum行或者function所在行附近的10行
 - list/l ：查看上一次list命令列出的代码后面的10行
 - list/l m,n ：查看从第m行到第n行的源码
 
### <font color="orange">断点</font>
 - break/b linenum/function ：在第linenum行或函数function处停住
 - break/b +/-offset ：在当前行号的前面/后面的offset行停住，offset为自然数
 - break filename:linenum ：在源文件filename的linenum行处停住
 - break filename:function ：在源文件filename的function函数入口处停住
 - break *address ：在程序运行的内存地址处停住
 - break ：break没有参数时，表示在吓一跳指令处停住

查看断点
 - info break

###<font color="orange">查看寄存器</font>
 - info registers ：查看寄存器的情况（除了浮点寄存器）
 - info all-registers ：查看所有寄存器的情况
 - info registers $regname  ：查看制定寄存器的情况，例如 info $rbp
 
修改寄存器的值
 - set $regname value ：改变指定寄存器的值，例如 set $rbx 0x5

###<font color="orange">查看变量的值</font>
 - p varable ：打印变量varable的值
 - p &varable ：打印变量varable的地址

###<font color="orange">查看帧栈信息</font>
查看调用栈
 - backtrace/bt ：显示程序的调用栈信息
 - backtrace/bt n ：显示程序的调用栈信息，只显示栈顶n帧

查看帧信息
 + frame/f n ：查看第n帧的信息
 + frame/f addr ：查看pc地址为addr的帧的相关信息
 + up n ：查看当前帧上面第n帧的信息
 + down n ：查看当前帧下面第n帧的信息

查看更新详细的信息
 * info frame、info frame n、info frame addr  查看制定帧的详细信息
 * info args ：查看当前帧中的参数
 * info locals ：查看当前帧中的局部变量
 * info catch ：查看当前帧中的异常处理器

###<font color="orange">退出</font>
 - quit/q ：退出GDB调试，若退出时被调试的程序尚未结束，GDB会提示，请求确认


目前只用过这些命令，以后会持续更新。。。。。

#参考文献
[比较全面的gdb调试命令](http://blog.csdn.net/dadalan/article/details/3758025)
[GDB使用小结](http://www.dutor.net/index.php/2011/02/gdb-summary/)
[手把手教你玩转GDB(四)——–函数调用栈(call stack)探密](http://www.wuzesheng.com/?p=1409)
