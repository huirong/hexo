title: Intel Pin 4 ：修改应用程序指令
date: 2016-01-12 15:11:47
tags: Pin
---
Pin通常用于插装应用程序，它也用于修改程序指令。
<!-- more -->
最简单的方法是插入一个分析程序模拟一个指令，然后使用INS_Delete()删除原来的指令。还可以插入 直接 或 间接 分支指令（使用  INS_InsertDirectJump 和 INS_InsertIndirectJump），模拟控制流转移指令。

指令访问的内存地址可修改为由 INS_RewriteMemoryOperand 计算出的值的引用。

注意：只有当所有的 instrumentation routines 执行完后，修改才生效。因此，所有的 instrumentation routines 看到的都是原始的、未修改的指令。

# 参考文献
[Pin 用户手册：Modifying Application Instructions](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/index.html#MODIFYING) 
