---
layout: post
title: "linux kprobe"
description: 
category: linux-kernel
tags: [linux, debug, driver]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. kprobes  
  
Kprobes 提供了一个强行进入任何（除了Kprobes模块自身的实现代码）内核例程，并从中断处理器无干扰地收集信息的接口。使用 Kprobes 可以轻松地收集处理器寄存器和全局数据结构等调试信息，而无需对Linux内核频繁编译和启动。  
  
它的基本工作机制是：用户指定一个探测点，并把一个用户定义的处理函数关联到该探测点，当内核执行到该探测点时，相应的关联函数被执行，然后继续执行正常的代码路径。  
  
kprobe实现了三种类型的探测点: kprobes, jprobes和kretprobes (也叫返回探测点)。 kprobes是可以被插入到内核的任何指令位置的探测点，jprobes则只能被插入到一个内核函数的入口（一般用于调试函数参数），而kretprobes则是在指定的内核函数返回时才被执行（一般用于调试函数返回值）。
  
一般，使用kprobe的程序实现作一个内核模块，模块的初始化函数来负责安装探测点，退出函数卸载那些被安装的探测点。kprobe提供了接口函数（APIs）来安装或卸载探测点。目前kprobe支持如下架构：i386、x86_64、ppc64、ia64(不支持对slot1指令的探测)、sparc64 (返回探测还没有实现)。  
  
###2. kprobes 族的工作原理  
  
当安装一个kprobes探测点时，kprobe首先备份被探测的指令，然后使用断点指令(即在i386和x86_64的int3指令)来取代被探测指令的头一个或几个字节。当CPU执行到探测点时，将因运行断点指令而执行trap操作，那将导致保存CPU的寄存器，调用相应的trap处理函数，而trap处理函数将调用相应的notifier_call_chain（内核中一种异步工作机制）中注册的所有notifier函数，kprobe正是通过向trap对应的notifier_call_chain注册关联到探测点的处理函数来实现探测处理的。当kprobe注册的notifier被执行时，它首先执行关联到探测点的pre_handler函数，并把相应的kprobe struct和保存的寄存器作为该函数的参数，接着，kprobe单步执行被探测指令的备份，最后，kprobe执行post_handler。等所有这些运行完毕后，紧跟在被探测指令后的指令流将被正常执行。  
  
jprobe通过注册kprobes在被探测函数入口的来实现，它能无缝地访问被探测函数的参数。jprobe处理函数应当和被探测函数有同样的原型，而且该处理函数在函数末必须调用kprobe提供的函数jprobe_return()。当执行到该探测点时，kprobe备份CPU寄存器和栈的一些部分，然后修改指令寄存器指向jprobe处理函数，当执行该jprobe处理函数时，寄存器和栈内容与执行真正的被探测函数一模一样，因此它不需要任何特别的处理就能访问函数参数， 在该处理函数执行到最后时，它调用jprobe_return()，那导致寄存器和栈恢复到执行探测点时的状态，因此被探测函数能被正常运行。需要注意，被探测函数的参数可能通过栈传递，也可能通过寄存器传递，但是jprobe对于两种情况都能工作，因为它既备份了栈，又备份了寄存器，当然，前提是jprobe处理函数原型必须与被探测函数完全一样。  
  
kretprobe也使用了kprobes来实现，当用户调用register_kretprobe()时，kprobe在被探测函数的入口建立了一个探测点，当执行到探测点时，kprobe保存了被探测函数的返回地址并取代返回地址为一个trampoline的地址，kprobe在初始化时定义了该trampoline并且为该trampoline注册了一个kprobe,当被探测函数执行它的返回指令时，控制传递到该trampoline，因此kprobe已经注册的对应于trampoline的处理函数将被执行，而该处理函数会调用用户关联到该kretprobe上的处理函数，处理完毕后，设置指令寄存器指向已经备份的函数返回地址，因而原来的函数返回被正常执行。  
  
