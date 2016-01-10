title: Intel Pin 4 ：示例（再续）
date: 2016-01-10 10:47:17
tags: Pin
---
继续Intel Pin示例系列。
<!-- more -->
# Using TLS
Pin提供了高效的线程本地存储（ thread local storage (TLS) ）APIs，用于创建线程特定数据。
下例展示了如何使用这些APIs。
```
$ ../../../pin -t obj-ia32/inscount_tls.so -- obj-ia32/thread_lin
$ head
Count[0]= 237993
Count[1]= 213296
Count[2]= 209223
Count[3]= 209223
Count[4]= 209223
Count[5]= 209223
Count[6]= 209223
Count[7]= 209223
Count[8]= 209223
Count[9]= 209223
```

源码参见 source/tools/ManualExamples/inscount_tls.cpp
```
#include <iostream>
#include <fstream>
#include "pin.H"

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "inscount_tls.out", "specify output file name");

PIN_LOCK lock;
INT32 numThreads = 0;
ofstream OutFile;

// Force each thread's data to be in its own data cache line so that
// multiple threads do not contend for the same data cache line.
// This avoids the false sharing problem.
#define PADSIZE 56  // 64 byte line size: 64-8

// a running count of the instructions
class thread_data_t
{
  public:
    thread_data_t() : _count(0) {}
    UINT64 _count;
    UINT8 _pad[PADSIZE];
};


// key for accessing TLS storage in the threads. initialized once in main()
static  TLS_KEY tls_key;

// function to access thread-specific data
thread_data_t* get_tls(THREADID threadid)
{
    thread_data_t* tdata = 
          static_cast<thread_data_t*>(PIN_GetThreadData(tls_key, threadid));
    return tdata;
}

// This function is called before every block
VOID PIN_FAST_ANALYSIS_CALL docount(UINT32 c, THREADID threadid)
{
    thread_data_t* tdata = get_tls(threadid);
    tdata->_count += c;
}

VOID ThreadStart(THREADID threadid, CONTEXT *ctxt, INT32 flags, VOID *v)
{
    PIN_GetLock(&lock, threadid+1);
    numThreads++;
    PIN_ReleaseLock(&lock);

    thread_data_t* tdata = new thread_data_t;

    PIN_SetThreadData(tls_key, tdata, threadid);
}


// Pin calls this function every time a new basic block is encountered.
// It inserts a call to docount.
VOID Trace(TRACE trace, VOID *v)
{
    // Visit every basic block  in the trace
    for (BBL bbl = TRACE_BblHead(trace); BBL_Valid(bbl); bbl = BBL_Next(bbl))
    {
        // Insert a call to docount for every bbl, passing the number of instructions.
        
        BBL_InsertCall(bbl, IPOINT_ANYWHERE, (AFUNPTR)docount, IARG_FAST_ANALYSIS_CALL,
                       IARG_UINT32, BBL_NumIns(bbl), IARG_THREAD_ID, IARG_END);
    }
}

// This function is called when the application exits
VOID Fini(INT32 code, VOID *v)
{
    // Write to a file since cout and cerr maybe closed by the application
    OutFile << "Total number of threads = " << numThreads << endl;
    
    for (INT32 t=0; t<numThreads; t++)
    {
        thread_data_t* tdata = get_tls(t);
        OutFile << "Count[" << decstr(t) << "]= " << tdata->_count << endl;
    }

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
    PIN_InitSymbols();
    if (PIN_Init(argc, argv)) return Usage();

    OutFile.open(KnobOutputFile.Value().c_str());

    // Initialize the lock
    PIN_InitLock(&lock);

    // Obtain  a key for TLS storage.
    tls_key = PIN_CreateThreadDataKey(0);

    // Register ThreadStart to be called when a thread starts.
    PIN_AddThreadStartFunction(ThreadStart, 0);

    // Register Instruction to be called to instrument instructions.
    TRACE_AddInstrumentFunction(Trace, 0);

    // Register Fini to be called when the application exits.
    PIN_AddFiniFunction(Fini, 0);

    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Using the Fast Buffering APIs
Pin可以处理缓冲区数据。
如果分析函数近用于加载参数到缓冲区，则可使用缓冲API，减少性能开销。
PIN_DefineTraceBuffer()定义缓冲区，线程启动时分配缓冲区，线程退出时回收缓冲区。
 INS_InsertFillBuffer()将请求数据直接写入给定缓冲区。
 当缓冲区满或线程退出时，PIN_DefineTraceBuffer()中的回调函数处理缓冲区。
 下例记录了所有访问内存指令的PC和这些指令的有效地址。
 注意：快速缓存APIs中不能使用 IARG_REG_REFERENCE, IARG_REG_CONST_REFERENCE and IARG_CONTEXT。

```
$ ../../../pin -t obj-ia32/buffer_linux.so -- obj-ia32/thread_lin
$ tail buffer.out.*.*
3263df   330108
3263df   330108
3263f1   a92f43fc
3263f7   a92f4d7d
326404   a92f43fc
32640a   a92f4bf8
32640a   a92f4bf8
32640f   a92f4d94
32641b   a92f43fc
326421   a92f4bf8
```

此例子适用于Linux平台，源码参见  source/tools/ManualExamples/buffer_linux.cpp
Windows用户参见 source/tools/ManualExamples/buffer_windows.cpp
```
/*
 * Sample buffering tool
 * 
 * This tool collects an address trace of instructions that access memory
 * by filling a buffer.  When the buffer overflows,the callback writes all
 * of the collected records to a file.
 *
 */

