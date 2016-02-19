---
layout: post
title: "linux kernel debug 方法"
description: 
category: linux-kernel
tags: [linux, debug, driver]
mathjax: 
chart:
comments: false
---

###1. 内核调试选项  
  
在编译内核的时候，为了方便调试和测试代码，内核提供了许多配置选项， 如

1. Page alloc debugging ：CONFIG_DEBUG_PAGEALLOC
   + 不使用该选项时，释放的内存页将从内核地址空间中移出。使用该选项后，内核推迟移 出内存页的过程，因此能够发现内存泄漏的错误。  
2. Debug memory allocations ：CONFIG_DEBUG_SLAB
   + 该打开该选项时，在内核执行内存分配之前将执行多种类型检查，通过这些类型检查可 以发现诸如内核过量分配或者未初始化等错误。内核将会在每次分配内存前后时设置一些警戒值，如果这些值发生了变化那么内核就会知道内存已经被操作过并给出明确的提示，从而使各种隐晦的错误变得容易被跟踪。  
3. Spinlock debugging ：CONFIG_DEBUG_SPINLOCK
   + 打开此选项时，内核将能够发现spinlock未初始化及各种其他的错误,能用于排除一些死锁引起的错误。  
4. Sleep-inside-spinlock checking：CONFIG_DEBUG_SPINLOCK_SLEEP
   + 打开该选项时，当spinlock的持有者要睡眠时会执行相应的检查。实际上即使调用者目前没有睡眠，而只是存在睡眠的可能性时也会给出提示。    
5. Compile the kernel with debug info ：CONFIG_DEBUG_INFO  
   + 打开该选项时，编译出的内核将会包含全部的调试信息，使用gdb时需要这些调试信息。  
6. Stack utilization instrumentation ：CONFIG_DEBUG_STACK_USAGE  
   + 该选项用于跟踪内核栈的溢出错误，一个内核栈溢出错误的明显的现象是产生oops错 误却没有列出系统的调用栈信息。该选项将使内核进行栈溢出检查，并使内核进行栈使用的统计。  
7. Driver Core verbose debug messages：CONFIG_DEBUG_DRIVER  
   + 该选项位于"Device drivers-> Generic Driver Options"下，打开该选项使得内核驱动核心产生大量的调试信息，并将他们记录到系统日志中。  
8. Verbose SCSI error reporting (kernel size +=12K) ：CONFIG_SCSI_CONSTANTS
   + 该选项位于"Device drivers/SCSI device support"下。当SCSI设备出错时内核将给出详细的出错信息。  
9. Event debugging：CONFIG_INPUT_EVBUG
   + 打开该选项时，会将输入子系统的错误及所有事件都输出到系统日志中。该选项在产生了详细的输入报告的同时，也会导致一定的安全问题。  

以上内核编译选项需要读者根据自己所进行的内核编程的实际情况，灵活选取。在使用以上介绍的三种源代码级的内核调试工具时，一般需要选取CONFIG_DEBUG_INFO选项，以使编译的内核包含调试信息。配置内核时，在内核调试选项菜单里有更多的选项可以选择。  
  
###2. 使用 BUG() 和 BUG_ON() 宏  
  
	#ifndef HAVE_ARCH_BUG
	#define BUG() do { 
		printk("BUG: failure at %s:%d/%s()! ", __FILE__, __LINE__, __FUNCTION__); 
		panic("BUG!");   /* 引发更严重的错误，不但打印错误消息，而且整个系统业会挂起 */
	} while (0)
	#endif
    
	#ifndef HAVE_ARCH_BUG_ON
	#define BUG_ON(condition) do { if (unlikely(condition)) BUG(); } while(0)
	#endif
    
他们会引发oops导致栈的回溯，并打印出错信息  
  
###3. dump_stack()  
  
有时候， 你只是需要打印一下内核函数调用流程， 可以使用 dump_stack() 打印调用栈。  
  
###4. kallsyms  
  
