

# I've been assigned a SyzyASAN bug, what do I do? #

## 1. Get an existing minidump if you can. ##

You will more than likely have a minidump from http://go/crash (Google internal only), or if the bug originated from ClusterFuzz.
If you haven't already, set up your symbol path and source paths. This will allow your debugger to automagically retrieve appropriate symbols and source code as you peruse the crash dump.

You can do this by running something like this in a command line shell - you only need to do this once per machine/user account.
```
mkdir %TEMP%\srcsrv
setx _NT_SOURCE_PATH "SRV*%TEMP%\srcsrv"
set _NT_SOURCE_PATH=SRV*%TEMP%\srcsrv

mkdir %TEMP%\symbols\google
mkdir %TEMP%\symbols\microsoft

setx _NT_SYMBOL_PATH "SRV*%TEMP%\symbols\google*http://chromium-browser-symsrv.commondatastorage.googleapis.com;SRV*%TEMP%\symbols\microsoft*http://msdl.microsoft.com/download/symbols"

set _NT_SYMBOL_PATH=SRV*%TEMP%\symbols\google*http://chromium-browser-symsrv.commondatastorage.googleapis.com;SRV*%TEMP%\symbols\microsoft*http://msdl.microsoft.com/download/symbols
```

We like to use Windbg

Otherwise, you'll have to try to use the information in the bug to reproduce it.

## 2. Explore the minidump or a running process. ##

If you're using a minidump you can open it directly in WinDbg and you'll immediately be placed into the frame where the exception was raised. You can get detailed information about the crash, and hopefully this will be useful in finding a repro or providing insight into a fix.

```
# Print the stack trace associated with the crash. This should start in 
# system libraries, and go through syzyasan_rtl.dll.
> kn

# Find the "asan_rtl!agent::asan::OnError" frame in the 
# previous stack trace and make it the current frame.
> .frame %FRAME_NUMBER_IN_HEX%

# Dump the error info associated with the crash.
> dt error_info

# Dump the stack trace associated with the allocation.
> dps @@(error_info->alloc_stack) L@@(error_info->alloc_stack_size)

# Dump the stack trace associated with the free.
> dps @@(error_info->free_stack) L@@(error_info->free_stack_size)

# Switch contexts to the exception record in the minidump.
> .ecxr

# Dump the stack trace associated with the ASAN error.
> kv
```

The minidump will automatically grab some regions of memory so you may have access to the actual object that was used-after-free, or other data structures. SyzyASAN will preserve the contents of the object so you may be able to glean information from it.

Note that SyzyASAN prefixes each allocated block with a header, which contains the state of the block as well as serving like a red-zone to catch underruns. You can inspect the state of the header by a command such as:
```
# Inspect the block header for the object at 0x04189bd8.
>dt asan_rtl!agent::asan::HeapProxy::BlockHeader 0x04189bd8-10
   +0x000 magic_number     : 0y110010101000000011100111 (0xca80e7)
   +0x000 state            : 0y00000000 ( 0, ALLOCATED )
   +0x004 block_size       : 0x14
   +0x008 alloc_stack      : 0x00d9f260 agent::asan::StackCapture
   +0x00c alloc_tid        : 0x1bb8
```

You can also dump the contents of memory snippets containing heap data, and use the SyzyASAN magic value (0xca80e7, or dword-aligned XXe780ca in hex dumps) to locate the block headers.
SyzyASAN will sometime report invalid access where the underlying cause is heap corruption. If you suspect this is the cause of your error report, then looking for nearby block headers and correlating their locations and lengths will often allow you to determine whether that's the fact.

## 3. Make sure you try to reproduce with a SyzyASAN build. ##

First of all, your life will be much easier if you reproduce the bug with a SyzyASAN instrumented version of Chrome. You can tell if your version of Chrome is instrumented with SyzyASAN if `dumpbin /imports` shows a dependency on `syzyasan_rtl.dll`. Also, the version number will be w.x.y.z, where z > 0.

In recent SyzyASAN builds you'll also see SyzyASAN mentioned in the version string in e.g. chrome://version, as so:
```
Google Chrome	29.0.1507.2 (Official Build 199891) canary SyzyASan
```

## 4a. Instrumenting your own build. ##

If you are unable to get ahold of the latest SyzyASAN instrumented canary you can always instrument Chromium yourself. The latest Syzygy binaries are included in `third_party\syzygy\binaries\exe`. Once you've built `chrome.dll` you can instrument it using:

