title: Intel Pin 7 ：调试 Pintool 注意事项
date: 2016-01-13 09:39:32
tags: Pin
---
本文是调试 Pintool，并非在 Pintool 下运行的应用程序。
<!-- more -->
# Using GDB on Linux
在 Pin 和 Pintool 下运行应用程序时，Pin 和 应用程序都加载进内存，Pintool 通常是由 Pin 加载的共享对象。本节主要介绍如何使用 GDB 发现 Pintool 中的 bug。

由于 Pin 使用 debugging API 开启应用程序，所以不能直接在 GDB 中运行 Pin。需在 Pin 命令行中加 -pause_tool 选项，然后在另一个终端中使用 GDB 附加 Pin。-pause_tool 选项的功能，打印 PID 并暂停 n 秒。

Pin 内置搜索算法查找 tool，因此大多数情况下 GDB 不能加载 tool 的调试信息。GDB 可使用以下几种方法找到调试信息。
 - 使用完整路径运行 Pin
 - Pin 给出确切的 GDB 命令加载调试信息

如下所示，使用 "info sharedlibrary" 检查 GDB 是否加载了 tool 的调试信息，可以发现 GDB 读取了 tool 的符号。
```
(gdb) info sharedlibrary
From        To          Syms Read   Shared Object Library
0x001b3ea0  0x001b4d80  Yes         /lib/libdl.so.2
0x003b3820  0x00431d74  Yes         /usr/intel/pkgs/gcc/4.2.0/lib/libstdc++.so.6
0x0084f4f0  0x00866f8c  Yes         /lib/i686/libm.so.6
0x00df8760  0x00dffcc4  Yes         /usr/intel/pkgs/gcc/4.2.0/lib/libgcc_s.so.1
0x00e5fa00  0x00f60398  Yes         /lib/i686/libc.so.6
0x40001c50  0x4001367f  Yes         /lib/ld-linux.so.2
0x008977f0  0x00af7784  Yes         ./dcache.so
```

## 具体步骤
例如，如果 tool 为 opcodemix，应用程序是 /bin/ls，GDB的使用步骤如下：
此例子运行于 Intel(R) 64 Linux 平台，IA-32 架构使用 "ia32"。

 1. 进入 tool 所在的目录，启动 GDB 调试 Pin，但不运行（即，不使用 run 命令）。

    ```
    $ /usr/bin/gdb ../../../intel64/bin/pinbin
    GNU gdb Red Hat Linux (6.3.0.0-1.132.EL4rh)
    Copyright 2004 Free Software Foundation, Inc.
    GDB is free software, covered by the GNU General Public License, and you are
    welcome to change it and/or distribute copies of it under certain conditions.
    Type "show copying" to see the conditions.
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    This GDB was configured as "x86_64-redhat-linux-gnu"...Using host libthread_db library "/lib64/tls/libthread_db.so.1"
    (gdb)
    ```
 2. 在另一个终端使用 -pause_tool 选项启动应用程序

    ```
    $ ../../../pin -pause_tool 10 -t obj-intel64/opcodemix.so -- /bin/ls
    Pausing to attach to pid 28769
    To load the tool's debug info to gdb use:
       add-symbol-file .../source/tools/SimpleExamples/obj-intel64/opcodemix.so 0x2a959e9830
    ```
 3. 然后回到 GDB 附加程序

    ```
    (gdb) attach 28769
    Attaching to program: .../intel64/bin/pinbin, process 28769
    0x000000314b38f7a2 in ?? ()
    (gdb)
    ```
 4. 在 GDB 中运行第 2 步中提示的命令

    ```
    (gdb) add-symbol-file .../source/tools/SimpleExamples/obj-intel64/opcodemix.so 0x2a959e9830
    add symbol table from file ".../source/tools/SimpleExamples/obj-intel64/opcodemix.so" at
            .text_addr = 0x2a959e9830
            (y or n) y
            Reading symbols from .../source/tools/SimpleExamples/obj-intel64/opcodemix.so...done.
    (gdb)
    ```

 5. 使用 cont 命令设置断点，也可以先设置断点

    ```
    (gdb) b opcodemix.cpp:447
    Breakpoint 1 at 0x2a959ecf60: file opcodemix.cpp, line 447.
    (gdb) cont
    Continuing.

    Breakpoint 1, main (argc=7, argv=0x3ff00f12f8) at opcodemix.cpp:447
    447     int main(int argc, CHAR *argv[])
    (gdb)
    ```
 6. 如果程序没有退出，最后需运行 detach 命令释放控制权

    ```
    (gdb) detach
    Detaching from program: .../intel64/bin/pinbin, process 28769
    (gdb)
    ```

重编译程序后，再次使用 GDB 调试时，GDB 会提示二进制文件已修改，需重新读取调试信息。

# 参考文献
[Pin 用户手册：Tips for Debugging a Pintool](Tips for Debugging a Pintool)
