---
layout: post
title: "linux内核模块"
description: 
category: linux-kernel
tags: [linux, driver]
mathjax: 
chart:
comments: false
---

###1. linux内核模块

linux支持将内核功能， 单独编译为内核模块( *.ko 文件), 在系统运行期间动态加载进内核执行， 大大提高了灵活性(例如， 可以将driver编译为内核模块， 在需要时才加载进内核， 不需要时则不记载，不占用内存)

###2. 内核模块模板  
  
	#include <linux/module.h>
	#include <linux/init.h>
	#include <linux/stat.h>
	#include <linux/kprobes.h>

	/* module macros */
	MODULE_LICENSE("GPL");
	MODULE_AUTHOR("xxx@xxxx.com");
	MODULE_DESCRIPTION("module description");

	/* module constructor */
	static int __init my_init(void)
	{
		pr_info("test1: enter module_init\n");
		return 0;
	}

	/* module destructor */
	static void __exit my_exit(void)
	{
		pr_info("test1: cnt = %d\n", ucnt);
	}

	/* module entry/exit macro */
	module_init(my_init);
	module_exit(my_exit);                   
    
###3. 内核模块makefile  
  
	ifneq ($(KERNELRELEASE),)

	# kbuild part of makefile
	obj-m:=test1.o

	else
	.PHONY: clean
	# normal makefile
	KDIR ?= /home/sven/Sources/qemu-linux/diy-linux/linux-src/linux-3.10.39

	default:
		$(MAKE) -C $(KDIR) M=$(shell pwd) modules

	clean:
		-rm -rf Module.symvers *.ko *.o modules.order *.mod.c

	endif
    
###4. 内核模块相关工具  
  
+ insmod	加载模块
+ rmmod		卸载模块  
+ modprobe	模块加载/卸载工具（相比insmod/rmmod能自动解决依赖）
+ modinfo	读取模块（.ko文件）信息  
+ depmod	生成模块依赖关系文件
  
modeprobe在载入模块时， 能自动载入所依赖的模块。 modeprobe是通过读取“/lib/modules/<kernel-version>/modules.dep”文件来了解模块的依赖关系的。  
  
在加载模块时（比如加载test.ko）, 执行`modprobe test.ko`是不会自动解决依赖关系， `modeprobe test`才会。  
  
depmod用于生成模块的依赖关系文件， 通常是“/lib/modules/<kernel-version>/modules.dep”  
  
###5. 内核模块的版本检查  
  
模块运行时， 对内核的依赖不仅仅只是限于内核的版本， 还包括内核的配置， 所使用的gcc编译器版本等信息， 当加载模块时检查不能通过时，内核会打印出“Invalid module format ”消息。  
  
Linux 2.6内核的linux/vermagic.h中定义了内核模块的版本字符串 VERMAGIC_STRING， 在此字符串中包括了以下的内容   

+ 内核版本号  
+ gcc 版本号  
+ SMP  
+ PREEMPT  

等信息， 生成 VERMAGIC_STRING的代码如下  
  
  
		#ifdef CONFIG_SMP   //配置了SMP
		#define MODULE_VERMAGIC_SMP "SMP "
		#else
		#define MODULE_VERMAGIC_SMP ""
		#endif
  
		#ifdef CONFIG_PREEMPT     //配置了PREEMPT      
		#define MODULE_VERMAGIC_PREEMPT "preempt "
		#else
		#define MODULE_VERMAGIC_PREEMPT ""
		#endif  

		#ifdef CONFIG_MODULE_UNLOAD  //支持module卸载
		#define MODULE_VERMAGIC_MODULE_UNLOAD "mod_unload "
		#else
		#define MODULE_VERMAGIC_MODULE_UNLOAD ""
		#endif
 
		#ifndef MODULE_ARCH_VERMAGIC  //体系结构VERMAGIC
		#define MODULE_ARCH_VERMAGIC ""
		#endif 
		
        
		#define VERMAGIC_STRING                \           
		UTS_RELEASE " "                       \                
		MODULE_VERMAGIC_SMP MODULE_VERMAGIC_PREEMPT  \       
		MODULE_VERMAGIC_MODULE_UNLOAD MODULE_ARCH_VERMAGIC \                        
		"gcc-" __stringify(__GNUC__) "." __stringify(__GNUC_MINOR__)
    
在编译内核时， 最后的“MODPOST”阶段（执行的代码在scripts/mod/modpost.c），将VERMAGIC_STRING写入到模块的“modeinfo”段中， 使用“modinfo”工具可以读出这些信息

	$ modinfo ./ntfs.ko
    
	filename:       ./ntfs.ko
	license:        GPL
	version:        2.1.30
	description:    NTFS 1.2/3.x driver - Copyright (c) 2001-2011 Anton Altaparmakov and Tuxera Inc.
	author:         Anton Altaparmakov <anton@tuxera.com>
	srcversion:     238C11A84CF26BBEF2DE1B3
	depends:        
	intree:         Y
	vermagic:       3.5.0-17-generic SMP mod_unload modversions 
    
vermagic 那一栏即为版本信息  

在你为特定版本的内核编译模块时， 要是用同样的编译系统， 内核源码树和内核配置。  
  
**关于linux内核模块版本检查的详细信息， 在 “linux-kernel” 分类中有单独的章节来详细解释**

###6. 适配不同内核版本之间的接口差异  
  
内核接口在一些版本之间变化很大， 为了让你的模块能兼容大的版本变化， 你需要在模块代码里面判断内核版本， 然后执行不同的动作， 为此，你可以借助几个宏：  
  