```
instrument.exe --mode=asan --input-image=path\to\chrome.dll --output-image=path\to\another\chrome.dll --debug-friendly
```

Replace your original `chrome.dll` with the instrumented one, and make sure that `syzyasan_rtl.dll` is alongside it.

**NOTE:** You can't instrument Debug builds right now due to incremental linking being enabled. Easiest is to use a Release build, but if you want a Debug build you have to disable incremental linking and enable profile information (`/PROFILE`). This is not directly supported right now, but you can get this behaviour through judicious GYP file tweaking.

**NOTE:** SyzyASan doesn't support the Large Address Aware builds, you can use [editbin](http://msdn.microsoft.com/en-us/library/vstudio/d25ddyfc.aspx) to turn off this flag on chrome.exe.

## 4b. Get a build from the LKGR builder ##

You can find fresh instrumented builds of Chromium at https://commondatastorage.googleapis.com/chromium-browser-syzyasan/index.html .

The instrumented binaries (chrome.dll, chrome\_child.dll and content\_shell.exe) are located in the syzygy directory, while the original uninstrumented binaries are in the root. Just copy the module you're interested in (or both of them) with its symbols (**.pdb) to the root directory, along with the syzyasan\_rtl.dll (the SyzyASAN runtime library). After that you can run the build of Chrome under the SyzyASAN logger (also found in the syzygy directory).**

You can check that you're running an instrumented build by looking at the logger’s output: if you see something like “PID=XXXX; cmd-line=path/to/chrome.exe --optional-arguments” that means that process XXXX is using an instrumented binary.

## 5. Find a reproduction. ##

If the bug has been found by ClusterFuzz you're in luck, as there will be a nice self-contained way to reproduce it.

Otherwise it's up to you to find a way to reproduce the bug. The bug report should include information to help you do this, but there's no magic recipe here. You'll know you've got a reproduction when you see `asan_rtl` in the stack of the exception, and you see alloc/free stack traces that match the bug.

## 5. Fix it! ##

This is up to you! If we could automate this, we would!


# FAQ #

## Q. The crash stack doesn't seem to be related to the alloc/free stack, what should I do? ##

A. It’s probably a false positive, and you can ignore this bug (mark it as WontFix). SyzyASan uses a quarantine mechanism that keeps a fixed amount of blocks around after they’ve been freed (currently 16MB) in order to detect use-after-free bugs. If a wild access (really large out of bounds access, or dereference of uninitialized memory) happens to hit a quarantined block then the crash reporting machinery will bundle up the stack associated with the invalid access and the alloc and free stacks associated with the quarantined block. Due to the ‘wild’ nature of the access the alloc and free stacks may be clearly unrelated to the stack of the invalid access. However, there’s still an underlying issue: the access is still invalid, even if it not semantically related to the alloc and free stacks. This can also happen due to heap corruption (which can still occur because we are not able to instrument reads and writes outside of the chrome binaries, for example in system binaries).

## Q. Can I get a repro? ##

A. If the bug happened on a Canary/Dev build then it’s up to you to find a repro for it, as there’s no direct way to get a repro from the minidump. You can try to write a fuzzer if you think that it’ll be able to detect this bug on ClusterFuzz. If the bug was found by ClusterFuzz then you probably have a minimized test case attached directly to the bug report!

## Q. There are weird frames in the stack. ##

A. This may be an artifact of identical code folding, as is common for the simple functions/accessors. Multiple functions are mapped to the same block of code and during symbolization the first symbol matching the address is taken, which may or may not be the symbol that makes sense in the context.

## Q. This bug has only been seen once, should I really spend time on it? ##

A. Well, it’s up to you. If you consider that it’s really an extreme edge case, that can’t have any security/stability impact then it’s safe to ignore it. Sometime a one-off bug can still reveal a security/design issue (example: 290974) so it’s worth taking a few minutes to look at it. It’s also worth noting that we’ve previously seen one-timers become the top crashers of a future release…

## Q. Why have I been assigned to this bug? It’s not my fault! ##

A. We’re not trying to blame anyone when we assign bugs. We usually assign the bugs to people who seem to be familiar with the code where the crash happened (for example, they have recent commits in that area, or in the related OWNERS file). Since Chrome is such a huge project we sometimes get this wrong and may assign a bug to you incorrectly. If you know of someone more appropriate, please reassign it. If you have no idea then feel free to reassign it to the reporter.