title: Intel Pin 2 ：示例
date: 2015-12-30 21:09:35
tags: Pin
---
为了说明如何编写Pintools，这里简单介绍几个简单的例子。
对应的源码都可以在 source/tools/ManualExamples 目录中找到。
<!-- more -->
# Building the Example Tools
首先进入 source/tools/ManualExamples 目录
```
cd source/tools/ManualExamples
```
 - build 此目录中的所有examples：
    ```
    make all
    ```
 - build 并 run 单个example（如，inscount0）
     ```
     make inscount0.test
     ```
 - build 单个example但不run（如，inscount0）
     ```
     make obj-intel32/inscount0.so
     ```
     上述适用于 Intel32架构，对于 IA-32 ，使用 "obj-intel64" 替换 "obj-intel32"
    ```
    make obj-intel64/inscount0.so
    ```

** <font color="red">build Windows tools注意事项</font> ** 
由于使用 make build tools，首先需要安装 cygwin make

# Simple Instruction Count (Instruction Instrumentation) 
下面的例子是统计程序中所有执行过的指令数。在每个指令前，Pin插入 docount 调用代码，当程序退出时，结果默认保存在 inscount.out文件中。

运行指令和结果如下：
```
../../../pin -t obj-intel32/inscount0.so -- /bin/ls

cat inscount.out
```
![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ezqzh1lcl2j30nk0al0uu.jpg)

下例中的 KNOB 用于重定向输出，命令行选项"-o <file_name>"，添加在工具名称和"--"之间，如下所示：
```
../../../pin -t obj-intel64/inscount0.so -o inscount0.log -- /bin/ls
```

可以在 source/tools/ManualExamples/inscount0.cpp 找到该示例程序
```
#include <iostream>
#include <fstream>
#include "pin.H"

ofstream OutFile;

// The running count of instructions is kept here
// make it static to help the compiler optimize docount
static UINT64 icount = 0;

// This function is called before every instruction is executed
VOID docount() { icount++; }
    
// Pin calls this function every time a new instruction is encountered
VOID Instruction(INS ins, VOID *v)
{
    // Insert a call to docount before every instruction, no arguments are passed
    INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)docount, IARG_END);
}

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "inscount.out", "specify output file name");

// This function is called when the application exits
VOID Fini(INT32 code, VOID *v)
{
    // Write to a file since cout and cerr maybe closed by the application
    OutFile.setf(ios::showbase);
    OutFile << "Count " << icount << endl;
    OutFile.close();
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This tool counts the number of dynamic instructions executed" << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */
/*   argc, argv are the entire command line: pin -t <toolname> -- ...    */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();

    OutFile.open(KnobOutputFile.Value().c_str());

    // Register Instruction to be called to instrument instructions
    INS_AddInstrumentFunction(Instruction, 0);

    // Register Fini to be called when the application exits
    PIN_AddFiniFunction(Fini, 0);
    
    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Instruction Address Trace (Instruction Instrumentation)
上例中，在分析阶段没有传参，本节将介绍如何传递参数，参数可以是指令指针、当前寄存器的值、有效内存地址、常数等。查看完整参数类型列表，请参见[IARG_TYPE.](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/group__INST__ARGS.html#g7e2c955c99fa84246bb2bce1525b5681)

和上例相比，做了微小改动，使用INS_InsertCall 函数传递即将执行的指令地址，并使用printip 替换 docount ，打印指令地址。默认输出函数为 itrace.out。

运行指令和结果如下：
```
../../../pin -t obj-intel32/itrace.so -- /bin/ls

head itrace.out
```

![](http://ww4.sinaimg.cn/large/005CA6ZCgw1ezr0c4mj07j30mr0enac6.jpg)

此示例在 source/tools/ManualExamples/itrace.cpp

```
#include <stdio.h>
#include "pin.H"

FILE * trace;

// This function is called before every instruction is executed
// and prints the IP
VOID printip(VOID *ip) { fprintf(trace, "%p\n", ip); }

// Pin calls this function every time a new instruction is encountered
VOID Instruction(INS ins, VOID *v)
{
    // Insert a call to printip before every instruction, and pass it the IP
    INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)printip, IARG_INST_PTR, IARG_END);
}

