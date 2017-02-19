title: SmashTheStack IO level2
date: 2015-05-18 12:30:03
tags: 
- 缓冲区溢出
- Smash the Stack
categories: Smash the Stack
---
由上一关取得的密码进入本关卡
<!-- more -->
# I、函数说明
## ① signal()
【定义函数】void (*signal(int signum, void(* handler)(int)))(int);
【函数说明】signal()会依参数signum 指定的信号编号来设置该信号的处理函数. 当指定的信号到达时就会跳转到参数handler 指定的函数执行. 如果参数handler 不是函数指针, 则必须是下列两个常数之一：
- SIG_IGN 忽略参数signum 指定的信号.
- SIG_DFL 将参数signum 指定的信号重设为核心预设的信号处理方式.
eg: signal(SIGINT ,SIG_DFL );
SIGINT信号代表由InterruptKey产生，通常是CTRL+C或者是DELETE。发送给所有ForeGroundGroup的进程。 SIG_DFL代表执行系统默认操作，其实对于大多数信号的系统默认动作时终止该进程。

## ② SIGFPE
 数学相关的异常，如被0除，浮点溢出，等等
## ③ atoi()
【定义函数】int atoi (const char * str);
【函数说明】atoi()函数会扫描参数str字符串，跳过前面的空白字符（例如空格，tab缩进等，可以通过isspace() 函数来检测），直到遇上数字或正负符号才开始做转换，而再遇到非数字或字符串结束时('\0')才结束转换，并将结果返回。
【返回值】返回转换后的整型数；如果str不能转换成int或者str为空字符串，那么将返回 0。
## ④ abs()
【定义函数】int abs (int j);
【函数说明】abs()用来计算参数j 的绝对值，然后将结果返回。
【返回值】返回参数j 的绝对值结果。
# II、找漏洞
## ① 分析源码
源代码：
```
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void catcher(int a)
{
    setresuid(geteuid(),geteuid(),geteuid());
    printf("WIN!\n");
    system("/bin/sh");
    exit(0);
}

int main(int argc, char **argv)
{
    puts("source code is available in level02.c\n");

    if (argc != 3 || !atoi(argv[2]))
        return 1;
    signal(SIGFPE, catcher);
    return abs(atoi(argv[1])) / atoi(argv[2]);
}
```
源码分析：
- （1）main函数参数：
if (argc != 3 || !atoi(argv[2])) 如果不是三个参数或第二个参数为0 函数退出。
main函数文件名本身是一个参数，因此在运行时，应该输入两个参数，argc才为3
- （2）signal(SIGFPE, catcher);
如果有数字相关的异常产生，调用catcher()函数
- （3）返回 atoi(第一个参数)/atoi(第二个参数)的绝对值

## ② 查看运行效果
![](https://ww2.sinaimg.cn/large/005CA6ZCjw1esdajv6jf5j30ed0270t1.jpg)
没有任何反应
## ③ 找漏洞
最终目的是调用catcher()函数，执行system("/bin/sh");
应该产生一个与数字相关的异常，除数为0的情况已经排除在外了，只能想其他办法了。
不知大家是否记得整数的表示范围呢，32位操作系统：-2^31~2^31-1
绝对值之间差1，bingo，这是漏洞点，最小负数除1，然后取绝对值，溢出了。。。。。

![](https://ww1.sinaimg.cn/large/005CA6ZCjw1esdak4gsgtj30hp03mwf5.jpg)

PS：相信大家都看到了，还有个level02_alt.c函数，你也可以找这个函数的漏洞来获取下一关的密码，大家自己试试，我不写博客了。。。。。。