#include <iostream>
#include <fstream>
#include <stdlib.h>
#include <stddef.h>

#include "pin.H"
#include "portability.H"
using namespace std;

/*
 * Name of the output file
 */
KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool", "o", "buffer.out", "output file");

/*
 * The ID of the buffer
 */
BUFFER_ID bufId;

/*
 * Thread specific data
 */
TLS_KEY mlog_key;

/*
 * Number of OS pages for the buffer
 */
#define NUM_BUF_PAGES 1024


/*
 * Record of memory references.  Rather than having two separate
 * buffers for reads and writes, we just use one struct that includes a
 * flag for type.
 */
struct MEMREF
{
    ADDRINT     pc;
    ADDRINT     ea;
    UINT32      size;
    BOOL        read;
};


/*
 * MLOG - thread specific data that is not handled by the buffering API.
 */
class MLOG
{
  public:
    MLOG(THREADID tid);
    ~MLOG();

    VOID DumpBufferToFile( struct MEMREF * reference, UINT64 numElements, THREADID tid );

  private:
    ofstream _ofile;
};


MLOG::MLOG(THREADID tid)
{
    string filename = KnobOutputFile.Value() + "." + decstr(getpid_portable()) + "." + decstr(tid);
    
    _ofile.open(filename.c_str());
    
    if ( ! _ofile )
    {
        cerr << "Error: could not open output file." << endl;
        exit(1);
    }
    
    _ofile << hex;
}


MLOG::~MLOG()
{
    _ofile.close();
}


VOID MLOG::DumpBufferToFile( struct MEMREF * reference, UINT64 numElements, THREADID tid )
{
    for(UINT64 i=0; i<numElements; i++, reference++)
    {
        if (reference->ea != 0)
            _ofile << reference->pc << "   " << reference->ea << endl;
    }
}



/**************************************************************************
 *
 *  Instrumentation routines
 *
 **************************************************************************/

/*
 * Insert code to write data to a thread-specific buffer for instructions
 * that access memory.
 */
VOID Trace(TRACE trace, VOID *v)
{
    for(BBL bbl = TRACE_BblHead(trace); BBL_Valid(bbl); bbl=BBL_Next(bbl))
    {
        for(INS ins = BBL_InsHead(bbl); INS_Valid(ins); ins=INS_Next(ins))
        {
            UINT32 memoryOperands = INS_MemoryOperandCount(ins);

            for (UINT32 memOp = 0; memOp < memoryOperands; memOp++)
            {
                UINT32 refSize = INS_MemoryOperandSize(ins, memOp);

                // Note that if the operand is both read and written we log it once
                // for each.
                if (INS_MemoryOperandIsRead(ins, memOp))
                {
                    INS_InsertFillBuffer(ins, IPOINT_BEFORE, bufId,
                                         IARG_INST_PTR, offsetof(struct MEMREF, pc),
                                         IARG_MEMORYOP_EA, memOp, offsetof(struct MEMREF, ea),
                                         IARG_UINT32, refSize, offsetof(struct MEMREF, size), 
                                         IARG_BOOL, TRUE, offsetof(struct MEMREF, read),
                                         IARG_END);
                }

                if (INS_MemoryOperandIsWritten(ins, memOp))
                {
                    INS_InsertFillBuffer(ins, IPOINT_BEFORE, bufId,
                                         IARG_INST_PTR, offsetof(struct MEMREF, pc),
                                         IARG_MEMORYOP_EA, memOp, offsetof(struct MEMREF, ea),
                                         IARG_UINT32, refSize, offsetof(struct MEMREF, size), 
                                         IARG_BOOL, FALSE, offsetof(struct MEMREF, read),
                                         IARG_END);
                }
            }
        }
    }
}