+ UTS_RELEASE	扩展成版本字符串， 如“2.6.10”  
+ LINUX_VERSION_CODE 	扩展成内核版本的二进制形式（每一个部分用以个字节表示），2.6.10 的编码是 132618 ( 就是, 0x02060a )    
+ KERNEL_VERSION(major,minor,release) 	生成给定版本的二进制代码， 如KERNEL_VERSION(2.6.10) 扩展成 132618
  
###7. 分析内核模块  
  
LKM 只不过是一个特殊的可执行可链接格式（Executable and Linkable Format，ELF）对象文件。通常，必须链接对象文件才能在可执行文件中解析它们的符号和结果。由于必须将 LKM 加载到内核后 LKM 才能解析符号，所以 LKM 仍然是一个 ELF 对象。您可以在 LKM 上使用标准对象工具（在 2.6 版本中，内核对象带有后缀 .ko,）。例如，如果在 LKM 上使用 objdump 实用工具，您将发现一些熟悉的区段（section），比如 .text（说明）、.data（已初始化数据）和 .bss（块开始符号或未初始化数据）。
您还可以在模块中找到其他支持动态特性的区段。.init.text 区段包含 module_init 代码，.exit.text 区段包含 module_exit 代码。.modinfo 区段包含各种表示模块许可证、作者和描述等的宏文本。  
  
####7.1 内核模块的加载过程  
  
在用户空间中，insmod（插入模块）启动模块加载过程。insmod 命令定义需要加载的模块，并调用 init_module 用户空间系统调用，开始加载过程。2.6 版本内核的 insmod 命令经过修改后变得非常简单（70 行代码），可以在内核中执行更多工作。insmod 并不进行所有必要的符号解析（处理 kerneld），它只是通过 init_module 函数将模块二进制文件复制到内核，然后由内核完成剩余的任务。  
  
打印内核模块的加载过程的栈回溯：  
  
	static int __init my_init(void)
	{
		pr_info("test1: enter module_init\n");
		dump_stack();
		return 0;
	}

	static void __exit my_exit(void)
	{
		pr_info("test1: enter module_exit\n");
	}

	module_init(my_init);
	module_exit(my_exit);

加载模块后， 输出的call Trace：  
  
	callTrace：
	dump_stack
	my_init
	do_one_initcall
	load_module
	? free_notes_attrs
	? SyS_init_module
	SyS_init_module
	system_call_fastpath  
        
init_module 函数通过系统调用层，进入内核到达内核函数 sys_init_module(), 这是加载模块的主要函数，它利用许多其他函数完成困难的工作。  
  
在 sys_init_module（） 里面做的工作如下：  
  
1. 权限检查  
2. 调用 load_module() 来解析模块， 这里面做了主要的工作    
3. 将模块添加到系统中保存所有模块的双向链表中  
4. 通过 notifier 列表通知正在等待模块状态改变的线程  
5. 调用模块初始化函数 
6. 将模块的状态改为 MODULE_STATE_LIVE  
7. return  
  
**需要注意的是， 在模块的入口函数成功返回之前， 模块还并未完成加载， 因此， 在模块的入口函数中执行很多操作都会导致出错**

在 load_module() 里面做的工作如下  
  
1. 分配一块用于容纳整个模块的临时内存  
2. 通过copy_from_user 函数将 ELF 模块从用户空间读入到临时内存  
3. 检查模块的ELF文件完整性  
4. 然后会为每个区段头创建一组方便变量，简化随后的访问。因为 ELF 对象的偏移量是基于 0 的（除非重新分配），所以这些方便变量将相对偏移量包含到临时内存块中。在创建方便变量的过程中还会验证 ELF 区段头，确保加载的是有效模块。  
5. 任何可选的模块参数都从用户空间加载到另一个已分配的内核内存块
6. 模块的状态改为 MODULE_STATE_COMMING  
7. 如果需要 per-CPU 数据（这在检查区段头时确定），那么就分配 per-CPU 块。
8. 分配模块所需的最终内存  
9. 将必须的区段（ELF 头中的 SHF_ALLOC，或在执行期间占用内存的区段）从临时内存移动到最终内存中  
10. 解析符号，可以解析位于内核中的符号（被编译成内核映象），或临时的符号（从其他模块导出）。
11. 然后为每个剩余的区段迭代新的模块并执行重新定位。这个步骤与架构有关，因此依赖于为架构（./linux/arch/&lt;arch&gt;/kernel/module.c）定义的 helper 函数。最后，刷新指令缓存（因为使用了临时 .text 区段），执行一些额外的维护（释放临时模块内存，设置系统文件），并将模块最终
12. 返回到 load_module。

####7.2 模块的卸载过程  
  
卸载模块过程首先在用户空间调用 rmmod（删除模块）命令。在 rmmod 命令内部，对 delete_module 执行系统调用，它最终会导致在内核内部调用 sys_delete_module。  
  
在模块的退出函数中打印栈回溯消息如下：  
  
	call Trace ：
	dump_stack
	my_exit
	sys_delete_module
	system_call_fastpath  
        
在sys_delete_module中所做的工作如下：  
  
1. 检查权限
2. 检查该模块是否被依赖  
3. 检查模块是否存在  
4. 检查模块的状态是否为MODULE_STATE——LIVE  
5. 调用模块的退出函数  
6. 调用free_module()

调用free_module() 安全删除内核模块时所做的工作 ：  
  
1. 移除 sysfs 下的项 （因该是/sys/modules/xxx下的， 自己添加的项应该自己负责清理）
2. 移除模块的 kernel object
3. 调用架构相关的清理例程（可以在 ./linux/arch/&lt;arch&gt;/kernel/module.c 中找到）
4. 清理模块的依赖关系  
5. 释放模块参数和 per-cpu 数据的内存  
6. 释放ELf的内存（core和init）  
