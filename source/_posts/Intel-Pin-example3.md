title: Intel Pin 2 ：示例（再续）
date: 2016-01-08 10:02:33
tags: Pin
---
继续Intel Pin示例系列。
<!-- more -->
# Using PIN_SafeCopy()
PIN_SafeCopy()将指定字节数从源内存区拷贝到目的内存区，此函数确保安全返回，即使源或目的内存区不可访问（部分或全部）。

PIN_SafeCopy()还确保应用程序的安全读写。如，在Windows中，运行分析代码时，Pin替换掉特定TEB块，若Pin tool 直接访问此块，则访问到的是修改后的值，而不是原始值。

我们推荐使用此API读写应用程序内存。
```
$ ../../../pin -t obj-ia32/safecopy.so -- /bin/cp makefile obj-ia32/safecopy.so.makefile.copy
$ head safecopy.out
Emulate loading from addr 0xbff0057c to ebx
Emulate loading from addr 0x64ffd4 to eax
Emulate loading from addr 0xbff00598 to esi
Emulate loading from addr 0x6501c8 to edi
Emulate loading from addr 0x64ff14 to edx
Emulate loading from addr 0x64ff1c to edx
Emulate loading from addr 0x64ff24 to edx
Emulate loading from addr 0x64ff2c to edx
Emulate loading from addr 0x64ff34 to edx
Emulate loading from addr 0x64ff3c to edx
```

源码在 source/tools/ManualExamples/safecopy.cpp
```
#include <stdio.h>
#include "pin.H"
#include "pin_isa.H"
#include <iostream>
#include <fstream>

std::ofstream* out = 0;

//=======================================================
//  Analysis routines
//=======================================================

// Move from memory to register
ADDRINT DoLoad(REG reg, ADDRINT * addr)
{
    *out << "Emulate loading from addr " << addr << " to " << REG_StringShort(reg) << endl;
    ADDRINT value;
    PIN_SafeCopy(&value, addr, sizeof(ADDRINT));
    return value;
}

//=======================================================
// Instrumentation routines
//=======================================================

VOID EmulateLoad(INS ins, VOID* v)
{
    // Find the instructions that move a value from memory to a register
    if (INS_Opcode(ins) == XED_ICLASS_MOV &&
        INS_IsMemoryRead(ins) && 
        INS_OperandIsReg(ins, 0) &&
        INS_OperandIsMemory(ins, 1))
    {
        // op0 <- *op1
        INS_InsertCall(ins,
                       IPOINT_BEFORE,
                       AFUNPTR(DoLoad),
                       IARG_UINT32,
                       REG(INS_OperandReg(ins, 0)),
                       IARG_MEMORYREAD_EA,
                       IARG_RETURN_REGS,
                       INS_OperandReg(ins, 0),
                       IARG_END);

        // Delete the instruction
        INS_Delete(ins);
    }
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This tool demonstrates the use of SafeCopy" << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    // Write to a file since cout and cerr maybe closed by the application
    out = new std::ofstream("safecopy.out");

    // Initialize pin & symbol manager
    if (PIN_Init(argc, argv)) return Usage();
    PIN_InitSymbols();

    // Register EmulateLoad to be called to instrument instructions
    INS_AddInstrumentFunction(EmulateLoad, 0);

    // Never returns
    PIN_StartProgram();   
    return 0;
}
```

# Order of Instrumentation
Pin提供多种方法控制分析函数的执行顺序，可以通过插入操作（IPOINT）和调用顺序函数（CALL_ORDER）实现。
下例展示了，用三种不同的方式插装所有的返回指令。另一个例子可参见 source/tools/InstrumentationOrderAndVersion.

```
../../../pin -t obj-ia32/invocation.so -- obj-ia32/little_malloc
head invocation.out
```

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ezrwrmrpt8j30k509kzlc.jpg)