/**************************************************************************
 *
 *  Callback Routines
 *
 **************************************************************************/

VOID * BufferFull(BUFFER_ID id, THREADID tid, const CONTEXT *ctxt, VOID *buf,
                  UINT64 numElements, VOID *v)
{
    struct MEMREF * reference=(struct MEMREF*)buf;

    MLOG * mlog = static_cast<MLOG*>( PIN_GetThreadData( mlog_key, tid ) );

    mlog->DumpBufferToFile( reference, numElements, tid );
    
    return buf;
}


/*
 * Note that opening a file in a callback is only supported on Linux systems.
 * See buffer-win.cpp for how to work around this issue on Windows.
 */
VOID ThreadStart(THREADID tid, CONTEXT *ctxt, INT32 flags, VOID *v)
{
    // There is a new MLOG for every thread.  Opens the output file.
    MLOG * mlog = new MLOG(tid);

    // A thread will need to look up its MLOG, so save pointer in TLS
    PIN_SetThreadData(mlog_key, mlog, tid);

}


VOID ThreadFini(THREADID tid, const CONTEXT *ctxt, INT32 code, VOID *v)
{
    MLOG * mlog = static_cast<MLOG*>(PIN_GetThreadData(mlog_key, tid));

    delete mlog;

    PIN_SetThreadData(mlog_key, 0, tid);
}


/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This tool demonstrates the basic use of the buffering API." << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}


/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */
int main(int argc, char *argv[])
{
    // Initialize PIN library. Print help message if -h(elp) is specified
    // in the command line or the command line is invalid
    if( PIN_Init(argc,argv) )
    {
        return Usage();
    }
    
    // Initialize the memory reference buffer;
    // set up the callback to process the buffer.
    //
    bufId = PIN_DefineTraceBuffer(sizeof(struct MEMREF), NUM_BUF_PAGES,
                                  BufferFull, 0);

    if(bufId == BUFFER_ID_INVALID)
    {
        cerr << "Error: could not allocate initial buffer" << endl;
        return 1;
    }

    // Initialize thread-specific data not handled by buffering api.
    mlog_key = PIN_CreateThreadDataKey(0);
   
    // add an instrumentation function
    TRACE_AddInstrumentFunction(Trace, 0);

    // add callbacks
    PIN_AddThreadStartFunction(ThreadStart, 0);
    PIN_AddThreadFiniFunction(ThreadFini, 0);

    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Finding the Static Properties of an Image 查看镜像的静态属性
在不插装的情况下，可以使用Pin检查二进制文件，用于帮助程序员查看镜像的静态属性。
下例展示了，未插装时，镜像中的指令数。
```
//
// This tool prints a trace of image load and unload events
//

#include <stdio.h>
#include <iostream>
#include "pin.H"


// Pin calls this function every time a new img is loaded
// It can instrument the image, but this example merely
// counts the number of static instructions in the image

VOID ImageLoad(IMG img, VOID *v)
{
    UINT32 count = 0;
    
    for (SEC sec = IMG_SecHead(img); SEC_Valid(sec); sec = SEC_Next(sec))
    { 
        for (RTN rtn = SEC_RtnHead(sec); RTN_Valid(rtn); rtn = RTN_Next(rtn))
        {
            // Prepare for processing of RTN, an  RTN is not broken up into BBLs,
            // it is merely a sequence of INSs 
            RTN_Open(rtn);
            
            for (INS ins = RTN_InsHead(rtn); INS_Valid(ins); ins = INS_Next(ins))
            {
                count++;
            }

            // to preserve space, release data associated with RTN after we have processed it
            RTN_Close(rtn);
        }
    }
    fprintf(stderr, "Image %s has  %d instructions\n", IMG_Name(img).c_str(), count);
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This tool prints a log of image load and unload events" << endl;
    cerr << " along with static instruction counts for each image." << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    // prepare for image instrumentation mode
    PIN_InitSymbols();

    // Initialize pin
    if (PIN_Init(argc, argv)) return Usage();

    // Register ImageLoad to be called when an image is loaded
    IMG_AddInstrumentFunction(ImageLoad, 0);

    // Start the program, never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Detaching Pin from the Application
调用PIN_Detach放弃Pin对程序的控制权，程序以原有速度执行未插装的原始代码。

源码参见 source/tools/ManualExamples/detach.cpp
```
#include <stdio.h>
#include "pin.H"
#include <iostream>

// This tool shows how to detach Pin from an 
// application that is under Pin's control.

UINT64 icount = 0;

#define N 10000
VOID docount() 
{
    icount++;

    // Release control of application if 10000 
    // instructions have been executed
    if ((icount % N) == 0) 
    {
        PIN_Detach();
    }
}
 
VOID Instruction(INS ins, VOID *v)
{
    INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)docount, IARG_END);
}

