---
layout: post
title: "linux内核模块入/出口函数"
description: 
category: linux-kernel
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux 内核模块入口

在编写linux内核模块时(无论是built-in， 还是built module)， 通常使用“module_init()”宏来指定模块的入口函数，使用“module_exit()”宏来指定模块的出口函数， 例如

	#include <linux/init.h>
	#include <linux/module.h>
	#include <linux/kernel.h>

	static int __init hello_init(void)
	{
    		printk(KERN_INFO "sven : init\n");
    		return 0;
	}

	static void __exit hello_exit(void)
	{
    		printk(KERN_INFO "sven : exit\n");
	}

	module_init(hello_init);
	module_exit(hello_exit);

使用`module_init(hello_init)`指明了模块的入口函数为 hello_init(), (函数前的_init修饰符用于指定将函数的代码放置在 “.init.text” section中， **注意， 只是存放在对应的中间文件的“.init.text” section中， 而不是最终的vmlinux的“.init.text” section中，因为在链接时， 可能会合并一些section**， 在调用完后，将释放该section以释放内存)， 使用module_exit()宏指定模块的出口函数为hello_exit()

###2. built-in时内核模块更高级别的入口函数

built module 的内核模块只能够使用 module_init()宏来声明入口函数， 在insmod模块时， 入口函数被调用， 但是， 对于built-in的模块， 某一些功能模块需要更早的加载， 以便给后续的模块提供支持， 例如文件系统， 网络协议栈等， 对于built-in的内核模块， linux kernel 提供了不同的入口等级， 在"linux/init.h" 中有

	#define early_initcall(fn)		__define_initcall(fn, early)

	#define pure_initcall(fn)		__define_initcall(fn, 0)

	#define core_initcall(fn)		__define_initcall(fn, 1)
	#define core_initcall_sync(fn)	__define_initcall(fn, 1s)
	#define postcore_initcall(fn)	__define_initcall(fn, 2)
	#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
	#define arch_initcall(fn)		__define_initcall(fn, 3)
	#define arch_initcall_sync(fn)	__define_initcall(fn, 3s)
	#define subsys_initcall(fn)		__define_initcall(fn, 4)
	#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
	#define fs_initcall(fn)		__define_initcall(fn, 5)
	#define fs_initcall_sync(fn)	__define_initcall(fn, 5s)
	#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
	#define device_initcall(fn)		__define_initcall(fn, 6)
	#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
	#define late_initcall(fn)		__define_initcall(fn, 7)
	#define late_initcall_sync(fn)	__define_initcall(fn, 7s)

按照从上往下的顺序，越往下， 调用的时间越靠后， 注意其中的rootfs_initcall()， 用于初始化initramfs等， 为初始化根文件系统做准备， 对于module_init(), 有

	#define module_init(x)	__initcall(x);
	#define __initcall(fn) device_initcall(fn)

可见， module_init 的级别为6

另外还有两个声明入口的宏,  这两种入口的调用时机比以上所有的入口都要早， 用于初始化console和安全机制(例如smack， 类似于SELinux)，因为此时系统中大部分功能都未ready， 一般的模块请勿使用

	console_initcall(fn)
	security_initcall(fn)

###3. built-in时内核模块不同的等级入口的实现