源码参见 source/tools/ManualExamples/invocation.cpp
```
#include "pin.H"
#include <iostream>
#include <fstream>
using namespace std;


KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "invocation.out", "specify output file name");

ofstream OutFile;

/*
 * Analysis routines
 */
VOID Taken( const CONTEXT * ctxt)
{
    ADDRINT TakenIP = (ADDRINT)PIN_GetContextReg( ctxt, REG_INST_PTR );
    OutFile << "Taken: IP = " << hex << TakenIP << dec << endl;
}

VOID Before(CONTEXT * ctxt)
{
    ADDRINT BeforeIP = (ADDRINT)PIN_GetContextReg( ctxt, REG_INST_PTR);
    OutFile << "Before: IP = " << hex << BeforeIP << dec << endl;
}


VOID After(CONTEXT * ctxt)
{
    ADDRINT AfterIP = (ADDRINT)PIN_GetContextReg( ctxt, REG_INST_PTR);
    OutFile << "After: IP = " << hex << AfterIP << dec << endl;
}

    
/*
 * Instrumentation routines
 */
VOID ImageLoad(IMG img, VOID *v)
{
    for (SEC sec = IMG_SecHead(img); SEC_Valid(sec); sec = SEC_Next(sec))
    {
        // RTN_InsertCall() and INS_InsertCall() are executed in order of
        // appearance.  In the code sequence below, the IPOINT_AFTER is
        // executed before the IPOINT_BEFORE.
        for (RTN rtn = SEC_RtnHead(sec); RTN_Valid(rtn); rtn = RTN_Next(rtn))
        {
            // Open the RTN.
            RTN_Open( rtn );
            
            // IPOINT_AFTER is implemented by instrumenting each return
            // instruction in a routine.  Pin tries to find all return
            // instructions, but success is not guaranteed.
            RTN_InsertCall( rtn, IPOINT_AFTER, (AFUNPTR)After,
                            IARG_CONTEXT, IARG_END);
            
            // Examine each instruction in the routine.
            for( INS ins = RTN_InsHead(rtn); INS_Valid(ins); ins = INS_Next(ins) )
            {
                if( INS_IsRet(ins) )
                {
                    // instrument each return instruction.
                    // IPOINT_TAKEN_BRANCH always occurs last.
                    INS_InsertCall( ins, IPOINT_BEFORE, (AFUNPTR)Before,
                                   IARG_CONTEXT, IARG_END);
                    INS_InsertCall( ins, IPOINT_TAKEN_BRANCH, (AFUNPTR)Taken,
                                   IARG_CONTEXT, IARG_END);
                }
            }
            // Close the RTN.
            RTN_Close( rtn );
        }
    }
}

VOID Fini(INT32 code, VOID *v)
{
    OutFile.close();
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This is the invocation pintool" << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    // Initialize pin & symbol manager
    if (PIN_Init(argc, argv)) return Usage();
    PIN_InitSymbols();

    // Register ImageLoad to be called to instrument instructions
    IMG_AddInstrumentFunction(ImageLoad, 0);
    PIN_AddFiniFunction(Fini, 0);

    // Write to a file since cout and cerr maybe closed by the application
    OutFile.open(KnobOutputFile.Value().c_str());
    OutFile.setf(ios::showbase);
    
    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
/* ===================================================================== */

```

# Finding the Value of Function Arguments
RTN_insertCall()可获得函数参数和返回地址。

下例打印malloc()和free()函数的参数和malloc()函数的返回地址：
```
../../../pin -t obj-ia32/malloctrace.so -- /bin/cp makefile obj-ia32/malloctrace.so.makefile.copy
head malloctrace.out
```

