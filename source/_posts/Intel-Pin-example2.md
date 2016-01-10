title: Intel Pin 2 ：示例（续）
date: 2016-01-07 16:53:54
tags: Pin
---
 本文是[上篇文章]()的续集。
 <!-- more -->
 
# Detecting the Loading and Unloading of Images (Image Instrumentation)
下面的例子展示，每当加载或卸载镜像时，输出访问文件轨迹。此例仅做教学，并无实际意义。

命令和结果：
```
 ../../../pin -t obj-ia32/imageload.so -- /bin/ls
 cat imageload.out
```

![](http://ww3.sinaimg.cn/large/005CA6ZCgw1ezru15kndlj30k80kogoo.jpg)

此例子在 source/tools/ManualExamples/imageload.cpp

```
//
// This tool prints a trace of image load and unload events
//

#include "pin.H"
#include <iostream>
#include <fstream>
#include <stdlib.h>

using namespace std;

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "imageload.out", "specify file name");

ofstream TraceFile;

// Pin calls this function every time a new img is loaded
// It can instrument the image, but this example does not
// Note that imgs (including shared libraries) are loaded lazily

VOID ImageLoad(IMG img, VOID *v)
{
    TraceFile << "Loading " << IMG_Name(img) << ", Image id = " << IMG_Id(img) << endl;
}

// Pin calls this function every time a new img is unloaded
// You can't instrument an image that is about to be unloaded
VOID ImageUnload(IMG img, VOID *v)
{
    TraceFile << "Unloading " << IMG_Name(img) << endl;
}

// This function is called when the application exits
// It closes the output file.
VOID Fini(INT32 code, VOID *v)
{
    if (TraceFile.is_open()) { TraceFile.close(); }
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    PIN_ERROR("This tool prints a log of image load and unload events\n"
             + KNOB_BASE::StringKnobSummary() + "\n");
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    // Initialize symbol processing
    PIN_InitSymbols();
    
    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();
    
    TraceFile.open(KnobOutputFile.Value().c_str());
    
    // Register ImageLoad to be called when an image is loaded
    IMG_AddInstrumentFunction(ImageLoad, 0);

    // Register ImageUnload to be called when an image is unloaded
    IMG_AddUnloadFunction(ImageUnload, 0);

    // Register Fini to be called when the application exits
    PIN_AddFiniFunction(Fini, 0);
    
    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# More Efficient Instruction Counting (Trace Instrumentation)
上篇文章第一个例子 Simple Instruction Count (Instruction Instrumentation) ，在每条指令前插入一条call指令，计算执行过的指令数。本节介绍一种更高效的方法，在 instrumentation阶段，计算一个BBL中的指令数，然后将每个BBL指令数累加。

命令和结果：
```
../../../pin -t obj-ia32/inscount1.so -- /bin/ls
cat inscount.out
```

![](http://ww1.sinaimg.cn/large/005CA6ZCgw1ezruetz3f1j30k30ei76c.jpg)

本例源码在 source/tools/ManualExamples/inscount1.cpp
```
#include <iostream>
#include <fstream>
#include "pin.H"

ofstream OutFile;

// The running count of instructions is kept here
// make it static to help the compiler optimize docount
static UINT64 icount = 0;

// This function is called before every block
VOID docount(UINT32 c) { icount += c; }
    
// Pin calls this function every time a new basic block is encountered
// It inserts a call to docount
VOID Trace(TRACE trace, VOID *v)
{
    // Visit every basic block  in the trace
    for (BBL bbl = TRACE_BblHead(trace); BBL_Valid(bbl); bbl = BBL_Next(bbl))
    {
        // Insert a call to docount before every bbl, passing the number of instructions
        BBL_InsertCall(bbl, IPOINT_BEFORE, (AFUNPTR)docount, IARG_UINT32, BBL_NumIns(bbl), IARG_END);
    }
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

int main(int argc, char * argv[])
{
    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();

    OutFile.open(KnobOutputFile.Value().c_str());

    // Register Instruction to be called to instrument instructions
    TRACE_AddInstrumentFunction(Trace, 0);

    // Register Fini to be called when the application exits
    PIN_AddFiniFunction(Fini, 0);
    
    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Procedure Instruction Count (Routine Instrumentation)
下例将介绍，计算程序中调用procedure数、每个procedure中执行过的指令数。结果默认保存在proccount.out中。

命令和结果：
```
$ ../../../pin -t obj-ia32/proccount.so -- /bin/grep proccount.cpp Makefile
proccount_SOURCES = proccount.cpp
$ head proccount.out
              Procedure           Image            Address        Calls Instructions
                  _fini       libc.so.6         0x40144d00            1           21
__deregister_frame_info       libc.so.6         0x40143f60            2           70
  __register_frame_info       libc.so.6         0x40143df0            2           62
              fde_merge       libc.so.6         0x40143870            0            8
            __init_misc       libc.so.6         0x40115824            1           85
            __getclktck       libc.so.6         0x401157f4            0            2
                 munmap       libc.so.6         0x40112ca0            1            9
                   mmap       libc.so.6         0x40112bb0            1           23
            getpagesize       libc.so.6         0x4010f934            2           26
$
```

源码在 source/tools/ManualExamples/proccount.cpp
```
//
// This tool counts the number of times a routine is executed and 
// the number of instructions executed in a routine
//

#include <fstream>
#include <iomanip>
#include <iostream>
#include <string.h>
#include "pin.H"

ofstream outFile;

// Holds instruction count for a single procedure
typedef struct RtnCount
{
    string _name;
    string _image;
    ADDRINT _address;
    RTN _rtn;
    UINT64 _rtnCount;
    UINT64 _icount;
    struct RtnCount * _next;
} RTN_COUNT;

// Linked list of instruction counts for each routine
RTN_COUNT * RtnList = 0;

// This function is called before every instruction is executed
VOID docount(UINT64 * counter)
{
    (*counter)++;
}
    
const char * StripPath(const char * path)
{
    const char * file = strrchr(path,'/');
    if (file)
        return file+1;
    else
        return path;
}

// Pin calls this function every time a new rtn is executed
VOID Routine(RTN rtn, VOID *v)
{
    
    // Allocate a counter for this routine
    RTN_COUNT * rc = new RTN_COUNT;

    // The RTN goes away when the image is unloaded, so save it now
    // because we need it in the fini
    rc->_name = RTN_Name(rtn);
    rc->_image = StripPath(IMG_Name(SEC_Img(RTN_Sec(rtn))).c_str());
    rc->_address = RTN_Address(rtn);
    rc->_icount = 0;
    rc->_rtnCount = 0;

    // Add to list of routines
    rc->_next = RtnList;
    RtnList = rc;
            
    RTN_Open(rtn);
            
    // Insert a call at the entry point of a routine to increment the call count
    RTN_InsertCall(rtn, IPOINT_BEFORE, (AFUNPTR)docount, IARG_PTR, &(rc->_rtnCount), IARG_END);
    
    // For each instruction of the routine
    for (INS ins = RTN_InsHead(rtn); INS_Valid(ins); ins = INS_Next(ins))
    {
        // Insert a call to docount to increment the instruction counter for this rtn
        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)docount, IARG_PTR, &(rc->_icount), IARG_END);
    }

    
    RTN_Close(rtn);
}

// This function is called when the application exits
// It prints the name and count for each procedure
VOID Fini(INT32 code, VOID *v)
{
    outFile << setw(23) << "Procedure" << " "
          << setw(15) << "Image" << " "
          << setw(18) << "Address" << " "
          << setw(12) << "Calls" << " "
          << setw(12) << "Instructions" << endl;

    for (RTN_COUNT * rc = RtnList; rc; rc = rc->_next)
    {
        if (rc->_icount > 0)
            outFile << setw(23) << rc->_name << " "
                  << setw(15) << rc->_image << " "
                  << setw(18) << hex << rc->_address << dec <<" "
                  << setw(12) << rc->_rtnCount << " "
                  << setw(12) << rc->_icount << endl;
    }

}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This Pintool counts the number of times a routine is executed" << endl;
    cerr << "and the number of instructions executed in a routine" << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    // Initialize symbol table code, needed for rtn instrumentation
    PIN_InitSymbols();

    outFile.open("proccount.out");

    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();

    // Register Routine to be called to instrument rtn
    RTN_AddInstrumentFunction(Routine, 0);

    // Register Fini to be called when the application exits
    PIN_AddFiniFunction(Fini, 0);
    
    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# 参考文献
[Pin 用户手册](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/)