VOID ByeWorld(VOID *v)
{
    std::cerr << endl << "Detached at icount = " << N << endl;
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage()
{
    cerr << "This tool demonstrates how to detach Pin from an " << endl;
    cerr << "application that is under Pin's control" << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char * argv[])
{
    if (PIN_Init(argc, argv)) return Usage();

    // Callback function to invoke for every 
    // execution of an instruction
    INS_AddInstrumentFunction(Instruction, 0);
    
    // Callback functions to invoke before
    // Pin releases control of the application
    PIN_AddDetachFunction(ByeWorld, 0);
    
    // Never returns
    PIN_StartProgram();
    
    return 0;
}
```

# Instrumenting Child Processes
PIN_AddFollowChildProcessFunction()允许程序员在execv进程开始前定义想要执行的函数。
在命令行中使用  -follow_execv 选项插装子进程，如下：
```
$ ../../../pin -follow_execv -t obj-intel64/follow_child_tool.so -- obj-intel64/follow_child_app1 obj-intel64/follow_child_app2
```

源码参见 source/tools/ManualExamples/follow_child_tool.cpp ，使用下列命令build test文件：
```
$ make follow_child_tool.test
```

```
#include "pin.H"
#include <iostream>
#include <stdio.h>
#include <unistd.h>

/* ===================================================================== */
/* Command line Switches */
/* ===================================================================== */


BOOL FollowChild(CHILD_PROCESS childProcess, VOID * userData)
{
    fprintf(stdout, "before child:%u\n", getpid());
    return TRUE;
}        

/* ===================================================================== */

int main(INT32 argc, CHAR **argv)
{
    PIN_Init(argc, argv);

    PIN_AddFollowChildProcessFunction(FollowChild, 0);

    PIN_StartProgram();

    return 0;
}
```

# Managed platforms support
Pintools中的 RTN_IsDynamic() 校验动态生成的代码。
下例展示了 RTN_IsDynamic() 的用法，通过插装程序，计算发现并执行的指令总数，指令分为三类：原生指令，动态指令和 instructions without any known routine。

```
$ set CL_CONFIG_USE_VTUNE=True
$ set INTEL_JIT_PROFILER32=ia32\bin\pinjitprofiling.dll
$ ia32\bin\pin.exe -t source\tools\JitProfilingApiTests\obj-ia32\DynamicInsCount.dll -support_jit_api -o DynamicInsCount.out -- ..\OpenCL\Win32\Debug\BitonicSort.exe
No command line arguments specified, using default values.
Initializing OpenCL runtime...
Trying to run on a CPU
OpenCL data alignment is 128 bytes.
Reading file 'BitonicSort.cl' (size 3435 bytes)
Sort order is ascending
Input size is 1048576 items
Executing OpenCL kernel...
Executing reference...
Performing verification...
Verification succeeded.
NDRange perf. counter time 12994.272962 ms.
Releasing resources...
$ type JitInsCount.out
===============================================
Number of executed native instructions: 7631596649
Number of executed jitted instructions: 438983207
Number of executed instructions without any known routine: 12246
===============================================
Number of discovered native instructions: 870531
Number of discovered jitted instructions: 223
Number of discovered instructions without any known routine: 36
===============================================

```

源码参见  source.cpp
```
#include "pin.H"
#include <iostream>
#include <fstream>

// ==================================================================
// Global variables
// ==================================================================

UINT64 insNativeDiscoveredCount = 0;  //number of discovered native instructions
UINT64 insDynamicDiscoveredCount = 0; //number of discovered dynamic instructions
UINT64 insNoRtnDiscoveredCount = 0;   //number of discovered instructions without any known routine

UINT64 insNativeExecutedCount = 0;  //number of executed native instructions
UINT64 insDynamicExecutedCount = 0; //number of executed dynamic instructions
UINT64 insNoRtnExecutedCount = 0;   //number of executed instructions without any known routine

std::ostream * out = &cerr;

// =====================================================================
// Command line switches
// =====================================================================

KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE,  "pintool", "o", "", "specify file name for output");

