---
layout: post 
title: Android平台抓取native crash log
summary: Android Tombstones原理及如何抓取native crash log
tags: android,native crash
---

{{ page.title }}
================

Android开发中，在Java层可以方便的捕获crashlog，但对于 Native 层的 crashlog 通常无法直接获取，只能通过系统的logcat来分析crash日志。

做过 Linux 和 Win32 开发的都知道，在pc上程序crash时可以生成 core dump 文件通过相关的工具分析函数调用堆栈

那么作为软件开发者有没有方法自己获取native层的crashlog呢？Android 系统是 Linux 内核，既然在Linux中crash时可以生成dump文件，那么在Android中也是有办法的。

## Linux系统的Crash dump ##

### Linux 栈调用回溯 ###

对 Linux 应用程序而言， 因为有 glibc 库的支持， 所以构造程序的函数调用链相对容易。在 glibc 库提供的关于堆栈回朔的一系列库函数中，其核心函数是 ``` backtrace() ```。它负责遍历从程序入口点到当前调用点的所有堆栈帧，然后生成函数调用的地址序列。为了完成函数地址和函数名称的转换，函数 ``` backtrace_symbols() ``` 负责将 ``` backtrace() ```生成的地址序列转换成一系列字符串列表，在每个字符串列表中包括了函数名称，当前指令在函数中的偏移量和函数的返回地址。由于 ``` backtrace_symbols() ``` 需要动态申请空间以保存字符串列表，如果应用程序 crash 时破坏了系统内存，可能导致 backtrace_symbols()结果错误。为此，glibc库还提供了一个更安全的地址转换函数：``` backtrace_symbols_fd() ``` 。该函数将生成的字符串直接输出到外部文件，而不再需要申请新的内存空间。对于 ```backtrace() ``` 的详细使用方法可以通过 ``` man backtrace ``` 查看。

在Andrid中，由于谷歌没有使用glibc库，而是使用了精简版本的bionic库，其中并没有 ``` backtrace() ``` 可用。

### Linux 信号机制 ###

信号机制是 Linux 进程间通信的一种重要方式，Linux 信号一方面用于正常的进程间通信和同步，如任务控制(SIGINT, SIGTSTP,SIGKILL, SIGCONT，……)；另一方面，它还负责监控系统异常及中断。 当应用程序运行异常时， Linux 内核将产生错误信号并通知当前进程。
当前进程在接收到该错误信号后，可以有三种不同的处理方式。 
   1. 忽略该信号。  
   2. 捕捉该信号并执行对应的信号处理函数(signal handler)。  
   3. 执行该信号的缺省操作(如 SIGTERM， 其缺省操作是终止进程)。

当 Linux 应用程序在执行时发生严重错误，一般会导致程序 crash。其中，Linux 专门提供了一类 crash 信号，在程序接收到此类信号时，缺省操作是将 crash 的现场信息记录到 core 文件，然后终止进程。

Crash信号列表

<table class="table table-bordered table-striped table-condensed">
        <tr>
            <td>Signal</td>
            <td>Description</td>
        </tr>
        <tr>
        	<td>SIGSEGV</td>
        	<td> Invalid memory reference. </td>
        </tr>
        <tr>
        	<td>SIGBUS</td>
        	<td> Access to an undefined portion of a memory object. </td>
        </tr>
        <tr>
        	<td>SIGFPE</td>
            <td>Arithmetic operation error, like divide by zero. </td>
        </tr>
		<tr>
			<td>SIGILL </td>
			<td> Illegal instruction, like execute garbage or a privileged instruction </td>
		</tr>
		<tr>
			<td>SIGSYS</td>
			<td> Bad system call.</td>
		</tr> 
		<tr>
			<td>SIGXCPU </td>
			<td> CPU time limit exceeded. </td>
		</tr> 
		<tr>
			<td>SIGXFSZ</td>
			<td>  File size limit exceeded. </td>
		</tr> 
    </table>

