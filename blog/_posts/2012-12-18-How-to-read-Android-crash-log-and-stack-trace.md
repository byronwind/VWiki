---
layout: post 
title: 查看Android Native Crash 日志
summary: 查看Android Native代码崩溃日志
tags: Android,stack trace
---

{{ page.title }}
================

Read the original [post][1].

An Android crash in C/C++ code often generates some crash log which looks like the following. They can be seen either with "adb logcat" or found in one of the tombstones under /data/tombstones. This article briefly explains the structure of the log, how to read it, and how to translate the addresses into symbols with the [stack][2] tool.

The basic items in the log is explained below. The first is the build information in the system property "ro.build.fingerprint"

    I/DEBUG   (  730): *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** 
    I/DEBUG   (  730): Build fingerprint: 'generic/generic/generic/:1.5/...'

Then, it shows the process ID number (pid) and the thread id (tid). Knowing the PID, one can find its information with /proc/<pid> (of course, before the crash happens). In this example, the PID and TID are the same. However, if the crash happens in a child thread, the thread ID tid will be different from pid. The child thread itself is also a process in Linux (having its task_struct). The difference from a child process created by fork is that it shares address space and other data with the parent process.

    I/DEBUG   (  730): pid: 876, tid: 876  >>> /system/bin/mediaserver <<<

The following shows the signal which caused the process to abort, in this case, it's a segmentation fault. This is followed by the register values.

    I/DEBUG   (  730): signal 11 (SIGSEGV), fault addr 00000010
    I/DEBUG   (  730):  r0 00000000  r1 00016618  r2 80248f78  r3 00000000
    I/DEBUG   (  730):  r4 80248f78  r5 0000d330  r6 80248f78  r7 beaf9974
    I/DEBUG   (  730):  r8 00000000  r9 00000000  10 00000000  fp 00000000
    I/DEBUG   (  730):  ip afd020c8  sp beaf98d8  lr 8021edcd  pc 8021c630  cpsr a0000030

This is the call stack trace. #00 is the depth of the stack pointer. The "pc <addr>" is the PC address in the stack. Sometimes, the "lr" link register containing the return address is shown instead of PC. It is followed by the file containing the code. Of course, the address doesn't make much sense without resolving it to a symbol. This can be done with the "stack" tool. Download [stack][1] here.

    I/DEBUG   (  730):          #00  pc 0001c630  /system/lib/libhelixplayer.so
    I/DEBUG   (  730):          #01  pc 0001edca  /system/lib/libhelixplayer.so
    I/DEBUG   (  730):          #02  pc 0001ff0a  /system/lib/libhelixplayer.so
    I/DEBUG   (  730):          #03  pc 000214e0  /system/lib/libutils.so
    I/DEBUG   (  730):          #04  pc 0000e322  /system/lib/libmediaplayerservice.so
    ...
    I/DEBUG   (  730):          #15  pc b0001516  /system/bin/linker

The following is actually the current stack with the stack pointer address and code dump. Each line contains 4 bytes (one machine word), and the address is in ascending order. The words in the stack are mapped onto the memory region it belongs to.

    I/DEBUG   (  730): stack:
    I/DEBUG   (  730):     beaf9898  00016618  [heap]
    I/DEBUG   (  730):     beaf989c  beaf98d0  [stack]
    I/DEBUG   (  730):     beaf98a0  0000db28  [heap]
    I/DEBUG   (  730):     beaf98a4  beaf98f8  [stack]
    I/DEBUG   (  730):     beaf98b8  8021cf4d  /system/lib/libhelixplayer.so
    I/DEBUG   (  730):     beaf98bc  80248f78
    I/DEBUG   (  730): #00 beaf98d8  0000d330  [heap]
    I/DEBUG   (  730):     beaf98dc  00000000
    I/DEBUG   (  730):     beaf98e0  0000d330  [heap]
    I/DEBUG   (  730):     beaf98e4  8021edcd  /system/lib/libhelixplayer.so
    I/DEBUG   (  730): #01 beaf98e8  80248f78
    I/DEBUG   (  730):     beaf98ec  80248f78

References
--------------
Debugging Native Code: http://source.android.com/porting/debugging_native.html

 [1]: http://bootloader.wikidot.com/linux:android:crashlog
 [2]: http://bootloader.wikidot.com/local--files/linux:android:crashlog/stack