被探测函数的返回地址保存在类型为kretprobe_instance的变量中，结构kretprobe的maxactive字段指定了被探测函数可以被同时探测的实例数，函数register_kretprobe()将预分配指定数量的kretprobe_instance。如果被探测函数是非递归的并且调用时已经保持了自旋锁（spinlock），那么maxactive为1就足够了； 如果被探测函数是非递归的且运行时是抢占失效的，那么maxactive为NR_CPUS就可以了；如果maxactive被设置为小于等于0, 它被设置到缺省值（如果抢占使能， 即配置了 CONFIG_PREEMPT，缺省值为10和2*NR_CPUS中的最大值，否则缺省值为NR_CPUS）。  
  
如果maxactive被设置的太小了，一些探测点的执行可能被丢失，但是不影响系统的正常运行，在结构kretprobe中nmissed字段将记录被丢失的探测点执行数，它在返回探测点被注册时设置为0，每次当执行探测函数而没有kretprobe_instance可用时，它就加1。
  
###3. kprobes的不足  
  
kprobes也有无能为力的时候， 在使用前，请你看看这些。 
  
+ kprobe允许在同一地址注册多个kprobes，但是不能同时在该地址上有多个jprobes。  
+ 通常，用户可以在内核的任何位置注册探测点，特别是可以对中断处理函数注册探测点，但是也有一些例外。如果用户尝试在实现kprobe的代码(包括kernel/kprobes.c和arch/*/kernel/kprobes.c以及do_page_fault和notifier_call_chain)中注册探测点，register_*probe将返回-EINVAL.  
+ 如果为一个内联(inline)函数注册探测点，kprobe无法保证对该函数的所有实例都注册探测点，因为gcc可能隐式地内联一个函数。因此，要记住，用户可能看不到预期的探测点的执行。  
+ 一个探测点处理函数能够修改被探测函数的上下文，如修改内核数据结构，寄存器等。因此，kprobe可以用来安装bug解决代码或注入一些错误或测试代码。  
+ 如果一个探测处理函数调用了另一个探测点，该探测点的处理函数不将运行，但是它的nmissed数将加1。多个探测点处理函数或同一处理函数的多个实例能够在不同的CPU上同时运行。  
+ 除了注册和卸载，kprobe不会使用mutexe或分配内存。
+ 探测点处理函数在运行时是失效抢占的，依赖于特定的架构，探测点处理函数运行时也可能是中断失效的。因此，对于任何探测点处理函数，不要使用导致睡眠或进程调度的任何内核函数（如尝试获得semaphore)。  
+ kretprobe是通过取代返回地址为预定义的trampoline的地址来实现的，因此栈回溯和gcc内嵌函数 __builtin_return_address()调用将返回trampoline的地址而不是真正的被探测函数的返回地址。  
+ 如果一个函数的调用次数与它的返回次数不相同，那么在该函数上注册的kretprobe探测点可能产生无法预料的结果（do_exit()就是一个典型的例子，但do_execve() 和 do_fork()没有问题）。  
+ 当进入或退出一个函数时，如果CPU正运行在一个非当前任务所有的栈上，那么该函数的kretprobe探测可能产生无法预料的结果，因此kprobe并不支持在x86_64上对__switch_to()的返回探测，如果用户对它注册探测点，注册函数将返回-EINVAL。  
  
###4. 让内核支持kprobes
  
kprobe已经被包含在2.6内核中，但是只有最新的内核才提供了上面描述的全部功能，为了使能kprobe，用户必须在编译内核时设置CONFIG_KPROBES，即选择在“Instrumentation Support“中的“Kprobes”项。如果用户希望动态加载和卸载使用kprobe的模块，还必须确保“Loadable module support” (CONFIG_MODULES)和“Module unloading” (CONFIG_MODULE_UNLOAD)设置为y。如果用户还想使用kallsyms_lookup_name()来得到被探测函数的地址，也要确保CONFIG_KALLSYMS设置为y，当然设置CONFIG_KALLSYMS_ALL为y将更好。  
  
###5. 获取内核函数的地址  
  
在注册过程中，您还需要指定插入探测器的内核例程的地址。使用这些方法中的任意一个来获得内核例程 的地址  
  
1. 从 System.map 文件直接得到地址  
   + 例如，要得到 do_fork 的地址，可以在命令行执行 `$grep do_fork /usr/src/linux/System.map`
2. 使用 nm 命令。 
   + `$nm vmlinuz |grep do_fork`
3. 从 /proc/kallsyms 文件获得地址  
   + `$cat /proc/kallsyms |grep do_fork`
4. 使用 kallsyms_lookup_name() 例程  
   + 这个例程是在 kernel/kallsyms.c 文件中定义的，要使用它，必须启用 CONFIG_KALLSYMS 编译内核。 kallsyms_lookup_name() 接受一个字符串格式内核例程名， 返回那个内核例程的地址。例如： kallsyms_lookup_name("do_fork");
然后在 init_moudle 中注册您的探测器  
5. 直接使用内核函数名（针对哪些被 EXPORT_SYMBOL 导出的函数）  
  
###6. 使用kprobes  
  
kprobes 使用一个 kprobe 结构体来描述  
  
	struct kprobe {
		......
		kprobe_opcode_t	*addr;
		const char 		*symbol_name;
		unsigned int 	offset;
		......
		kprobe_pre_handler_t 	pre_handler;
		kprobe_post_handler_t 	post_handler;
		kprobe_fault_handler_t 	fault_handler;
		kprobe_break_handler_t 	break_handler;
		......
	}

其中， addr为需要注入探测点的内核代码， 或者你也可以直接给symbol_name赋值要插入的内核函数名来指定插入点，offset为插入点相对于给定地址的偏移量。  
使用以下函数来注册/取消注册的探测点。  
  
	int register_kprobe(struct kprobe *kp);
	void unregister_kprobe(struct kprobe *kp);
    
示例程序如下
  
	#include <linux/module.h>
  	#include <linux/init.h>
	#include <linux/kprobes.h>

	MODULE_LICENSE("GPL");

	static int kprobes_pre(struct kprobe* t_kp, struct pt_regs *regs)
	{
		pr_info("kprobes_pre: in %s function\n", t_kp->symbol_name);
		return 0;
  	}

	static void kprobes_post(struct kprobe* t_kp, struct pt_regs *regs)
	{
		pr_info("kprobes_post: in %s function\n", t_kp->symbol_name);
  	}

	static int kprobes_fault(struct kprobe* t_kp, struct pt_regs *regs)
	{
		pr_info("kprobes_fault: in %s function\n", t_kp->symbol_name);
		return 0;
	}

	static int kprobes_break(struct kprobe* t_kp, struct pt_regs *regs)
	{
		pr_info("kprobes_break: in %s function\n", t_kp->symbol_name);
		return 0;
	}

	struct kprobe kp = {
		//.addr           = 0xffffffff8103a500,
		.symbol_name    = "do_fork",
		.pre_handler    = kprobes_pre,
		.post_handler   = kprobes_post,
		.fault_handler  = kprobes_fault,
		.break_handler  = kprobes_break,
	};

	static int __init my_init(void)
	{
		pr_info("kprobe-test: my_init()\n");
		if( register_kprobe(&kp) < 0 )
			pr_err("register kprobe fail\n");
		else    pr_info("register kprobe success\n");

		return 0;
	}

	static void __exit my_exit(void)
	{
		pr_err("kprobe-test: my_exit()\n");
		if( &kp )
			unregister_kprobe(&kp);
	}

	module_init(my_init);
	module_exit(my_exit);

###7. jprobes的使用  
  
jprobe 基于 kprobe实现， 使用 jprobe 结构体来描述一个jprobe插入点  
  
	struct jprobe {
		struct kprobe kp;
		void *entry;	/* probe handling code to jump to */
	};
    