![](https://ww3.sinaimg.cn/large/005CA6ZCgw1ezrwzop5gjj30k4071q3o.jpg)

源码参见 source/tools/ManualExamples/malloctrace.cpp
```
#include "pin.H"
#include <iostream>
#include <fstream>

/* ===================================================================== */
/* Names of malloc and free */
/* ===================================================================== */
#if defined(TARGET_MAC)
#define MALLOC "_malloc"
#define FREE "_free"
#else
#define MALLOC "malloc"
#define FREE "free"
#endif

/* ===================================================================== */
/* Global Variables */
/* ===================================================================== */

std::ofstream TraceFile;

/* ===================================================================== */
/* Commandline Switches */
/* ===================================================================== */

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "malloctrace.out", "specify trace file name");

/* ===================================================================== */


/* ===================================================================== */
/* Analysis routines                                                     */
/* ===================================================================== */
 
VOID Arg1Before(CHAR * name, ADDRINT size)
{
    TraceFile << name << "(" << size << ")" << endl;
}

VOID MallocAfter(ADDRINT ret)
{
    TraceFile << "  returns " << ret << endl;
}


/* ===================================================================== */
/* Instrumentation routines                                              */
/* ===================================================================== */
   
VOID Image(IMG img, VOID *v)
{
    // Instrument the malloc() and free() functions.  Print the input argument
    // of each malloc() or free(), and the return value of malloc().
    //
    //  Find the malloc() function.
    RTN mallocRtn = RTN_FindByName(img, MALLOC);
    if (RTN_Valid(mallocRtn))
    {
        RTN_Open(mallocRtn);

        // Instrument malloc() to print the input argument value and the return value.
        RTN_InsertCall(mallocRtn, IPOINT_BEFORE, (AFUNPTR)Arg1Before,
                       IARG_ADDRINT, MALLOC,
                       IARG_FUNCARG_ENTRYPOINT_VALUE, 0,
                       IARG_END);
        RTN_InsertCall(mallocRtn, IPOINT_AFTER, (AFUNPTR)MallocAfter,
                       IARG_FUNCRET_EXITPOINT_VALUE, IARG_END);

        RTN_Close(mallocRtn);
    }

    // Find the free() function.
    RTN freeRtn = RTN_FindByName(img, FREE);
    if (RTN_Valid(freeRtn))
    {
        RTN_Open(freeRtn);
        // Instrument free() to print the input argument value.
        RTN_InsertCall(freeRtn, IPOINT_BEFORE, (AFUNPTR)Arg1Before,
                       IARG_ADDRINT, FREE,
                       IARG_FUNCARG_ENTRYPOINT_VALUE, 0,
                       IARG_END);
        RTN_Close(freeRtn);
    }
}

/* ===================================================================== */

VOID Fini(INT32 code, VOID *v)
{
    TraceFile.close();
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */
   
INT32 Usage()
{
    cerr << "This tool produces a trace of calls to malloc." << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char *argv[])
{
    // Initialize pin & symbol manager
    PIN_InitSymbols();
    if( PIN_Init(argc,argv) )
    {
        return Usage();
    }
    
    // Write to a file since cout and cerr maybe closed by the application
    TraceFile.open(KnobOutputFile.Value().c_str());
    TraceFile << hex;
    TraceFile.setf(ios::showbase);
    
    // Register Image to be called to instrument functions.
    IMG_AddInstrumentFunction(Image, 0);
    PIN_AddFiniFunction(Fini, 0);

    // Never returns
    PIN_StartProgram();
    
    return 0;
}

/* ===================================================================== */
/* eof */
/* ===================================================================== */
```

# Finding Functions By Name on Windows
在Windows上通常函数名查找函数地址方法略有不同。
不同的信号量都指向相同的函数地址，最终的是检查所有信号量的名字。

下例展示了，在信号量表中查找函数名字，然后利用信号量地址查找合适的返回地址。

```
$ ..\..\..\pin -t obj-ia32\w_malloctrace.dll -- ..\Tests\obj-ia32\cp-pin.exe makefile w_malloctrace.makefile.copy
$ head *.out
Before: RtlAllocateHeap(00150000, 0, 0x94)
After: RtlAllocateHeap  returns 0x153440
After: RtlAllocateHeap  returns 0x153440
Before: RtlAllocateHeap(00150000, 0, 0x20)
After: RtlAllocateHeap  returns 0
After: RtlAllocateHeap  returns 0x1567c0
Before: RtlAllocateHeap(019E0000, 0x8, 0x1800)
After: RtlAllocateHeap  returns 0x19e0688
Before: RtlAllocateHeap(00150000, 0, 0x1a)thread begin 0

After: RtlAllocateHeap  returns 0
```

源码参见 source/tools/ManualExamples/w_malloctrace.cpp
```
/* ===================================================================== */
/* This example demonstrates finding a function by name on Windows.      */
/* ===================================================================== */

#include "pin.H"
namespace WINDOWS
{
#include<Windows.h>
}
#include <iostream>
#include <fstream>

/* ===================================================================== */
/* Global Variables */
/* ===================================================================== */

std::ofstream TraceFile;

/* ===================================================================== */
/* Commandline Switches */
/* ===================================================================== */

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "w_malloctrace.out", "specify trace file name");

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This tool produces a trace of calls to RtlAllocateHeap.";
    cerr << endl << endl;
    cerr << KNOB_BASE::StringKnobSummary();
    cerr << endl;
    return -1;
}

/* ===================================================================== */
/* Analysis routines                                                     */
/* ===================================================================== */
 
VOID Before(CHAR * name, WINDOWS::HANDLE hHeap,
            WINDOWS::DWORD dwFlags, WINDOWS::DWORD dwBytes) 
{
    TraceFile << "Before: " << name << "(" << hex << hHeap << ", "
              << dwFlags << ", " << dwBytes << ")" << dec << endl;
}

VOID After(CHAR * name, ADDRINT ret)
{
    TraceFile << "After: " << name << "  returns " << hex
              << ret << dec << endl;
}


/* ===================================================================== */
/* Instrumentation routines                                              */
/* ===================================================================== */
   
VOID Image(IMG img, VOID *v)
{
    // Walk through the symbols in the symbol table.
    //
    for (SYM sym = IMG_RegsymHead(img); SYM_Valid(sym); sym = SYM_Next(sym))
    {
        string undFuncName = PIN_UndecorateSymbolName(SYM_Name(sym), UNDECORATION_NAME_ONLY);

        //  Find the RtlAllocHeap() function.
        if (undFuncName == "RtlAllocateHeap")
        {
            RTN allocRtn = RTN_FindByAddress(IMG_LowAddress(img) + SYM_Value(sym));
            
            if (RTN_Valid(allocRtn))
            {
                // Instrument to print the input argument value and the return value.
                RTN_Open(allocRtn);
                
                RTN_InsertCall(allocRtn, IPOINT_BEFORE, (AFUNPTR)Before,
                               IARG_ADDRINT, "RtlAllocateHeap",
                               IARG_FUNCARG_ENTRYPOINT_VALUE, 0,
                               IARG_FUNCARG_ENTRYPOINT_VALUE, 1,
                               IARG_FUNCARG_ENTRYPOINT_VALUE, 2,
                               IARG_END);
                RTN_InsertCall(allocRtn, IPOINT_AFTER, (AFUNPTR)After,
                               IARG_ADDRINT, "RtlAllocateHeap",
                               IARG_FUNCRET_EXITPOINT_VALUE,
                               IARG_END);
                
                RTN_Close(allocRtn);
            }
        }
    }
}

/* ===================================================================== */

VOID Fini(INT32 code, VOID *v)
{
    TraceFile.close();
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char *argv[])
{
    // Initialize pin & symbol manager
    PIN_InitSymbols();
    if( PIN_Init(argc,argv) )
    {
        return Usage();
    }
    
    // Write to a file since cout and cerr maybe closed by the application
    TraceFile.open(KnobOutputFile.Value().c_str());
    TraceFile << hex;
    TraceFile.setf(ios::showbase);
    
    // Register Image to be called to instrument functions.
    IMG_AddInstrumentFunction(Image, 0);
    PIN_AddFiniFunction(Fini, 0);

    // Never returns
    PIN_StartProgram();
    
    return 0;
}

/* ===================================================================== */
/* eof */
/* ===================================================================== */
```

# Instrumenting Threaded Applications
下例演示了使用 ThreadStart() 和 ThreadFini() 通知回调函数。尽管 ThreadStart() 和 ThreadFini() 是在 VM 和 客户机锁 中执行，但他们依然同其他 analysis 例程竞争共享资源，使用  PIN_GetLock() 解决此问题。

注意：当运行多线程程序时，如果Pin tool回调函数中有打开文件操作，会出现死锁。
解决方法：在 main 函数中打开文件，并使用 线程ID 标记数据，具体参见 source/tools/ManualExamples/buffer_windows.cpp 
Linux中不存在此问题。

```
$ ../../../pin -t obj-ia32/malloc_mt.so -- obj-ia32/thread_lin
$ head malloc_mt.out
thread begin 0
thread 0 entered malloc(24d)
thread 0 entered malloc(57)
thread 0 entered malloc(c)
thread 0 entered malloc(3c0)
thread 0 entered malloc(c)
thread 0 entered malloc(58)
thread 0 entered malloc(56)
thread 0 entered malloc(19)
thread 0 entered malloc(25c)
```

源码参见 source/tools/ManualExamples/malloc_mt.cpp
```
#include <stdio.h>
#include "pin.H"

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "malloc_mt.out", "specify output file name");

//==============================================================
//  Analysis Routines
//==============================================================
// Note:  threadid+1 is used as an argument to the PIN_GetLock()
//        routine as a debugging aid.  This is the value that
//        the lock is set to, so it must be non-zero.

// lock serializes access to the output file.
FILE * out;
PIN_LOCK lock;

// Note that opening a file in a callback is only supported on Linux systems.
// See buffer-win.cpp for how to work around this issue on Windows.
//
// This routine is executed every time a thread is created.
VOID ThreadStart(THREADID threadid, CONTEXT *ctxt, INT32 flags, VOID *v)
{
    PIN_GetLock(&lock, threadid+1);
    fprintf(out, "thread begin %d\n",threadid);
    fflush(out);
    PIN_ReleaseLock(&lock);
}

// This routine is executed every time a thread is destroyed.
VOID ThreadFini(THREADID threadid, const CONTEXT *ctxt, INT32 code, VOID *v)
{
    PIN_GetLock(&lock, threadid+1);
    fprintf(out, "thread end %d code %d\n",threadid, code);
    fflush(out);
    PIN_ReleaseLock(&lock);
}

// This routine is executed each time malloc is called.
VOID BeforeMalloc( int size, THREADID threadid )
{
    PIN_GetLock(&lock, threadid+1);
    fprintf(out, "thread %d entered malloc(%d)\n", threadid, size);
    fflush(out);
    PIN_ReleaseLock(&lock);
}


//====================================================================
// Instrumentation Routines
//====================================================================

// This routine is executed for each image.
VOID ImageLoad(IMG img, VOID *)
{
    RTN rtn = RTN_FindByName(img, "malloc");
    
    if ( RTN_Valid( rtn ))
    {
        RTN_Open(rtn);
        
        RTN_InsertCall(rtn, IPOINT_BEFORE, AFUNPTR(BeforeMalloc),
                       IARG_FUNCARG_ENTRYPOINT_VALUE, 0,
                       IARG_THREAD_ID, IARG_END);

        RTN_Close(rtn);
    }
}

// This routine is executed once at the end.
VOID Fini(INT32 code, VOID *v)
{
    fclose(out);
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    PIN_ERROR("This Pintool prints a trace of malloc calls in the guest application\n"
              + KNOB_BASE::StringKnobSummary() + "\n");
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(INT32 argc, CHAR **argv)
{
    // Initialize the pin lock
    PIN_InitLock(&lock);
    
    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();
    PIN_InitSymbols();
    
    out = fopen(KnobOutputFile.Value().c_str(), "w");

    // Register ImageLoad to be called when each image is loaded.
    IMG_AddInstrumentFunction(ImageLoad, 0);

    // Register Analysis routines to be called when a thread begins/ends
    PIN_AddThreadStartFunction(ThreadStart, 0);
    PIN_AddThreadFiniFunction(ThreadFini, 0);

    // Register Fini to be called when the application exits
    PIN_AddFiniFunction(Fini, 0);
    
    // Never returns
    PIN_StartProgram();
    
    return 0;
}
```

# 参考文献
[Pin 用户手册](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/)