以module_init()为例来说明

	#define __define_initcall(fn, id) \
		static initcall_t __initcall_##fn##id __used \
		__attribute__((__section__(".initcall" #id ".init"))) = fn

	#define device_initcall(fn)		__define_initcall(fn, 6)
	#define __initcall(fn) device_initcall(fn)

	#define module_init(x)	__initcall(x);

宏定义展开可得

	#define module_init(fn)	\
		static initcall_t __initcall##fn##6 __used	\
		__attribute__((__section__(".initcall6.init))) = fn

又有
	typedef int (*initcall_t)(void);

因此， module_init() 宏最终生成了一个静态的函数指针变量， 并且存储在 “.initcall6.init” section 中 (module_init的等级为6， 所以是“.initcall6.init”)


例如展开`module_init(hello_init)`可得

	static initcall_t __initcallhello_init6 __used __attribute__((__section__(".initcall6.init“))) = hello_init;

即定义了一个函数指针__initcallhello_init6， 指向hello_init函数， 该函数指针存放在 ".initcall6.init“ section中， 而“\_\_used\_\_”修饰符起始是一个gcc属性

	#define __used			__attribute__((__used__))

\_\_used\_\_ 告诉编译器无论 GCC 是否发现这个函数的调用实例，都要使用这个函数

对于使用其它等级来声明的入口函数的指针，， 则会被存放在对应的section中， 例如 “.initcallearly.init”， “.initcall4s.init”， “.initcallrootfs.init” 等

对于console_initcall() 和 security_initcall(), 有

	#define console_initcall(fn) \
		static initcall_t __initcall_##fn \
		__used __section(.con_initcall.init) = fn

	#define security_initcall(fn) \
		static initcall_t __initcall_##fn \
		__used __section(.security_initcall.init) = fn

即分别在 “.con_initcall.init” 和 “.security_initcall.init” section中存放入口函数指针

####3.1 built-in时内核模块入口函数指针的调用时机

上面的分析过程可以看出， 最终在 ".initcall6.init“ section 中保存了kernel module的入口函数的指针， 以x86为例， 在

在"include/asm-generic/vmlinux.lds.h"中

	#define INIT_CALLS_LEVEL(level)						\
		VMLINUX_SYMBOL(__initcall##level##_start) = .;		\
		*(.initcall##level##.init)				\
		*(.initcall##level##s.init)		

	#define INIT_CALLS							\
		VMLINUX_SYMBOL(__initcall_start) = .;			\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					\
		INIT_CALLS_LEVEL(1)					\
		INIT_CALLS_LEVEL(2)					\
		INIT_CALLS_LEVEL(3)					\
		INIT_CALLS_LEVEL(4)					\
		INIT_CALLS_LEVEL(5)					\
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					\
		INIT_CALLS_LEVEL(7)					\
		VMLINUX_SYMBOL(__initcall_end) = .;

	#define INIT_DATA_SECTION(initsetup_align)				\
		.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {		\
		......
		INIT_CALLS						\
		......
	}

“source/arch/x86/kernel/vmlinux.lds.S”中有

	SECTIONS {
		......
		INIT_DATA_SECTION(16)
		......
	}
	
适当的展开后可得

	SECTIONS {
		......
		.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {

			......

			VMLINUX_SYMBOL(__initcall_start) = .;
			*(.initcallearly.init)
		
			VMLINUX_SYMBOL(__initcall0_start) = .;
			*(.initcall##level0.init)
			*(.initcall##level0s.init)

			VMLINUX_SYMBOL(__initcall1_start) = .;
			*(.initcall##level1.init)
			*(.initcall##level1s.init)

			VMLINUX_SYMBOL(__initcall2_start) = .;
			*(.initcall##level2.init)
			*(.initcall##level2s.init)

			......

			VMLINUX_SYMBOL(__initcall7_start) = .;
			*(.initcall##level7.init)
			*(.initcall##level7s.init)

			VMLINUX_SYMBOL(__initcall_end) = .;

			......
		}
		......
	}

这一段ld链接脚本定义了如何生成vmlinux 中的 “init.data” section， 先来看VMLINUX_SYMBOL宏

	#define __VMLINUX_SYMBOL(x) x
	#define VMLINUX_SYMBOL(x) __VMLINUX_SYMBOL(x)

那么 `VMLINUX_SYMBOL(__initcall_start) = .;`展开后为

	__initcall_start = .;

在ld脚本语法中， 这一行的意义是：定义一个符号  __initcall_start， 并且和地址“.”绑定 (“.”是一个特殊的符号， 代表当前位置加载到内存中后的地址)，这个符号在程序中可以声明为extern变量， 直接使用， 再来看接下来的一句

	*(.initcallearly.init)

意义是将 “.initcallearly.init” section添加到最终的 “init.data” section中

最终， 在vmlinux的 “init.data” section中， 一次包含了如下的section （不同等级的入口函数指针放置在不同的section中）：

+ “.initcallearly.init”
+ ”.initcall0.init“
+ ”.initcall0s.init“
+ ”.initcall1.init“
+ ”.initcall1s.init“
+ ”.initcall2.init“
+ ”.initcall2s.init“
+ ”.initcall3.init“
+ ”.initcall3s.init“
+ ”.initcall4.init“
+ ”.initcall4s.init“
+ ”.initcall5.init“
+ ”.initcall5s.init“
+ ”.initcall6.init“
+ ”.initcall6s.init“
+ ”.initcall7.init“
+ ”.initcall7s.init“

并且还定义了如下的变量

+ __initcall_start : 存储了“.initcallearly.init” section加载进内存后的地址
+ __initcall0_start ： 存储了“.initcall0.init” section加载进内存后的地址
+ __initcall1_start ： 存储了“.initcall1.init” section加载进内存后的地址
+ __initcall2_start ： 存储了“.initcall2.init” section加载进内存后的地址
+ __initcall3_start ： 存储了“.initcall3.init” section加载进内存后的地址
+ __initcall4_start ： 存储了“.initcall4.init” section加载进内存后的地址
+ __initcall5_start ： 存储了“.initcall5.init” section加载进内存后的地址
+ __initcall6_start ： 存储了“.initcall6.init” section加载进内存后的地址
+ __initcall7_start ： 存储了“.initcall7.init” section加载进内存后的地址
+ __initcall_end ： 存储了“.initcall7s.init” section加载进内存后的结尾的地址

**通过上述的这些变量， 可以获取所有的built-in的 module的入口函数的指针**

在linux源码的 “init/main.c” 声明了这些变量
	
	extern initcall_t __initcall_start[];
	extern initcall_t __initcall0_start[];
	extern initcall_t __initcall1_start[];
	extern initcall_t __initcall2_start[];
	extern initcall_t __initcall3_start[];
	extern initcall_t __initcall4_start[];
	extern initcall_t __initcall5_start[];
	extern initcall_t __initcall6_start[];
	extern initcall_t __initcall7_start[];
	extern initcall_t __initcall_end[];

	static initcall_t *initcall_levels[] __initdata = {
		__initcall0_start,
		__initcall1_start,
		__initcall2_start,
		__initcall3_start,
		__initcall4_start,
		__initcall5_start,
		__initcall6_start,
		__initcall7_start,
		__initcall_end,
	};

而在linux kernel的初始化过程中

	start_kernel()
		rest_init()
			kernel_thread(kernel_init)
				kernel_init_freeable()
					do_basic_setup()
						do_initcalls()

	static void __init do_initcalls(void)
	{
		int level;

		for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
			do_initcall_level(level);
	}

	static void __init do_initcall_level(int level)
	{
		......
		for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
			do_one_initcall(*fn);
	}

即依次遍历每一个level中的每一个函数指针， 分别调用这些函数， 也即分别调用了不同的模块的入口函数

####3.2 onsole_initcall() 和 security_initcall() 的函数指针调用的时机

再来看console_initcall() 和 security_initcall()， 它们的函数指针的分别存储在“.con_initcall.init”和“.security_initcall.init” section中， 而在"include/asm-generic/vmlinux.lds.h"中

	#define CON_INITCALL							\
		VMLINUX_SYMBOL(__con_initcall_start) = .;		\
		*(.con_initcall.init)					\
		VMLINUX_SYMBOL(__con_initcall_end) = .;

	#define SECURITY_INITCALL						\
		VMLINUX_SYMBOL(__security_initcall_start) = .;		\
		*(.security_initcall.init)				\
		VMLINUX_SYMBOL(__security_initcall_end) = .;

	#define INIT_DATA_SECTION(initsetup_align)				\
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {		\
		......
		INIT_CALLS						\
		CON_INITCALL						\
		SECURITY_INITCALL					\
		......
	}

而在 “source/arch/x86/kernel/vmlinux.lds.S”中有

	SECTIONS {
		......
		INIT_DATA_SECTION(16)
		......
	}

适当的展开后有

	SECTIONS {
		......
		.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {
			......

			__con_initcall_start = .;
			*(.con_initcall.init)
			__con_initcall_end = .;

			__security_initcall_start =. ;
			*(.security_initcall.init)
			__security_initcall_end = .;

			......
		}
		......
	}

即在linux kerne中通过变量 __con_initcall_start 和 __con_initcall_end 可以访问 “.con_initcall.init” section中的函数指针， 而通过变量 __security_initcall_start 和 __security_initcall_end 可以访问 “.security_initcall.init” section中的函数指针， 在“linux/init.h”中声明了这几个外部变量

	extern initcall_t __con_initcall_start[], __con_initcall_end[];
	extern initcall_t __security_initcall_start[], __security_initcall_end[];

在 linux kernel的初始化过程中， 有

	start_kernel()
		......
		console_init()
		......
		security_init()
		......
		rest_init()

在console_init()中会依次调用 ”.con_initcall.init“ section中的函数指针， security_init()中会依次调用 “.security_initcall.init” section中的函数指针， 最后的 rest_init()才完成调用其它等级的入口函数

###4. built-in时内核模块出口函数的实现

**直接编译进内核的模块的出口函数(清理函数)会被忽略掉， 因为编译进内核的模块不许要做清理**， 但是我们还是要来了解一下它的实现

在 "linux/init.h" 中有

	#define __exit_call	__used __section(.exitcall.exit)

	typedef void (*exitcall_t)(void);
	#define __exitcall(fn) 	static exitcall_t __exitcall_##fn __exit_call = fn
	#define module_exit(x)	__exitcall(x);

即根据出口函数的名称，  定义了一个静态的函数指针变量， 指向出口函数， 例如， 出口函数为 hello_exit()时有

	static void (*__exitcall_hello_exit)(void)  __used __section(.exitcall.exit) = hello_exit;

该函数指针变量被放置在 “.exitcall.exit” section中， “__used” 修饰符指出该符号可能不会被使用， 编译器不要给出警告

而在 "include/asm-generic/vmlinux.lds.h" 中有

	#define EXIT_TEXT							\
		*(.exit.text)							\
		DEV_DISCARD(exit.text)						\
		CPU_DISCARD(exit.text)						\
		MEM_DISCARD(exit.text)

	#define DISCARDS							\
		/DISCARD/ : {							\
		EXIT_TEXT							\
		EXIT_DATA							\
		EXIT_CALL							\
		*(.discard)							\
		*(.discard.*)							\
	}

在对应的vmlinux链接脚本文件 vmlinux.lds.S 中有

	SECTIONS		{

		......

		/* Sections to be discarded */
    		DISCARDS
    		/DISCARD/ : { *(.eh_frame) }
	}

适当展开后有

	SECTIONS		{

		......

		/* Sections to be discarded */
    		/DISCARD/ : {
			*(.exit.text)
			......
		}
    		/DISCARD/ : { *(.eh_frame) }
	}

**在ld链接脚本的语法中，有一个特殊的输出section，名为/DISCARD/，被该section引用的任何输入section将不会出现在输出文件内， 即在最终链接生成的vmlinux中， ".exit.text" section直接被丢弃掉**

###5. built module时内核模块入口的实现

需要注意的是，在build为模块时， 不论以何种其它的等级来声明模块的入口参数， 最终都会被替换为 module_init()

	#ifndef	MODULE
		#define early_initcall(fn)		__define_initcall(fn, early)
		#define pure_initcall(fn)		__define_initcall(fn, 0)
		....
	#else
		#define early_initcall(fn)		module_init(fn)
		#define core_initcall(fn)		module_init(fn)
		#define postcore_initcall(fn)	module_init(fn)
		#define arch_initcall(fn)		module_init(fn)
		#define subsys_initcall(fn)		module_init(fn)
		#define fs_initcall(fn)		module_init(fn)
		#define device_initcall(fn)		module_init(fn)
		#define late_initcall(fn)		module_init(fn)
		......
		#define module_init(initfn)					\
			static inline initcall_t __inittest(void)		\
			{ return initfn; }					\
			int init_module(void) __attribute__((alias(#initfn)));
		......
	#endif

	#define security_initcall(fn)		module_init(fn)

**可以看到， 在编译为模块时， module_init() 宏会将模块的入口函数声明为 函数指针 “init_module” 的一个别名(即两个符号等价)**

linux kernel中使用 struct module来描述一个模块的相关信息， 其中 module.init 指向模块的入口函数， 在编译模块的 MODPOST阶段(即modpost命令中的add_header()函数中)， 会生成一个struct module  __this_module， 并且附加在模块的 “.gnu.linkonce.this_module” section中

**在编写内核模块时常用的宏  THIS_MODULE 即获取MODPOST阶段生成的 struct module  __this_module的地址， 无论是built-in 还是built module**

	static void add_header(struct buffer *b, struct module *mod)
	{
		buf_printf(b, "#include <linux/module.h>\n");
		buf_printf(b, "#include <linux/vermagic.h>\n");
		buf_printf(b, "#include <linux/compiler.h>\n");
		buf_printf(b, "\n");
		buf_printf(b, "MODULE_INFO(vermagic, VERMAGIC_STRING);\n");
		buf_printf(b, "\n");
		buf_printf(b, "struct module __this_module\n");
		buf_printf(b, "__attribute__((section(\".gnu.linkonce.this_module\"))) = {\n");
		buf_printf(b, "\t.name = KBUILD_MODNAME,\n");
		if (mod->has_init)
			buf_printf(b, "\t.init = init_module,\n");
		if (mod->has_cleanup)
			(b, "#ifdef CONFIG_MODULE_UNLOAD\n"
			      "\t.exit = cleanup_module,\n"
			      "#endif\n");
		buf_printf(b, "\t.arch = MODULE_ARCH_INIT,\n");
		buf_printf(b, "};\n");
	}

在MODPOST阶段， 会为每一个模块添加一个 xxx.mod.c 文件， 例如， 若模块名为 hello， 则会添加一个 hello.mod.c 文件， 其中包含了 VERMAGIC， __this_module,  __module_depends 等信息， 而_this_module.init 会被初始化为 init_module (即对应的模块入口函数的别名)， 这一文件最终会被编译链接到模块中去

在用户空间调用系统调用 init_module() 来加载模块时, 内核中的执行流程如下：

	SYSCALL_DEFINE3(init_module)
		load_module()
			layout_and_allocate()
				setup_load_info()	
					info->index.mod = find_sec(info, ".gnu.linkonce.this_module");
			do_init_module()
				do_one_initcall(mod->init);

在 setup_load_info() 中， 使用 ".gnu.linkonce.this_module" section的内容来初始化一个 struct module， 然后在 do_one_initcall() 中调用 module.init 函数指针， 即调用了模块的初始化函数

###6. built module时内核模块的出口的实现

built module时内核模块的出口的实现与模块的入口的实现相类似， 请先阅读 “built module时内核模块入口的实现”

在将内核模块编译成module时， 在 “linux/init.h” 中有

	#define module_exit(exitfn)					\
		static inline exitcall_t __exittest(void)		\
		{ return exitfn; }					\
		void cleanup_module(void) __attribute__((alias(#exitfn)))

即将对应的模块的出口函数声明为一个函数指针变量 cleanup_module 的别名(即两个符号等价)， 与上一节中的module形式的内核模块的入口函数的实现类似， 在在MODPOST阶段， 在模块的 “.gnu.linkonce.this_module” section中附加的 struct module __this_module中， module.exit 指向 cleanup_module 符号

在 用户空间调用系统调用 delete_module() 卸载内核模块时， kernel中的调用流程为

	SYSCALL_DEFINE2(delete_module)
		mod->exit()

即调用了模块的出口函数