jprobe自己实现了 struct kprobe 的 pre_handler， post_handler ... 这些handler函数， 用于插入要调试函数的代理函数， 因此， 在注册探测点时， 只需指定探测地址（addr或者symbol_name加offset值）,以及自己的代理函数（entry 域），代理函数必须要和要调试的函数原型一样, 且代理函数必须使用 jprobe_return() 来返回。  
  
使用下列函数注册/取消注册 jprobe 探测点  
  
	int register_jprobe(struct jprobe *jp);
	void unregister_jprobe(struct jprobe *jp);
    
jprobe的使用示例  

	#include <linux/module.h>
	#include <linux/init.h>
	#include <linux/sched.h>
	#include <linux/kprobes.h>

	MODULE_LICENSE("GPL");

	static int test(int var)
	{
		pr_info("jprobe-test: in test() var is %d\n", var);
		return 0;
	}

	static int proxy_test(int var)
	{
		pr_info("jprobe-test: in proxy_test() var is %d\n", var);
		jprobe_return();
		return 0;
	}

	struct jprobe jp = {
		.kp     = {     .addr   = test, },
		.entry  = proxy_test
	};

	static int __init my_init(void)
	{
		pr_info("jprobe-test: my_init()\n");
		if( register_jprobe(&jp) < 0 )
			pr_err("jprobe-test: register jprobe fail\n");
		else    {
			pr_info("jprobe-test: register jprobe success\n");
		}

		return 0;
	}

	static void __exit my_exit(void)
	{
		pr_info("jprobe-test: now call test(615)\n");
		test(615);

		pr_err("jprobe-test: my_exit()\n");
		if( &jp )
			unregister_jprobe(&jp);                                                        
	}

	module_init(my_init);
	module_exit(my_exit);