// =====================================================================
// Utilities
// =====================================================================

// Print out help message.
INT32 Usage()
{
    cerr << "This tool prints out the number of native and dynamic instructions" << endl;
    cerr << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

// =====================================================================
// Analysis routines
// =====================================================================

// This function is called before every native instruction is executed
VOID InsNativeCount()
{
    ++insNativeExecutedCount;
}

// This function is called before every dynamic instruction is executed
VOID InsDynamicCount()
{
    ++insDynamicExecutedCount;
}

// This function is called before every instruction without any known routine is executed
VOID InsNoRtnCount()
{
    ++insNoRtnExecutedCount;
}

// =====================================================================
// Instrumentation callbacks
// =====================================================================

// Pin calls this function every time a new instruction is encountered
VOID Instruction(INS ins, VOID *v)
{
    RTN rtn = INS_Rtn(ins);
    if (!RTN_Valid(rtn))
    {
        ++insNoRtnDiscoveredCount;
        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)InsNoRtnCount, IARG_END);
    }
    else if (RTN_IsDynamic(rtn))
    {
        ++insDynamicDiscoveredCount;
        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)InsDynamicCount, IARG_END);
    }
    else
    {
        ++insNativeDiscoveredCount;
        INS_InsertCall(ins, IPOINT_BEFORE, (AFUNPTR)InsNativeCount, IARG_END);
    }
}

// Print out analysis results.
// This function is called when the application exits.
// @param[in]   code            exit code of the application
// @param[in]   v               value specified by the tool in the
//                              PIN_AddFiniFunction function call
VOID Fini(INT32 code, VOID *v)
{
    *out <<  "===============================================" << endl;
    *out <<  "Number of executed native instructions: " << insNativeExecutedCount << endl;
    *out <<  "Number of executed dynamic instructions: " << insDynamicExecutedCount << endl;
    *out <<  "Number of executed instructions without any known routine: " << insNoRtnExecutedCount << endl;
    *out <<  "===============================================" << endl;
    *out <<  "Number of discovered native instructions: " << insNativeDiscoveredCount << endl;
    *out <<  "Number of discovered dynamic instructions: " << insDynamicDiscoveredCount << endl;
    *out <<  "Number of discovered instructions without any known routine: " << insNoRtnDiscoveredCount << endl;
    *out <<  "===============================================" << endl;

    string fileName = KnobOutputFile.Value();
    if (!fileName.empty())
    {
        delete out;
    }
}

// The main procedure of the tool.
// This function is called when the application image is loaded but not yet started.
// @param[in]   argc            total number of elements in the argv array
// @param[in]   argv            array of command line arguments,
//                              including pin -t <toolname> -- ...
int main(int argc, char *argv[])
{
    // Initialize symbol processing
    PIN_InitSymbols();

    // Initialize PIN library. Print help message if -h(elp) is specified
    // in the command line or the command line is invalid
    if(PIN_Init(argc,argv))
    {
        return Usage();
    }

    string fileName = KnobOutputFile.Value();

    if (!fileName.empty())
    {
        out = new std::ofstream(fileName.c_str());
    }

    // Register Instruction to be called to instrument instructions
    INS_AddInstrumentFunction(Instruction, NULL);

    // Register function to be called when the application exits
    PIN_AddFiniFunction(Fini, NULL);

    // Start the program, never returns
    PIN_StartProgram();

    return 0;
}
```

# 参考文献
[Pin 用户手册](https://software.intel.com/sites/landingpage/pintool/docs/67254/Pin/html/)




