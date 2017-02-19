title: SmashTheStack IO level4
date: 2015-05-22 21:42:13
tags: 
- 缓冲区溢出
- Smash the Stack
categories: Smash the Stack
---
密码：nSwmULj2LpDnRGU2
<!-- more -->
# I、源代码
## ① 源代码：
```
//writen by bla
#include <stdlib.h>
#include <stdio.h>

int main() {
    char username[1024];
    FILE* f = popen("whoami","r");
    fgets(username, sizeof(username), f);
    printf("Welcome %s", username);

return 0;
}
```
## ② 执行结果：
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1esldar1lq1j30i801kglq.jpg)
## ③ 分析结果
welcome level5，说明系统是在level5下执行的whoami命令，而我们想要的是在level5下执行 cat /home/level5/.pass 命令，查看level5密码
程序不需要任何输入，我们不能构造输入；不能改变源代码，因此不能在popen中添加命令
我们可不可以改变whoami命令的内容呢？？
先看下相关介绍，或许会有思路
# II、相关知识
## ① PATH环境变量
$PATH：决定了shell将到哪些目录中寻找命令或程序，PATH的值是一系列目录，当您运行一个程序时，Linux在这些目录下进行搜寻编译链接。

<font color="red">思路：</font>在$PATH中添加一个目录，指向我们自己创建的whoami命令文件，whoami中的内容是 cat /home/level5/.pass，shell执行whoami命令时，先找到我们创建的目录，然后就不在向下查看，直接执行命令
## ② which
which命令的作用是，在PATH变量指定的路径中，搜索某个系统命令的位置，并且返回第一个搜索结果。也就是说，使用which命令，就可以看到某个系统命令是否存在，以及执行的到底是哪一个位置的命令。
# III、找漏洞
## ① 查看命令位置
```
which whoami
```
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1esldtip2x7j30k301jt8t.jpg)
## ② 构造PATH并验证是否成功
自定义whoami内容：
![](http://ww3.sinaimg.cn/large/005CA6ZCgw1esldn58eobj30k403n3zw.jpg)

赋予执行权限
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1esldo6oa1kj30k40230t9.jpg)

添加新的whoami命令路径：
![](http://ww4.sinaimg.cn/large/005CA6ZCjw1esldp2b06jj30k302ldgx.jpg)

再次查看whoami路径：
![](http://ww3.sinaimg.cn/large/005CA6ZCjw1esldpzi8w0j30k301it8t.jpg)

## ③ 查看密码
![](http://ww1.sinaimg.cn/large/005CA6ZCgw1esldqe3qs3j30k2012dfx.jpg)
密码：nSwmULj2LpDnRGU2