注意， 我试图在 my_init() 里面， 注册了jprobe之后调用 test(), 但是会引发内核 crash, 因为在module_init()返回之前， 模块还没有加载完成。  
  
###8. kretprobe  
  
	struct kretprobe	{
		struct kprobe kp;
		kretprobe_handler_t handler;
		int maxactive;
		int nmissed;
		struct hlist_head free_instances;
		struct hlist_head used_instances;
		......
	}
   
+ kp	 		该成员是kretprobe内嵌的struct kprobe结构。
+ handler			该成员是调试者定义的回调函数.
+ maxacitive		该成员是最多支持的返回地址实例数。
+ nmissed			该成员记录有多少次该函数返回没有被回调函数处理。
+ free_instance		用于链接未使用的返回地址实例，在注册时初始化。
+ used_instances	该成员是正在被使用的返回地址实例链表。

注册/取消注册 kretprobe探测点  

	int register_kretprobe(struct kretprobe *rp);
	void unregister_kretprobe(struct kretprobe *rp);
        
kretprobe 示例 
  
	#include <linux/module.h>
	#include <linux/init.h>
	#include <linux/sched.h>
	#include <linux/kprobes.h>

	MODULE_LICENSE("GPL");

	static int ret_test(struct kretprobe_instance *ri, struct pt_regs *regs)
	{
		int ret;
		ret = (int)regs_return_value(regs);
		pr_info("kretprobe-test: in ret_test() ret %d\n", ret);
		return 0;
	}

	struct kretprobe krp = {
		.kp             = {     .symbol_name    = "do_fork", },
		.handler        = ret_test,
		.maxactive      =       20,
	};

	static int __init my_init(void)
	{
		pr_info("kretprobe-test: my_init()\n");
		if( register_kretprobe(&krp) < 0 )
			pr_err("kretprobe-test: register kretprobe fail\n");
		else    {
			pr_info("kretprobe-test: register kretprobe success\n");
		}

		return 0;
	}

	static void __exit my_exit(void)
	{
		pr_err("kretprobe-test: my_exit()\n");
		if( &krp )
			unregister_kretprobe(&krp);
	}

	module_init(my_init);
	module_exit(my_exit);  
    
