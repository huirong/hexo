title: Intel Pin 8 ：编写 Pintool 时的性能开销
date: 2016-01-13 14:46:12
tags: Pin
---
Pintool 的编写方式对性能影响很大，本文主要介绍提高性能的几种技术。
<!-- more -->
从示例开始，下面的代码段源自 source/tools/SimpleExamples/edgcnt.cpp
tool 的 instrumentation 组件如下：
```
VOID Instruction(INS ins, void *v)
{
      ...

      if ( [ins is a branch or a call instruction] )
      {


        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR) docount2,
                       IARG_INST_PTR,
                       IARG_BRANCH_TARGET_ADDR,
                       IARG_BRANCH_TAKEN,
                       IARG_END);
      }

      ...
}
```

analysis 组件：
```
VOID docount2( ADDRINT src, ADDRINT dst, INT32 taken )
{
    if(!taken) return;
    COUNTER *pedg = Lookup( src,dst );
    pedg->_count++;
}
```

此 tool 用于计算控制流转移频率，需统计 calls 和 分支 指令，<font color="red">为了简单起见，本文统称为分支指令</font>
tool 的工作流程：
 - instrumentation 组件为每个分支指令插入 docount2 函数。
 - docount2 的 src、dst 、taken参数，为分支指令的源地址和目的地址，taken 表明指令是否执行。
 - 若没有执行，直接返回；如执行，说明发生控制流转移，通过 src 和 dst 参数查找此控制流边的计数器（如没找到，新建计数器），然后计数器加 1 。


# Shifting Computation for Analysis to Instrumentation Code
通常应用程序中平均每 5 条指令一条分支指令，接着调用 Lookup 函数，大大降低了应用程序执行速度。

改善方法：
我们注意到，每条指令的 instrumentation 代码只调用一次，而每次执行指令时都会执行对应的 analysis 代码。因此，如果将 analysis 中的部分计算转移到 instrumentation 中，将全面提高性能。

本例中
 - 直接分支指令，程序中有大量直接分支指令，因此在 instruction() 中调用 Lookup() 
 - 间接分支指令（即call），相对较少，使用原有 analysis 代码
基于此，我们增加了一个轻量级 analysis 函数，docount，而原有的 docount2() 保持不变。
```
VOID docount( COUNTER *pedg, INT32 taken )
{
    if( !taken ) return;
    pedg->_count++;
}
```

instrumentation 相对复杂：
```
VOID Instruction(INS ins, void *v)
{
      ...

    if (INS_IsDirectBranchOrCall(ins))
    {
        COUNTER *pedg = Lookup( INS_Address(ins),  INS_DirectBranchOrCallTargetAddress(ins) );

        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR) docount,
                       IARG_ADDRINT, pedg,
                       IARG_BRANCH_TAKEN,
                       IARG_END);
    }
    else
    {
        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR) docount2,
                       IARG_INST_PTR,
                       IARG_BRANCH_TARGET_ADDR,
                       IARG_BRANCH_TAKEN,
                       IARG_END);
    }

      ...
}
```

# Eliminating Control Flow
docount() 函数为内联函数将有助于提高性能，没有任何控制流转移（单基本块）的例程几乎可以保证为内联函数。
如果参数 taken 为 0 或 1 可以使用如下方法消除控制流转移：
```
VOID docount( COUNTER *pedg, INT32 taken )
{
    pedg->_count += taken;
}
```

