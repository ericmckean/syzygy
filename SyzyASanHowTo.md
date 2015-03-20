

# SyzyASan workflow #

The typical SyzyASan workflow is the following:
  * Instrument the target binary.
  * Run the instrumented binary.
  * Wait for a memory error to be found !
  * Report/fix the memory error.

You can get the latest version of our tools [here](http://syzygy-archive.commondatastorage.googleapis.com/builds/official/f0332d538f3eb84c2a55a3d0506a4a0b58d1c09e/binaries.zip).

# How-to #

## Do the instrumentation. ##

### Instrument chrome.dll, chrome\_child.dll and pdf.dll ###

  1. Set the following environment variables:
    * GYP\_GENERATORS=ninja
    * GYP\_DEFINES=syzyasan=1 win\_z7=1 chromium\_win\_pch=0 component=static\_library your\_other\_gyp\_defines `[1]`
  1. Do a glient sync/runhooks or just a "python build\gyp\_chromium" in order to regenerate the projects.
  1. Compile the "All\_syzygy" target ("ninja -C out\Release All\_syzygy").
  1. Extract out\Release\syzygy\chrome.7z to the location of your choice, put the symbols of the instrumented binaries (out\Release\syzygy\`*`.pdb) next to them and you're done.
  1. You can test the instrumentation by running the logger (see section below) and navigating to chrome://crash/heap-overflow, you should get a crash report in the logger's console.

`[1]` component=shared\_library and disable\_nacl=1 are not supported yet.

### Instrumentation of another binary ###

To instrument a binary you need to use the 'instrument.exe' executable provided by syzygy.

There's some restriction to this:
  * You binary should be a Win32 PE binary linked with the /PROFILE flag and statically linked to the CRT.
  * It shouldn't be [large address aware](http://msdn.microsoft.com/en-us/library/wz223b1z(v=vs.80).aspx) (you can use [editbin](http://msdn.microsoft.com/en-us/library/vstudio/d25ddyfc.aspx) to disable the large address aware flag).
  * If you are instrumenting a Chrome build, you might run out of memory if you are using a 32 bits version of Windows, we recommend that you use Windows 7/8 x64.
  * Your binary should be compiled with level function linking enabled and buffer security checks disabled.
  * The instrumenter requires to have the DIA SDK (msdia100.dll), it's installed by default with Visual Studio 2010 but we can't redistribute it. msdia100.dll is part of the C++ 2010 redistributable, but the path (C:\Program Files (x86)\Common Files\microsoft shared\VC) is not registered like with Visual Studio (C++). However, copying the msdia100.dll file to the instrumenter directory (or registering the path) should solve the problem.

The command line to do the instrumentation is the following:
```
instrument.exe --mode=asan --input-image=YourProgram.(exe/dll) --input-pdb=(optionnal)YourPdb.pdb --output-image=OutputImage.(exe/dll) --output-pdb=(optionnal)OutputPdb.pdb --debug-friendly
```

You can add the --overwrite flag to allow output files to be overwritten.

## Run the instrumented binary. ##

Before running the instrumented binary you first should copy the syzyasan\_rtl.dll file to a directory in the DLL search path for the instrumented binary (the same directory as the binary or a directory in the binary's PATH). Then you can simply start the instrumented binary from a command prompt. By default the error reports are printed into the console. If you are instrumenting chrome.dll you might be interested to use our logger to get the stack traces out of the sandboxed processes (i.e. the renderer).

### Use the logger ###

You just need to prefix your executable command line by 'agent\_logger.exe start --arguments -- '. Run 'agent\_logger.exe' to get the list of the available arguments.

### Environment variable ###

You can set the environment variable SYZYGY\_ASAN\_OPTIONS to give a command line to Asan runtime. The available arguments are:
  * quarantine\_size : The default size of the quarantine of the HeapProxy, in bytes.
  * max\_num\_frames: The max number of frames for a stack trace (default to 62).
  * compression\_reporting\_period: The number of allocations between reports of the stack trace cache compression ratio.
  * ignored\_stack\_ids : The list of ignored stack ids, we expect the value to be in hexadecimal format and separated by a semi-colon (i.e. 0xABABABAB;0x01234567;0xaabbaabb). The stack id is at the beginning of a SyzyASan error report. Keep in mind that the stack ids are relative to a particular build.
  * bottom\_frames\_to\_skip : The number of bottom frames to skip on a stack trace (default to 0).
  * exit\_on\_failure : Indicates if we should just exit on error rather than calling the error handler (default to false).
  * minidump\_on\_failure : If true, we should generate a minidump whenever an error is detected (default to false). This only works when you run under the logger.
  * no\_log\_as\_text: If set, we won't generate a textual log describing any errors. This only works when you run under the logger.
  * trailer\_padding\_size : The size of the padding added to every memory block trailer.

Example: If you want to get a minidump on failure and don't want a textual log set : "SYZYGY\_ASAN\_OPTIONS=--minidump\_on\_failure --no\_log\_as\_text" before running the instrumented binary.

## Interpret the error report. ##

Here is an example of a SyzyASan error report:

```
SyzyASAN error: heap-buffer-underflow on address 0x00B5731F (stack_id=0x0000537A)
READ of size 4 at 0x00A300A2
Backtrace:
        f [0x012F1037+15] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:90)
        e [0x012F107C+12] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:95)
        d [0x012F1097+12] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:99)
        c [0x012F10B2+12] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:103)
        b [0x012F10CD+12] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:107)
        a [0x012F10E8+12] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:111)
        main [0x012F1116+32] (c:\src\testsprojects\testasan\testasan\testasan\asan_test_program.cc:120)
        __tmainCRTStartup [0x012F1A90+267] (f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c:278)
        BaseThreadInitThunk [0x74C533AA+18]
        RtlInitializeExceptionChain [0x76FF9EF2+99]
        RtlInitializeExceptionChain [0x76FF9EC5+54]
0x00B5731F is located 1 bytes to the left of 4-bytes region [0x00B57320,0x00B57324)
previously allocated here (stack_id=0x1F3C69E0):
Backtrace:
        agent::asan::HeapProxy::Alloc [0x0F06CBD7+231] (d:\src\syzygy\src\syzygy\agent\asan\asan_heap.cc:179)
        asan_HeapAlloc [0x0F068597+167] (d:\src\syzygy\src\syzygy\agent\asan\asan_rtl_impl.cc:97)
        malloc [0x012F1872+75] (f:\dd\vctools\crt_bld\self_x86\crt\src\malloc.c:89)
Shadow bytes around the buggy address:
  0x00b57200: fb fb fb fb fb 00 fa fa
  0x00b57240: fa fa 00 00 02 fb fb fb
  0x00b57280: fb fb 00 fa fa fa fa 00
  0x00b572c0: 00 00 fb fb fb fb fb 00
=>0x00b57300: fa fa fa[fa]04 fb fb fb
  0x00b57340: fb fb fb fb 00 00 00 00
  0x00b57380: 00 00 00 00 00 00 00 00
  0x00b573c0: 00 00 00 00 00 00 00 00
  0x00b57400: 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:     fa
  Heap righ redzone:     fb
  Freed Heap region:     fd
```

You can distinguish several parts on this report:
  * The error type and and address of the memory access. The current supported errors are:
    * Use-after-free.
    * Heap-buffer-over/underflow.
    * Double-free.
  * The stack id of this bad access, it may be use to filter out this bug.
  * The mode and the size of the access.
  * The stack trace when the bad access has been made.
  * (If available) The relative position of the access in its memory block.
  * (If available) The allocation stack trace of the memory block containing this address. This stack trace might be incomplete.
  * (If available) The free stack trace of the memory block containing this address. This stack trace might be incomplete.
  * The state of the shadow memory around this location.

### Use WinDbg to get more details about the error. ###

If you run the instrumented binary under Windbg it'll automatically break when a memory error is detected. To do this prefix your command line with 'windbg -o -G -c ".ocommand ASAN" (presuming windbg is in your path).

A report similar to the previous one will be printed into the Windbg command prompt. In this report you will see a line containing 'Caller's context ("context address") and stack trace:', you can then use the command 'cxr "context address"' to switch to the context when the bad access has been made.

## Report an error ##

If you have find an error in Chrome or in SyzyASan please fill a bug report on the appropriate project.

### File a bug on SyzyASan. ###

Go to https://code.google.com/p/syzygy/issues/list and report a new issue. Use the 'Syzygy report' template. Please provide as much details as possible.

### File a bug on Chromium. ###

Go to http://crbug.com/new and fill a new bug. Paste the full SyzyASan report into it and indicate which version of Chrome (or Chromium) you are using. Please provide as much information as possible to help us to reproduce the bug. Please put syzygy-team@chromium.org in copy to this bug report.