###9. kprobes 中用到的cpu异常处理  
  
CPU异常是在CPU运行期间，由于外围硬件发出中断信号或是执行CPU异常指令等情况所引发的。  

CPU异常可分为硬件异常和软件异常。  
  
硬件异常也称为硬件中断，一般是有外围硬件设备发出中断信号引起。当外围设备发出一个中断信号，该信号被发往中断处理器仲裁，比较有名的是 8259中断处理芯片，目前Intel的CPU一般采用APIC(Advanced Programmable Interrupt Controllers)来实现。中断处理器仲裁后发往 CPU的中断引脚，这时CPU会开始一次中断处理流程。  
  
软件异常不是由外围设备发出的，而是由程序员写入程序里的一些 CPU 异常指令引起的，比如int，int3这类指令就会引起一次软件异常。在早期的 CPU上Linux系统调用的实现就是利用 int指令。当CPU执行到这些异常指令时，同样会开始一次异常处理流程。 
  
操作系统CPU异常处理的实现是和特定CPU体系结构紧密相关的。Intel IA32体系的CPU的每个中断或是异常都有一个向量号，这些向量号从 0到255。Linux内核中，每一个中断向量号都对应中断描述符表(IDT)中的一项，因此中断描述符表一共有 256项。IDT每一项包含8个字节，这8个字节的其中一部分是中断处理函数的地址。表项的其他字段的含义这里不再介绍，可以从 Intel IA32程序员手册中查到。Linux内核在初始化阶段用set_trap_gate()宏初始中断描述符表。  
  
Linux内核中，异常处理是通过两级跳转实现的。比如，如果当CPU运行过程中接收到一次异常信号，此时CPU会到根据中断向量号到中断描述符表中找到对应项，再跳转到表项中所指的地址去执行。  

这里跳转到的地址一般情况下对应源文件 linux/arch/i386/kernel/entry.S 中的某个汇编函数入口。汇编函数经过一定的初始化工作后再跳转到C函数中去执行。这里的C函数才是真正的异常处理函数，当从C函数返回到原来的汇编函数 后，汇编函数还会做一部分后续工作。最后汇编函数执行 iret指令从中断上下文中返回到被中断的代码中继续执行。这是 Linux内核处理异常一般过程，因此称Linux的异常处理过程是两次跳转实现的。  
  
在Kprobes的实现中同样也用到了CPU异常，当插入一个探测点的时候，Kprobes处理例程会把插入点处的指令保存起来，然后用int3指令代替。当CPU运行到插入的int3指令时，Linux内核就进入了异常处理流程，之后再运行调试者预定义的回调函数。利用 CPU异常来实现探测点的触发并处理是Kprobes实现的关键。  
  
###10. kprobes 中用到的单步调试  
  
调试器的单独执行功能是非常有用的，程序员可以通过单步执行代码来确定程序执行的流程，随时掌握变量变化情况，从而精确定位程序错误发生的位置。可以说单步执行功能是一个调试器必须具备的功能。single-step技术就是为了调试器的单步执行而设计的。  
  
single-step技术的主要思想是，当程序执行到某条想要单独执行 CPU指令时，在执行之前产生一次CPU异常，此时把异常返回时的CPU的EFLAGS寄存器的TF(调试位)位置为1，把IF(中断屏蔽位)标志位置为 0，然后把EIP指向单步执行的指令。当单步指令执行完成后，CPU会自动产生一次调试异常（由于TF被置位）。此时，调试器一般都会把控制权又交回调试 器，回到交互模式。  
  
在Kprobes实现中同样也用到了single-step技术，但目的不是为了回到交互模式，而是把控制权交回Kprobes控制流程。当Kprobes完成pre_handler()处理后，就会利用single-step技术执行被调试指令。此时，Kprobes会利用debug异常，执行post_handler()。这是single-step技术在Kprobes的主要应用。  
  
