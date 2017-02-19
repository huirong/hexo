title: SmashTheStack IO level3
date: 2015-05-22 20:25:03
tags: 
- 缓冲区溢出
- Smash the Stack
categories: Smash the Stack
---
密码：IFd92yzOnSMv9tkX
<!--more-->
这个是最基本的缓冲区溢出，如果不明白缓冲区溢出的，可以自己百度。。。。。。
# I、源码
## ① 源代码：
```
//bla, based on work by beach

#include <stdio.h>
#include <string.h>

void good()
{
    puts("Win.");
    execl("/bin/sh", "sh", NULL);
}
void bad()
{
    printf("I'm so sorry, you're at %p and you want to be at %p\n", bad, good);
}

int main(int argc, char **argv, char **envp)
{
    void (*functionpointer)(void) = bad;
    char buffer[50];
    if(argc != 2 || strlen(argv[1]) < 4)
        return 0;

    memcpy(buffer, argv[1], strlen(argv[1]));
    memset(buffer, 0, strlen(argv[1]) - 4);

    printf("This is exciting we're going to %p\n", functionpointer);
    functionpointer();

    return 0;
}
```
## ② 执行结果
![](https://ww2.sinaimg.cn/large/005CA6ZCgw1eslav1k4bwj30io027q3r.jpg)
## ③ 分析结果
最终要跳转到good()函数执行，调用 execl("/bin/sh", "sh", NULL);
bad()函数的起始地址是0x80484a4，good()函数的起始地址是0x8048474
起初functionpointer是指向bad()函数的，需要构造输入，使buffer溢出，修改functionpointer的值
# II、函数说明
## ① memcpy()
【定义函数】void *memcpy(void*dest, const void *src, size_t n);
【函数说明】
1. source和destin所指内存区域不能重叠，函数返回指向destin的指针。
2. 与strcpy相比，memcpy并不是遇到'\0'就结束，而是一定会拷贝完n个字节。 memcpy用来做内存拷贝，你可以拿它拷贝任何数据类型的对象，可以指定拷贝的数据长度；
3. 如果目标数组destin本身已有数据，执行memcpy（）后，将覆盖原有数据（最多覆盖n）。如果要追加数据，则每次执行memcpy后，要将目标数组地址增加到你要追加数据的地址。
 注意，source和destin都不一定是数组，任意的可读写的空间均可。
【返回值】函数返回一个指向dest的指针。

## ② memset()
【定义函数】void * memset( void * ptr, int value, size_t num );
【函数说明】
 memset()会将ptr所指的内存区域的前num个字节的值都设置为value，然后返回指向ptr的指针。
memset()可以将一段内存空间全部设置为特定的值，所以经常用来初始化字符数组。
【返回值】返回指向 ptr 的指针。
# III、找漏洞
## ① 栈
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1eslavce0j0j30mc0c6776.jpg)
ebp:0xbffffc88，esp:0xbffffc10，栈范围：0xbffffc10~0xbffffc88
## ② 内存
![](https://ww1.sinaimg.cn/large/005CA6ZCgw1eslavkosj2j30nn093jwt.jpg)
查看functionpointer的地址，分析其和buffer起始地址的距离
functionpointer在buffer前定义，先入栈，在高地址，functionpointer（0x80484a4）的地址为0xbffffc7c
buffer的起始地址为0xbffffc30，中间隔了76Byte
## ③ 构造输入
需要先输入67个任意字符，到functionpointer前地址，0x8048474（good()函数起始地址）和0x80484a4（bad()函数起始地址）只有最后一个Byte不同，只用修改最后1Byte值，所以构造的输入：76*A+0x74
"$(python -c 'print "A"*76 + "t" ' )"
（t的ascii码值：0x74）

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1eslavpqx9dj30k80350tk.jpg)