// This function is called when the application exits
VOID Fini(INT32 code, VOID *v)
{
    fprintf(trace, "#eof\n");
    fclose(trace);
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    PIN_ERROR("This Pintool prints the IPs of every instruction executed\n" 
              + KNOB_BASE::StringKnobSummary() + "\n");
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    trace = fopen("itrace.out", "w");
    
    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();

    // Register Instruction to be called to instrument instructions
    INS_AddInstrumentFunction(Instruction, 0);

    // Register Fini to be called when the application exits
    PIN_AddFiniFunction(Fini, 0);
    
    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Memory Reference Trace (Instruction Instrumentation)
上例是插装所有指令，但有时只想对一类指令插装，如内存操作或分支指令。
可以通过调用包含 分类和检查功能的Pin API实现，具体描述参见[这里](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/group__INS__BASIC__API.html)。此外，这是 IA-32 ISA 特有的。

此例展示如何通过检查指令，选择性插装，并生成程序引用内存的轨迹。这有助于调试程序和模拟处理器数据缓存。

本例中，只插装 读、写内存指令。由于 instrumentation函数值调用一次，而 每次执行指令时都会调用analysis函数，因此只对内存操作插装比上例中插装每条指令快得多。

运行命令和结果如下“
```
../../../pin -t obj-ia32/pinatrace.so -- /bin/ls

head pinatrace.out
```

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ezr17kh5krj30k10ijmzn.jpg)

示例程序为 source/tools/ManualExamples/pinatrace.cpp
```
/*
 *  This file contains an ISA-portable PIN tool for tracing memory accesses.
 */

#include <stdio.h>
#include "pin.H"


FILE * trace;

// Print a memory read record
VOID RecordMemRead(VOID * ip, VOID * addr)
{
    fprintf(trace,"%p: R %p\n", ip, addr);
}

// Print a memory write record
VOID RecordMemWrite(VOID * ip, VOID * addr)
{
    fprintf(trace,"%p: W %p\n", ip, addr);
}

// Is called for every instruction and instruments reads and writes
VOID Instruction(INS ins, VOID *v)
{
    // Instruments memory accesses using a predicated call, i.e.
    // the instrumentation is called iff the instruction will actually be executed.
    //
    // On the IA-32 and Intel(R) 64 architectures conditional moves and REP 
    // prefixed instructions appear as predicated instructions in Pin.
    UINT32 memOperands = INS_MemoryOperandCount(ins);

    // Iterate over each memory operand of the instruction.
    for (UINT32 memOp = 0; memOp < memOperands; memOp++)
    {
        if (INS_MemoryOperandIsRead(ins, memOp))
        {
            INS_InsertPredicatedCall(
                ins, IPOINT_BEFORE, (AFUNPTR)RecordMemRead,
                IARG_INST_PTR,
                IARG_MEMORYOP_EA, memOp,
                IARG_END);
        }
        // Note that in some architectures a single memory operand can be 
        // both read and written (for instance incl (%eax) on IA-32)
        // In that case we instrument it once for read and once for write.
        if (INS_MemoryOperandIsWritten(ins, memOp))
        {
            INS_InsertPredicatedCall(
                ins, IPOINT_BEFORE, (AFUNPTR)RecordMemWrite,
                IARG_INST_PTR,
                IARG_MEMORYOP_EA, memOp,
                IARG_END);
        }
    }
}

VOID Fini(INT32 code, VOID *v)
{
    fprintf(trace, "#eof\n");
    fclose(trace);
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */
   
INT32 Usage()
{
    PIN_ERROR( "This Pintool prints a trace of memory addresses\n" 
              + KNOB_BASE::StringKnobSummary() + "\n");
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char *argv[])
{
    if (PIN_Init(argc, argv)) return Usage();

    trace = fopen("pinatrace.out", "w");

    INS_AddInstrumentFunction(Instruction, 0);
    PIN_AddFiniFunction(Fini, 0);

    // Never returns
    PIN_StartProgram();
    
    return 0;
}
```

# 参考文献
[Pin 用户手册](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/)