开发版2.5内核引入了kallsyms特性，它可以通过定义CONFIG_KALLSYMS编译选项启用。该选项可以载入内核镜像所对应的内存地址的符号名称（即函数名），所以内核可以打印解码之后的跟踪线索。相应，解码OOPS也不再需要System.map和ksymoops工具了。另外，这样做，会使内核变大些，因为地址对应符号名称必须始终驻留在内核所在内存上。  

    #cat /proc/kallsyms
     c0100240   T       _stext
     c0100240   t       run_init_process
     c0100240   T      stext
     c0100269   t       init

###5. system.map

System.map是一个特定内核的符号表，符号表时所有符号和其对应地址的列表，当内核编译出错时， 通过System.map中的符号表解析， 就可以找到出错的地址值找到对应的变量名，或者函数名。

System.map 是独一无二的，只要内核发生了修改， 再次编译时会产生一个全新的内核符号表。你应该使用正确的符号表， 错误的符号表不仅无助于解决问题， 反而会使问题变得隐蔽。

####5.1 System.map是怎么产生的：

你知道`nm`命令吗，对编译后的内核（vmlinux）使用它，你会得到所有符号的信息， 例如：
	
    nm vmlinux | grep -v '(compiled)|(.o$$)|( [aUw] )|(..ng$$)|(LASH[RL]DI)' | sort > System.map  
    cp /usr/src/linux/System.map /boot/System.map-2.4.7-10
    
虽然说内核并不真正使用System.map文件，但是其它的一些引用需要一个正确的System.map文件，必须确保那些应用能够找到符号表, 如果没有直接为klogd指定System.map文件的位置，他将按照顺序在如下位置寻找：

+ /boot/System.map
+ /System.map
+ /usr/src/linux/System.map  

####5.2 System.map内容中各列的意义：
使用`sudo cat /boot/System.map`显示内核符号表的内容并截取其中一部分：

	0000000000013840 D cpu_info
	0000000000013900 d hv_clock
	0000000000013940 D cpu_tlbstate
	0000000000013980 d gcwq_nr_running
	00000000000139c0 D runqueues
	0000000000014400 d sched_clock_data
	0000000000014440 d call_single_queue
	0000000000014480 d cfd_data
	0000000000014500 d csd_data
	0000000000014540 D softnet_data
	0000000000014680 D __per_cpu_end
	0000000001000000 A phys_startup_64

上面的三列中， 第一列为符号对应的地址， 第二列为符号的类别， 第三列为符号对应的地址。
符号的类别使用一个字母来代表， 小写字母表示局部符号，大写字母表示全局符号，各字母的含义如下：  

+ A The symbol's value is absolute, and will not be changed by further linking.  
+ B The symbol is in the uninitialized data section (known as BSS).  
+ C The symbol is common. Common symbols are uninitialized data. When linking, multiple common symbols may appear with the same name. If the symbol is defined anywhere, the common symbols are treated as undefined references. For more details on common symbols, see the discussion of -warn-common in Linker options.  
+ D The symbol is in the initialized data section.  
+ G The symbol is in an initialized data section for small objects. Some object file formats permit more efficient access to small data objects, such as a global int variable as opposed to a large global array.  
+ I The symbol is an indirect reference to another symbol. This is a GNU extension to the a.out object file format which is rarely used.  
+ N The symbol is a debugging symbol.  
+ R The symbol is in a read only data section.  
+ S The symbol is in an uninitialized data section for small objects.  
+ T The symbol is in the text (code) section.  
+ U The symbol is undefined.  
+ V The symbol is a weak object. When a weak defined symbol is linked with a normal defined symbol, the normal defined symbol is used with no error. When a weak undefined symbol is linked and the symbol is not defined, the value of the weak symbol becomes zero with no error.  
+ W The symbol is a weak symbol that has not been specifically tagged as a weak object symbol. When a weak defined symbol is linked with a normal defined symbol, the normal defined symbol is used with no error. When a weak undefined symbol is linked and the symbol is not defined, the value of the weak symbol becomes zero with no error.  
+ —— The symbol is a stabs symbol in an a.out object file. In this case, the next values printed are the stabs other field, the stabs desc field, and the stab type. Stabs symbols are used to hold debugging information. For more information, see Stabs.  
+ ? The symbol type is unknown, or object file format specific.