###11. kprobes 用到的RCU  
  
Linux内 核有时会访问到一些全局的数据结构，如果此时内核被抢占并且数据被修改，又或者在SMP系统（即多处理器系统）上运行，可能会有多个 CPU同时访问同一块内存的情况，此时内核数据就可能产生不一致性。为了避免这种情况，内核使用了一些同步机制，比如利用信号量同步，利用自旋锁同步等方 法。在2.6版本的内核中又引入了一种新的内核同步机制RCU，这是Read Copy Update的缩写。RCU的主要思想是分为两个部分，第一部分是防止其他访问者对被保护对象写入，第二部分是真正展开写入行为。读者可以随时访问被 RCU保护的对象而不用获得任何锁。写者先对对象的副本修改，在所有读者都退出时再执行写入行为，不同的写者之间需要同步。RCU同步机制适用于存在大量 读操作而很少写操作的情况。因为这种情况下，读操作不用获得任何锁就可以对共享对象进行读操作，极大提高了效率。  
   
在Kprobes对探测点数据结构的操作中也是大量存在读操作而只有很少部分写操作。为了提高效率，Kprobes的 实现中也引入了RCU机制。当发生探测点注册或是注销时，或是在执行探测点的回调函数时，探测点数据结构struct kprobe就会被RCU机制保护起来。根据RCU的原理，写者的操作是被延迟的，而如果读者发生了阻塞，那写者的操作直到读者被唤醒后才进行，这会大大 降低 RCU的效率。因此在执行Kprobes回调函数时内核的抢占以及CPU中断都被禁用了。对于Kprobes来说，用RCU机制实现回调函数访问也极大的提高了SMP系统上多探测点探测的效率，因为可以并行的执行回调函数。但在这样的机制下，回调函数必须被设计成可重入的函数。  
  
###12. kprobe的注册过程  
  
当调试者向内核插入一个 kprobe 模块时，首先会执行注册探测点操作。该操作主要由register_kprobe()函数完成，在下文中都称该函数为注册器。注册器的参数中包含一个 struct kprobe结构，该结构由调试者在调试模块中创建。  
  
首先，注册器会进行一些正确性检查工作，判断传入的 struct kprobe结构中symbol_name和addr 是否同时存在，如果是则返回错误。之后还会判断探测地址是否在内核代码段中并且不在Kprobes实现相关的代码中。这样的检查是很必要的，如果探测地址出现在Kprobes实现相关代码中就会造成递归现象。如果被探测的地址已经被注册过，则会在kprobe_table中以链表形式组织。  
  
其次，注册器会保存被探测地址的指令码到struct kprobe结构的ainsn.ainsn中去，以便以后进行single-step操作。对kprobe初始化完成后，注册器会把传入的struct kprobe结构指针插入哈希表中。最后，注册器把被探测的指令的第一个字节替换成 int3指令。到这里，kprobe的注册工作就完成了。  
  
##13. kprobe int3异常处理过程  
  
完成注册后kprobe的准备工作就完成了，一旦内核执行到被探测的指令，也就是注册时被替换成的int3指令时，就会引发一次软件异常。CPU会根据中断描述符表执行中断处理函数，int3的中断处理函数在/linux/arch/i386/kernel/entry.S中实现，KPROBE_ENTRY(int3)就是该中断处理函数的入口。汇编中断处理函数会调用 do_int3()函数，作为 int3 中断处理的 C 语言处理函数。  
  
do_int3()函数一开始就会去调用notify_die()函数，该函数的主要作用是调用内核代码注册的异常的 回 调 函 数 。 在 Kprobes 的初始化代码（init_Kprobes()函数）中调用了register_die_notifier() 用于注册异常回调函数 。 Kprobes注册的异常回调函数为probe_exceptions_notify()。  
  