### Linux 信号处理 sigaction ###  

    #include<signal.h>  
    int sigaction(int sig, struct sigaction *act , struct sigaction *oact) ;  
    
    struct sigaction{  
       void     (*sa_handler)(int);  
       void     (*sa_sigaction)(int, siginfo_t *, void *);  
       sigset_t   sa_mask;  
       int        sa_flags;  
       void     (*sa_restorer)(void);  
    }

这个函数可以:
   1. 给一个signal安装一个handler，并且在使用sigaction修改该handler之前，不用reinstall。
   2. 使用sigaction结构，该结构包含handler，其中可以指定2个handler，一个是使用sigiinfo_t等参数的handler，即支持给handler更多的参数，使其可以知道自己是被什么进程，那个用户，发来的什么信号，发来该信号的具体的原因是什么，当然要像这样，得给sigaction的sa_flags设置SA_SIGINFO标记。
   3．使用sigaction的sa_flags标记还可以指定系统调用被这个信号打断后，是直接返回，还是自动restart. 一个典型就是，一般我们不让SIGALRM信号将被打断的系统调用restart，因为SIGALARM一般本来就是用来打断一个block的调用的。
   4. 为了模仿老的signal函数的作用，实现unreliable 的类似signal的操作，可以通过给sa_flags设置SA_RESETHAND使handler不会自动reinstall,以及SA_NODEFER标记来使在本信号的handler内部，本信号不被自动block，当然如果你手动在sa_mask中指定要block本信号的话就可以将其block了。
   5. 通过使用sigaction结构中的sa_mask，可以在该handler执行的过程中，block一些信号，注意，这个mask是与我们使用sigprocmask设置的mask不同的mask，这个mask的作用范围仅限于本handler函数，而且他不会将我们用sigprocmask设置的mask取消，而仅仅是在其基础上再次将一些信号block掉，当handler结束时，系统会自动将mask恢复成以前的样子，所以这个sigaction中的sa_mask只作用本信号的handler的执行时间。

一个使用 sigaction 进行信号处理的示例：

    #include <signal.h>
     
    void sig_handler_with_arg(int sig,siginfo_t *sig_info,void *unused){……}
    
    int main(int argc,char **argv)
    {
        struct sigaction sa;  
        sigemptyset(&sa.sa_mask);
        sa.sa_sigaction = sig_handler_with_arg;
        sa.sa_flags = SA_RESETHAND;
  
 		sigaction(SIGSEGV, &sa, NULL);
 		...
     }

## Android tombstones 分析 ##

Android系统中应用出现nativecrash时，会在 ``` /data/tombstones ``` 目录下生成 tombstone_xx 的日志文件，记录了应用crash发生时的内存、寄存器、堆栈信息等。并且通过logcat将其内容输出。

Android 4.0中tombstones处理部分的源码位于 ``` /system/core/debuggerd ``` 和 ``` bonic/linker/debugger.c ``` 中。 

在 ``` bonic/linker/debugger.c ``` 中的 ``` debugger_init() ``` 中对7个Signal进行了注册处理,``` debugger_signal_handler ``` 作为信号处理函数。

    void debugger_init()
    {
      struct sigaction act;
      memset(&act, 0, sizeof(act));
      act.sa_sigaction = debugger_signal_handler;
      act.sa_flags = SA_RESTART | SA_SIGINFO;
      sigemptyset(&act.sa_mask);

      sigaction(SIGILL, &act, NULL);
      sigaction(SIGABRT, &act, NULL);
      sigaction(SIGBUS, &act, NULL);
      sigaction(SIGFPE, &act, NULL);
      sigaction(SIGSEGV, &act, NULL);
      sigaction(SIGSTKFLT, &act, NULL);
      sigaction(SIGPIPE, &act, NULL);
    }

在``` debugger_signal_handler ``` 中，通过socket client 于 /system/core/debuggerd 中的socket server进行通信，在/system/core/debuggerd中进行crash进程的分析( ``` handle_crashing_process ``` 函数中)，生成tombstones文件（``` dump_crash_report ``` 函数）。

``` unwind_backtrace_with_ptrace ``` 函数获取backtrae，通过 