此时执行权交到Kprobes之 中，kprobe_exception_notify()函数开始执行。该函数的参数中有一个参数val，该参数可以用于判断当前回调函数由有什么异常产 生的。这里异常由 int3指令产生，因此接收到的参数应该为“DIE_INT3”。此时，又会调用 kprobe_handler()函数，该函数是Kprobes处理int3异常的主要实现函数。该函数首先会把发生异常的地址记录下来，因为该地址就是注册探测点的地址。为了防止内核被抢占，该函数禁止内核抢占功能。在 i386 CPU上，进入int3中断处理时已经关闭CPU中断，目前Kprobes的实现中只有i386体系上会关闭CPU中断，在其他体系上的实现都没有这样做。  
  
接着，开始检查此次 int3 异常是否是由前一次 Kprobes 处理流程引发的，如果是由前一次Kprobes处理流程引发，则有两种可能性。第一是该次Kprobes处理由于前一次回调函数执行了被探测代码造成的，第二种可能性是由于 jprobe造成的，这部分将在 jprobe的实现一节中详细讨论。如果int3异常不是由前一次Kprobes处理流程引发的，根据先前记录下来的探测点地址到哈希表中找到已注册的struct kprobe结构。如果该结构中包含了pre_handler函数指针，则执行该预定的函数。  
  
执行完用户定义的 pre_handler函数时，已经完成了一部分的调试工作。接下来，就开始准备single-step步骤，该步骤用 prepare_singlestep()函数完成。这个函数与体系结构相关，下面是prepare_singlestep()函数在i386体系CPU 上的主要实现代码：  
程序1 　prepare_singlestep()函数部分代码  
01 regs->eflags |= TF_MASK;  
02 regs->eflags &= ~IF_MASK;  
03 regs->eip = (unsigned long)p->ainsn.insn;  
上面的代码中设置了EFLAGS中的TF位并清空IF位，同时把异常返回的指令寄存器地址改为保存起来的原探测指令处，当异常返回时这些设置就会生效。 single-step技术已经在上文中讨论过，这里不再赘述。执行完被探测的指令后，由于 CPU的标志寄存器被置位，此时又会引发一次CPU异常，该异常在Linux内核中被称为DEBUG异常。  
  
##14. Kprobe DEBUG异常处理  
   
Linux内核中对DEBUG异常的处理方式与处理int3异常很类似。DEGUG异常的中断处理函数也是在/linux/arch/i386/kernel/entry.S 中实现，KPROBE_ENTRY(debug)就是该异常的中断处理函数的入口。该函数会调用do_debug()函数进一步处理DEBUG异常，同样 的notify_die()函数被调用。与int3异常不同的是此时传入notify_die()函数的第一个参数是“DIE_DEBUG”。  
  
最终，notify_die() 函数会调用Kprobes初始化时注册的回调函数kprobe_exceptions_notify() 。此时，控制权又一次交回Kprobes 。  
  
kprobe_exceptions_notify() 判断传入的类型为DIE_DEBUG，这时会去调用post_kprobe_handler ()函数。post_kprobe_handler ()首先判断用户定义的post_handler回调函数是否存在，如果存在则执行之。  
  
之后，会调用 resume_execution()函数做一些会做恢复工作，该函数会把 EFLAGES寄存器的TF 为清空，并根据被探测指令类型的不同，做不同的处理。在 resume_execution()返回后，post_kprobe_handler ()函数就会启用在 int3 异常处理中被禁止的内核抢占功能。到这里，Kprobes对DEBUG异常的处理基本完成了，又把控制权交回内核。  
  
以上是kprobe执行的主要流程，可以看出kprobe利用了两次CPU异常的方式执行了用户定义的pre_handler 和 post_handler 回调函数。并通过 single-step 技术执行了被探测指令。当一次kprobe 执行周期完成后，又开始等待新一轮执行周期的到来。只有当调试者卸载了调试模块后，kprobe的生命周期才算结束。  
  
