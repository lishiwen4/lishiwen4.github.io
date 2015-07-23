---
layout: post
title: "linux 中断子系统"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux 中断子系统  
  
在一个完整的系统中， 将与中断相关的硬件划分为3类：  
  
+ 设备  
  
设备是发起中断的源，当设备需要请求某种服务的时候，它会发起一个硬件中断信号，通常，该信号会连接至中断控制器，由中断控制器做进一步的处理。在现代的移动设备中，发起中断的设备可以位于soc（system-on-chip）芯片的外部，也可以位于soc的内部，因为目前大多数soc都集成了大量的硬件IP，例如I2C、SPI、Display Controller等等。  
  
+ 中断控制器  
  
中断控制器负责收集所有中断源发起的中断，现有的中断控制器几乎都是可编程的，通过对中断控制器的编程，我们可以控制每个中断源的优先级、中断的电器类型，还可以打开和关闭某一个中断源，在smp系统中，甚至可以控制某个中断源发往哪一个CPU进行处理。对于ARM架构的soc，使用较多的中断控制器是VIC（Vector Interrupt Controller），进入多核时代以后，GIC（General Interrupt Controller）的应用也开始逐渐变多。  
  
+ cpu  
  
cpu是最终响应中断的部件，它通过对可编程中断控制器的编程操作，控制和管理者系统中的每个中断，当中断控制器最终判定一个中断可以被处理时，他会根据事先的设定，通知其中一个或者是某几个cpu对该中断进行处理，虽然中断控制器可以同时通知数个cpu对某一个中断进行处理，实际上，最后只会有一个cpu相应这个中断请求，但具体是哪个cpu进行响应是可能是随机的，中断控制器在硬件上对这一特性进行了保证，不过这也依赖于操作系统对中断系统的软件实现。在smp系统中，cpu之间也通过IPI（inter processor interrupt）中断进行通信。  
  
###2. irq  
  
在系统中注册的每一个中断源， 都会分配一个唯一的编号来标识该中断， 称之为 irq 号， 在驱动中， 调用和 irq 相关的 api 时， 都需要使用 irq 号来指定 中断源。 中断发生时，cpu通常会从中断控制器中获取相关信息，然后计算出相应的 IRQ 编号，然后把该 IRQ 编号传递到相应的驱动程序中。在移动设备中，每个中断源的 IRQ 编号都会在 arch 相关的一些头文件中，例如 “arch/xxx/mach-xxx/include/irqs.h”  
  
###3. 通用中断子系统的软件抽象  
  
在通用中断子系统（generic irq）出现之前，内核使用 \__do_IRQ 处理所有的中断，这意味着 \__do_IRQ 中要处理各种类型的中断，这会导致软件的复杂性增加，层次不分明，而且代码的可重用性也不好。事实上，到了内核版本2.6.38，\__do_IRQ 这种方式已经彻底在内核的代码中消失了。通用中断子系统的原型最初出现于 ARM 体系中，一开始内核的开发者们把3种中断类型区分出来，他们是：  
  
+ level type
+ edge type
+ simple type
  
后续又针对需要回应 eoi (end of interrupt) 的中断控制器以及 SMP 架构， 分别添加了：  
  
+ fast eoi type
+ percpu type  
  
把这些不同的中断类型抽象出来后，成为了中断子系统的流控层。要使所有的体系架构都可以重用这部分的代码，中断控制器也被进一步地封装起来，形成了中断子系统中的硬件封装层。可以用下面的图示表示通用中断子系统的层次结构：   
  
####3.1 硬件封装层(中断控制器的软件抽象)  
  
包含了体系架构相关的所有代码，包括中断控制器的抽象封装，arch相关的中断初始化，以及各个IRQ的相关数据结构的初始化工作，cpu的中断入口也会在arch相关的代码中实现。中断通用逻辑层通过标准的封装接口（实际上就是struct irq_chip定义的接口）访问并控制中断控制器的行为，体系相关的中断入口函数在获取IRQ编号后，通过中断通用逻辑层提供的标准函数，把中断调用传递到中断流控层中。  
硬件封装层使用 struct irq_chip 来定义  
  
	struct irq_chip {
		const char	*name;
		unsigned int	(*irq_startup)(struct irq_data *data);
		void		(*irq_shutdown)(struct irq_data *data);
		void		(*irq_enable)(struct irq_data *data);
		void		(*irq_disable)(struct irq_data *data);

		void		(*irq_ack)(struct irq_data *data);
		void		(*irq_mask)(struct irq_data *data);
		void		(*irq_mask_ack)(struct irq_data *data);
		void		(*irq_unmask)(struct irq_data *data);
		void		(*irq_eoi)(struct irq_data *data);

		int		(*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force);
		int		(*irq_retrigger)(struct irq_data *data);
		int		(*irq_set_type)(struct irq_data *data, unsigned int flow_type);
		int		(*irq_set_wake)(struct irq_data *data, unsigned int on);

		void		(*irq_bus_lock)(struct irq_data *data);
		void		(*irq_bus_sync_unlock)(struct irq_data *data);

		void		(*irq_cpu_online)(struct irq_data *data);
		void		(*irq_cpu_offline)(struct irq_data *data);

		void		(*irq_suspend)(struct irq_data *data);
		void		(*irq_resume)(struct irq_data *data);
		void		(*irq_pm_shutdown)(struct irq_data *data);

		void		(*irq_print_chip)(struct irq_data *data, struct seq_file *p);

		unsigned long	flags;
	};
    
+ name	在/proc/interrupts 中显示的名称  
+ irq_startup	开启中断(如果为null， 则默认调用irq_enable成员函数)
+ irq_shutdown	关闭中断(如果为null， 则默认调用irq_disable成员函数)
+ irq_enable	使能中断(如果为null， 则默认调用irq_unmask成员函数)
+ irq_disable	禁用irq
+ irq_ack		允许触发新中断（为了防止中断嵌套， 需要作出必要的处理后在能允许触发新的中断）
+ irq_mas		屏蔽中断源
+ irq_mask_ack	屏蔽中断源并允许触发新的中断
+ irq_unmas		解除对中断源的屏蔽  
+ irq_eoi		结束中断
+ irq_set_affinity	在SMP上设置中断的亲和性
+ irq_retrigger	重新发送中断到cpu
+ irq_set_type	设置中断的流控类型
+ irq_set_wake	使能/关闭中断唤醒
+ irq_bus_lock	用于锁定对低速总线的访问（中断控制器可能通过低速总线如i2c访问）
+ irq_bus_sync_unlock	用于解锁对低速总线的访问（中断控制器可能通过低速总线如i2c访问）
+ irq_cpu_online	为第二cpu配置中断源
+ irq_cpu_offline	取消为第二cpu配置中断源
+ irq_suspend		在suspend时被核心代码回调  
+ irq_resume		在resume时被核心代码回调
+ irq_pm_shutdown	在shutdown时被核心代码回调
+ irq_print_chip	用于打印chip信息
+ flags				chip的特定flag  
  
irq_chip 除了 “name” 和 "flags" 这两个成员， 其它成员都是回调函数， 针对每一个中断控制器，只要实现以上接口（不必所有的接口）， 并把它和相应的irq关联起来， 上层的实现即可通过这些接口访问中断控制器。  
  
中断控制器主要完成以下的功能  
  
+ 对各个irq的优先级进行控制；
+ 向CPU发出中断请求后，提供某种机制让CPU获得实际的中断源（irq编号）；
+ 控制各个irq的电气触发条件，例如边缘触发或者是电平触发；
+ 使能（enable）或者屏蔽（mask）某一个irq；
+ 提供嵌套中断请求的能力；
+ 提供清除中断请求的机制（ack）；
+ 有些控制器还需要CPU在处理完irq后对控制器发出eoi指令（end of interrupt）；
+ 在smp系统中，控制各个irq与cpu之间的亲缘关系（affinity）；
  
####3.2 中断流控  
  
所谓中断流控是指合理并正确地处理连续发生的中断，比如一个中断在处理中，同一个中断再次到达时如何处理，何时应该屏蔽中断，何时打开中断，何时回应中断控制器等一系列的操作。该层实现了与体系和硬件无关的中断流控处理操作，它针对不同的中断电气类型（level，edge......），实现了对应的标准中断流控处理函数，在这些处理函数中，最终会把中断控制权传递到驱动程序注册中断时传入的处理函数或者是中断线程中。目前内核提供了以下几个主要的中断流控函数的实现：  
  
+ handle_edge_eoi_irq
+ handle_edge_irq
+ handle_fasteoi_irq
+ handle_level_irq
+ handle_nested_irq
+ handle_percpu_devid_irq
+ handle_percpu_irq
+ handle_simple_irq  
  
####3.3 中断通用逻辑层  
  
该层实现了对中断系统几个重要数据的管理，并提供了一系列的辅助管理函数。同时，该层还实现了中断线程的实现和管理，共享中断和嵌套中断的实现和管理，另外它还提供了一些接口函数，它们将作为硬件封装层和中断流控层以及驱动程序API层之间的桥梁：  
  
+ generic_handle_irq  
+ irq_set_chip
+ irq_set_chained_handle
+ irq_to_desc
  
####3.4 驱动程序API  
  
该部分向驱动程序提供了一系列的API，用于向系统申请/释放中断，打开/关闭中断，设置中断类型和中断唤醒系统的特性等操作。驱动程序的开发者通常只会使用到这一层提供的这些API即可完成驱动程序的开发工作，其他的细节都由另外几个软件层较好地“隐藏”起来了  
  
+ enable_irq
+ disable_irq
+ disable_irq_nosync
+ request_threaded_irq
+ irq_set_affinity
  
###4. irq_desc  
  
系统中每一个irq都对应着一个irq_desc结构来描述  
  
	struct irq_desc {
		struct irq_data		irq_data;
		unsigned int __percpu	*kstat_irqs;
		irq_flow_handler_t	handle_irq;
	#ifdef CONFIG_IRQ_PREFLOW_FASTEOI
		irq_preflow_handler_t	preflow_handler;
	#endif
		struct irqaction	*action;	/* IRQ action list */
		unsigned int		status_use_accessors;
		unsigned int		core_internal_state__do_not_mess_with_it;
		unsigned int		depth;		/* nested irq disables */
		unsigned int		wake_depth;	/* nested wake enables */
		unsigned int		irq_count;	/* For detecting broken IRQs */
		unsigned long		last_unhandled;	/* Aging timer for unhandled count */
		unsigned int		irqs_unhandled;
		raw_spinlock_t		lock;
		struct cpumask		*percpu_enabled;
	#ifdef CONFIG_SMP
		const struct cpumask	*affinity_hint;
		struct irq_affinity_notify *affinity_notify;
	#ifdef CONFIG_GENERIC_PENDING_IRQ
		cpumask_var_t		pending_mask;
	#endif
	#endif
		unsigned long		threads_oneshot;
		atomic_t		threads_active;
		wait_queue_head_t       wait_for_threads;
	#ifdef CONFIG_PROC_FS
		struct proc_dir_entry	*dir;
	#endif
		int			parent_irq;
		struct module		*owner;
		const char		*name;
	} ____cacheline_internodealigned_in_smp;  
      
+ kstat_irqs	用于irq的统计信息， 在 /proc 中可以看到
+ handle_irq	高层的irq事件处理函数  
+ preflow_handle	在流控前回调
+ action		中断的处理函数（request_irq注册）链表  
+ status		状态信息
+ depth			中断disable 计数，用于nested irq_disable调用
+ wake_depth	唤醒功能的enable 计数， multiple irq_set_irq_wake调用
+ irq_count		中断计数
+ last_unhandled	
+ irqs_unhandled
+ lock			for SMP， 保护irq_desc自身
+ affinity_hint	用于提示用户空间， 作为提示irq和cpu亲緣关系的依据
+ pending_mas	用于调整irq在各个cpu之间的平衡
+ threads_oneshot	用于处理共享的oneshot线程
+ threads_active	当前处理的irq action 编号
+ wait_for_threads	wait_for_threads
+ irq_data	将硬件封装层中经常使用到的几个字段单独封装 
    

		struct irq_data {
			unsigned int		irq;
			unsigned long		hwirq;
			unsigned int		node;
			unsigned int		state_use_accessors;
			struct irq_chip		*chip;
			struct irq_domain	*domain;
			void			*handler_data;
			void			*chip_data;
			struct msi_desc		*msi_desc;
			cpumask_var_t		affinity;
		};
        
+ irq	irq的编号
+ hwirq	硬件irq编号
+ node	内存node
+ state_use_accessors  硬件封装层需要使用的状态信息，内核定义了一组函数用于访问该字段：irqd_xxxx()
+ chip 	指向该irq所属的中断控制器的irq_chip结构指针
+ handler_data	每个irq的私有数据指针，该字段由硬件封转装使用，例如用作底层硬件的多路复用中断
+ chip_data 中断控制器的私有数据，该字段由硬件封转层使用
+ msi_desc 用于PCIe总线的MSI或MSI-X中断机制
+ affinity  记录该irq与cpu之间的亲缘关系，每一个bit代表一个cpu，置位后代表该cpu可能处理该irq

####4.1 基于数组的irq_desc管理方式  
  
平台相关板级代码事先根据系统中的IRQ数量，定义常量：NR_IRQS，在kernel/irq/irqdesc.c中使用该常量定义irq_desc结构数组：  
  
	struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
		[0 ... NR_IRQS-1] = {
			.handle_irq	= handle_bad_irq,
			.depth		= 1,
			.lock		= __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock),
		}
	};
    
####4.2 基于基数树的irq_desc管理方式 
  
当内核的配置项CONFIG_SPARSE_IRQ被选中时，内核使用基数树（radix tree）来管理irq_desc结构，这一方式可以动态地分配irq_desc结构，对于那些具备大量IRQ数量或者IRQ编号不连续的系统，使用该方式管理irq_desc对内存的节省有好处，而且对那些自带中断控制器管理设备自身多个中断源的外部设备，它们可以在驱动程序中动态地申请这些中断源所对应的irq_desc结构，而不必在系统的编译阶段保留irq_desc结构所需的内存。  
  
###5. 中断的初始化  
  
###6. 中断的流控处理  
  
中断流控处理的方式因平台不同而有所差别， 简单地以ARM和x86为例：  
  
####6.1 ARM体系上的中断流控处理
  
在ARM体系中， 中断处理进入c代码的第一个函数是 asm_do_IRQ：  
  
	asmlinkage void __exception_irq_entry
	asm_do_IRQ(unsigned int irq, struct pt_regs *regs)
	{
		handle_IRQ(irq, regs);
	}
    
    void handle_IRQ(unsigned int irq, struct pt_regs *regs)
	{
		struct pt_regs *old_regs = set_irq_regs(regs);

		irq_enter();

		/*
	 	* Some hardware gives randomly wrong interrupts.  Rather
	 	* than crashing, do something sensible.
	 	*/
        
		if (unlikely(irq >= nr_irqs)) {
		if (printk_ratelimit())
			printk(KERN_WARNING "Bad IRQ%u\n", irq);
			ack_bad_irq(irq);
		} else {
			generic_handle_irq(irq);
		}

		irq_exit();
		set_irq_regs(old_regs);
	}

	int generic_handle_irq(unsigned int irq)
	{
		struct irq_desc *desc = irq_to_desc(irq);

		if (!desc)
			return -EINVAL;
		generic_handle_irq_desc(irq, desc);
		return 0;
	}
    
    static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)
	{
		desc->handle_irq(irq, desc);
	}
    
1. asm_do_IRQ 中直接调用 handle_IRQ() 
2. handle_IRQ 中调用irq_enter()更新系统统计信息并禁止进程抢占
3. handle_IRQ generic_handle_irq()
4. generic_handle_irq 中调用irq_to_desc将irq号转换为irq_desc
5. generic_handle_irq 调用desc->handle_irq() 执行中断的流控函数  
    
####6.2 x86体系的中断流控处理  
  
中断处理进如的c代码的第一个函数是 do_IRQ:  
  
	unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
	{
		struct pt_regs *old_regs = set_irq_regs(regs);

		/* high bit used in ret_from_ code  */
		unsigned vector = ~regs->orig_ax;
		unsigned irq;

		irq_enter();
		exit_idle();

		irq = __this_cpu_read(vector_irq[vector]);

		if (!handle_irq(irq, regs)) {
			ack_APIC_irq();

			if (printk_ratelimit())
			pr_emerg("%s: %d.%d No irq handler for vector (irq %d)\n",
				__func__, smp_processor_id(), vector, irq);
		}

		irq_exit();

		set_irq_regs(old_regs);
		return 1;
	}

	bool handle_irq(unsigned irq, struct pt_regs *regs)
	{
		struct irq_desc *desc;
		int overflow;

		overflow = check_stack_overflow();

		desc = irq_to_desc(irq);
		if (unlikely(!desc))
			return false;

		if (user_mode_vm(regs) || !execute_on_irq_stack(overflow, desc, irq)) {
			if (unlikely(overflow))
			print_stack_overflow();
			desc->handle_irq(irq, desc);
		}

		return true;
	}  
  
1. do_IRQ 调用 irq_enter()来更新系统的统计信息并禁止抢占  
2. do_IRQ 调用 handle_irq()  
3. handle_irq 中调用 check_stack_overflow() 检查中断栈是否溢出
4. handle_irq 中调用 irq_to_desc() 将irq号转换为 irq_desc  
5. handle_irq 中调用 execute_on_irq_stack 切换到中断栈上执行  
6. handle_irq 中调用 desc->handle_irq() 执行中断的流控函数  
  
####6.3 与体系无关的中断流控处理部分  
  
irq的中断流控类型为：  
  
	typedef	void (*irq_flow_handler_t)(unsigned int irq,
					    struct irq_desc *desc);
                          
通用中断子系统中目前实现了以下标准流控函数：  
  
+ handle_edge_eoi_irq
+ handle_edge_irq
+ handle_fasteoi_irq
+ handle_level_irq
+ handle_nested_irq
+ handle_percpu_devid_irq
+ handle_percpu_irq
+ handle_simple_irq  

当然， 我们也可以根据需要， 定义自己的流控处理函数， 并使用以下api来设置irq的流控处理函数  
  
+ irq_set_handle
+ irq_set_chip_and_handler
+ irq_set_chip_and_handler_name
+ irq_set_chained_handler  
  
#####6.3.1 handle_simple_irq  
  
	handle_simple_irq(unsigned int irq, struct irq_desc *desc)
	{
		raw_spin_lock(&desc->lock);

		if (unlikely(irqd_irq_inprogress(&desc->irq_data)))
			if (!irq_check_poll(desc))
				goto out_unlock;

		desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
		kstat_incr_irqs_this_cpu(irq, desc);

		if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
			desc->istate |= IRQS_PENDING;
			goto out_unlock;
		}

		handle_irq_event(desc);

	out_unlock:
		raw_spin_unlock(&desc->lock);
	}
  
该函数没有实现任何实质性的流控操作，在把irq_desc结构锁住后，直接调用handle_irq_event处理irq_desc中的action链表，它通常用于多路复用（类似于中断控制器级联）中的子中断，由父中断的流控回调中调用。或者用于无需进行硬件控制的中断中。  
  
#####6.3.2 handle_level_irq
  
	void handle_level_irq(unsigned int irq, struct irq_desc *desc)
	{
		raw_spin_lock(&desc->lock);
		mask_ack_irq(desc);

		if (unlikely(irqd_irq_inprogress(&desc->irq_data)))
			if (!irq_check_poll(desc))
				goto out_unlock;

		desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
		kstat_incr_irqs_this_cpu(irq, desc);

		/*
		 * If its disabled or no action available
	 	* keep it masked and get out of here
		 */
		if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
			desc->istate |= IRQS_PENDING;
			goto out_unlock;
		}

		handle_irq_event(desc);

		cond_unmask_irq(desc);

	out_unlock:
		raw_spin_unlock(&desc->lock);
	}
  
电平中断的特点是，只要设备的中断请求引脚（中断线）保持在预设的触发电平，中断就会一直被请求，所以，为了避免同一中断被重复响应，必须在处理中断前先把mask irq，然后ack irq，以便复位设备的中断请求引脚，响应完成后再unmask irq。实际的情况稍稍复杂一点，在mask和ack之后，还要判断IRQ_INPROGRESS标志位，如果该标志已经置位，则直接退出，不再做实质性的处理，IRQ_INPROGRESS标志在handle_irq_event的开始设置，在handle_irq_event结束时清除，如果监测到IRQ_INPROGRESS被置位，表明该irq正在被另一个CPU处理中，所以直接退出，对电平中断来说是正确的处理方法。  
  
因为电平中断的特性：只要没有ack irq，中断线会一直有效，所以我们不会错过某次中断请求，但是驱动程序的开发人员如果对该过程理解不透彻，特别容易发生某次中断被多次处理的情况。特别是使用了中断线程来响应中断的时候：通常mask_ack_irq只会清除中断控制器的pending状态，很多慢速设备（例如通过i2c或spi控制的设备）需要在中断线程中清除中断线的pending状态，但是未等到中断线程被调度执行的时候，handle_level_irq早就返回了，这时已经执行过unmask_irq，设备的中断线pending处于有效状态，中断控制器会再次发出中断请求，结果是设备的一次中断请求，产生了两次中断响应。要避免这种情况，最好的办法就是不要单独使用中断线程处理中断，而是要实现request_threaded_irq()的第二个参数irq_handler_t：handler，在handle回调中使用disable_irq()关闭该irq，然后在退出中断线程回调前再enable_irq()  
  
#####6.3.3 handle_edge_irq  
  
处理流程如下：

	void handle_edge_irq(unsigned int irq, struct irq_desc *desc)
	{
		raw_spin_lock(&desc->lock);

		desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
		/*
	 	* If we're currently running this IRQ, or its disabled,
	 	* we shouldn't process the IRQ. Mark it pending, handle
	 	* the necessary masking and go out
	 	*/
		if (unlikely(irqd_irq_disabled(&desc->irq_data) ||
		     	irqd_irq_inprogress(&desc->irq_data) || !desc->action)) {
			if (!irq_check_poll(desc)) {
				desc->istate |= IRQS_PENDING;
				mask_ack_irq(desc);
				goto out_unlock;
			}
		}
		kstat_incr_irqs_this_cpu(irq, desc);

		/* Start handling the irq */
		desc->irq_data.chip->irq_ack(&desc->irq_data);

		do {
			if (unlikely(!desc->action)) {
             	mask_irq(desc);
				goto out_unlock;
			}

			/*
		 	* When another irq arrived while we were handling
		 	* one, we could have masked the irq.
		 	* Renable it, if it was not disabled in meantime.
		 	*/
			if (unlikely(desc->istate & IRQS_PENDING)) {
				if (!irqd_irq_disabled(&desc->irq_data) &&
			    	irqd_irq_masked(&desc->irq_data))
					unmask_irq(desc);
			}

			handle_irq_event(desc);

		} while ((desc->istate & IRQS_PENDING) &&
		 	!irqd_irq_disabled(&desc->irq_data));

	out_unlock:
		raw_spin_unlock(&desc->lock);
	}
    
边沿触发中断的特点是，只有设备的中断请求引脚（中断线）的电平发生跳变时（由高变低或者有低变高），才会发出中断请求，因为跳变是一瞬间，而且不会像电平中断能保持住电平，所以处理不当就特别容易漏掉一次中断请求，为了避免这种情况，屏蔽中断的时间必须越短越好。内核的开发者们显然意识到这一点，在正是处理中断前，判断IRQ_PROGRESS标志没有被设置的情况下，只是ack irq，并没有mask irq，以便复位设备的中断请求引脚，在这之后的中断处理期间，另外的cpu可以再次响应同一个irq请求，如果IRQ_PROGRESS已经置位，表明另一个CPU正在处理该irq的上一次请求，这种情况下，他只是简单地设置IRQS_PENDING标志，然后mask_ack_irq后退出，中断请求交由原来的CPU继续处理。因为是mask_ack_irq，所以系统实际上只允许挂起一次中断。  
  
处理中断期间，另一次请求可能由另一个cpu响应后挂起，所以在处理完本次请求后还要判断IRQS_PENDING标志，如果被置位，当前cpu要接着处理被另一个cpu“委托”的请求。内核在这里设置了一个循环来处理这种情况，直到IRQS_PENDING标志无效为止，而且因为另一个cpu在响应并挂起irq时，会mask irq，所以在循环中要再次unmask irq，以便另一个cpu可以再次响应并挂起irq  
  
#####6.3.4 handle_fasteoi_irq  
  
	void handle_fasteoi_irq(unsigned int irq, struct irq_desc *desc)
	{
		raw_spin_lock(&desc->lock);

		if (unlikely(irqd_irq_inprogress(&desc->irq_data)))
			if (!irq_check_poll(desc))
				goto out;

		desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
		kstat_incr_irqs_this_cpu(irq, desc);

		/*
	 	* If its disabled or no action available
		 * then mask it and get out of here:
	 	*/
		if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
			desc->istate |= IRQS_PENDING;
			mask_irq(desc);
			goto out;
		}

		if (desc->istate & IRQS_ONESHOT)
			mask_irq(desc);

		preflow_handler(desc);
		handle_irq_event(desc);

		if (desc->istate & IRQS_ONESHOT)
			cond_unmask_irq(desc);

	out_eoi:
		desc->irq_data.chip->irq_eoi(&desc->irq_data);
	out_unlock:
		raw_spin_unlock(&desc->lock);
		return;
	out:
		if (!(desc->irq_data.chip->flags & IRQCHIP_EOI_IF_HANDLED))
			goto out_eoi;
		goto out_unlock;
	}
    
现代的中断控制器通常会在硬件上实现了中断流控功能，例如ARM体系中的GIC通用中断控制器。对于这种中断控制器，CPU只需要在每次处理完中断后发出一个end of interrupt（eoi），我们无需关注何时mask，何时unmask。不过虽然想着很完美，事情总有特殊的时候，所以内核还是给了我们插手的机会，它利用irq_desc结构中的preflow_handler字段，在正式处理中断前会通过preflow_handler函数调用该回调    
  
内核还提供了另外一个eoi版的函数：handle_edge_eoi_irq，它的处理类似于handle_edge_irq，只是无需实现mask和unmask的逻辑。  
  
#####6.3.5 handle_percpu_irq  
  
	void handle_percpu_irq(unsigned int irq, struct irq_desc *desc)
	{
		struct irq_chip *chip = irq_desc_get_chip(desc);

		kstat_incr_irqs_this_cpu(irq, desc);

		if (chip->irq_ack)
			chip->irq_ack(&desc->irq_data);

		handle_irq_event_percpu(desc, desc->action);

		if (chip->irq_eoi)
			chip->irq_eoi(&desc->irq_data);
	}
    
该函数用于smp系统，当某个irq只在一个cpu上处理时，我们可以无需用自旋锁对数据进行保护，也无需处理cpu之间的中断嵌套重入，所以函数很简单  
  
#####6.3.6 handle_nested_irq  
  
	void handle_nested_irq(unsigned int irq)
	{
		struct irq_desc *desc = irq_to_desc(irq);
		struct irqaction *action;
		irqreturn_t action_ret;

		might_sleep();

		raw_spin_lock_irq(&desc->lock);

		desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
		kstat_incr_irqs_this_cpu(irq, desc);

		action = desc->action;
		if (unlikely(!action || irqd_irq_disabled(&desc->irq_data))) {
			desc->istate |= IRQS_PENDING;
			goto out_unlock;
		}

		irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
		raw_spin_unlock_irq(&desc->lock);

		action_ret = action->thread_fn(action->irq, action->dev_id);
		if (!noirqdebug)
			note_interrupt(irq, desc, action_ret);

		raw_spin_lock_irq(&desc->lock);
		irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);

	out_unlock:
		raw_spin_unlock_irq(&desc->lock);
	}
    
该函数用于实现其中一种中断共享机制，当多个中断共享某一根中断线时，我们可以把这个中断线作为父中断，共享该中断的各个设备作为子中断，在父中断的中断线程中决定和分发响应哪个设备的请求，在得出真正发出请求的子设备后，调用handle_nested_irq来响应中断。所以，该函数是在进程上下文执行的，我们也无需扫描和执行irq_desc结构中的action链表。父中断在初始化时必须通过irq_set_nested_thread函数明确告知中断子系统：这些子中断属于线程嵌套中断类型，这样驱动程序在申请这些子中断时，内核不会为它们建立自己的中断线程，所有的子中断共享父中断的中断线程。  
  
针对中断控制器的级联的情况， 通常调用irq_set_chained_handler 来设置父中断的中断流控处理函数， 它同时设置IRQ_NOREQUEST、IRQ_NOPROBE、IRQ_NOTHREAD标志， 使得驱动不能再申请父中断的中断线程。
  
###7. 共享断的设备的中断处理  
  
移动设备系统中，存在着大量的多功能复合设备，最常见的是一个芯片中，内部集成了多个功能部件，或者是一个模块单元内部集成了功能部件，这些内部功能部件可以各自产生中断请求，但是芯片或者硬件模块对外只有一个中断请求引脚。或者是中断控制器的级联，也是相同的情况。  
  
对于这种情况， 可以使用多种方式处理这些设备的中断请求  
  
####7.1 单一中断模式  
  
对于这种复合设备，通常设备中会提供某种方式，以便让CPU获取真正的中断来源， 方式可以是一个内部寄存器，gpio的状态等等。单一中断模式是指驱动程序只申请一个irq，然后在中断处理程序中通过读取设备的内部寄存器，获取中断源，然后根据不同的中断源做出不同的处理  
  
####7.2 共享中断模式  
  
共享中断模式充分利用了通用中断子系统的特性，经过前面的讨论，我们知道，irq对应的irq_desc结构中的action字段，本质上是一个链表，这给我们实现中断共享提供了必要的基础，只要我们以相同的irq编号多次申请中断服务，那么，action链表上就会有多个irqaction实例，当中断发生时，中断子系统会遍历action链表，逐个执行irqaction实例中的handler回调，根据handler回调的返回值不同，决定是否唤醒中断线程。需要注意到是，申请多个中断时，irq编号要保持一致，flag参数最好也能保持一致，并且都要设上IRQF_SHARED标志。在使用共享中断时，最好handler和thread_fn都要提供，在各自的中断处理回调handler中，做出以下处理：  
  
1. 判断如果不是来自本设备：直接返回IRQ_NONE
2. 如果是来自本设备: 关闭irq, 返回IRQ_WAKE_THREAD，唤醒中断线程，thread_fn将会被执行  
  
####7.3 中断控制器级联模式  
  
多数多功能复合设备内部提供了基本的中断控制器功能，例如可以单独地控制某个子中断的打开和关闭，并且可以方便地获得子中断源，对于这种设备，我们可以把设备内的中断控制器实现为一个子控制器，然后使用中断控制器级联模式。这种模式下，各个子设备拥有各自独立的irq编号，中断服务通过父中断进行分发。   
  
对于父中断，具体的实现步骤如下：
  
1. 首先，父中断的irq编号可以从板级代码的预定义中获得，或者通过device的platform_data字段获得
2. 使用父中断的irq编号，利用irq_set_chained_handler函数修改父中断的流控函数
3. 使用父中断的irq编号，利用irq_set_handler_data设置流控函数的参数，该参数要能够用于判别子控制器的中断来源
4. 实现父中断的流控函数，其中只需获得并计算子设备的irq编号，然后调用generic_handle_irq即可
  
对于子设备， 具体的实现步骤如下：  
  
1. 为设备内的中断控制器实现一个irq_chip结构，实现其中必要的回调，例如irq_mask，irq_unmask，irq_ack等  
2. 循环每一个子设备，做余下的动作  
3. 为每个子设备，使用irq_alloc_descs函数申请irq编号
4. 使用irq_set_chip_data设置必要的cookie数据
5. 使用irq_set_chip_and_handler设置子控制器的irq_chip实例和子irq的流控处理程序，通常使用标准的流控函数，例如handle_edge_irq  
  
子设备的驱动程序使用自身申请到的irq编号，按照正常流程申请中断服务即可。  
使用这种方式，需在父中断的流控处理函数中读取子中断源， 流控处理函数运行在中断上下文， 如果是通过慢速总线(例如i2c)来访问设备就不太合适  
  
irq_set_chained_handler()  该API用于设置根控制器与子控制器相连的irq所对应的irq_desc.handle_irq回调函数，并且设置IRQ_NOPROBE和IRQ_NOTHREAD以及IRQ_NOREQUEST标志，这几个标志保证驱动程序不会错误地申请该irq，因为该irq已经被作为级联irq使用  
 
####7.4 中断线程嵌套模式   
 
该模式与中断控制器级联模式大体相似，只不过级联模式时，父中断无需通过request_threaded_irq申请中断服务，而是直接更换了父中断的流控回调，在父中断的流控回调中实现子中断的二次分发。但是这在有些情况下会给我们带来不便，因为流控回调要获取子控制器的中断源，而流控回调运行在中断上下文中，对于那些子控制器需要通过慢速总线访问的设备，在中断上下文中访问显然不太合适，这时我们可以把子中断分发放在父中断的中断线程中进行，这就是我所说的所谓中断线程嵌套模式。下面是大概的实现过程：  
  
对于父中断，具体的实现步骤如下：

1. 首先，父中断的irq编号可以从板级代码的预定义中获得，或者通过device的platform_data字段获得；
2. 使用父中断的irq编号，利用request_threaded_irq函数申请中断服务，需要提供thread_fn参数和dev_id参数dev_id参数要能够用于判别子控制器的中断来源
3. 实现父中断的thread_fn函数，其中只需获得并计算子设备的irq编号，然后调用handle_nested_irq即可  
  
对于子设备，具体的实现步骤如下：  
  
1. 为设备内的中断控制器实现一个irq_chip结构，实现其中必要的回调，例如irq_mask，irq_unmask，irq_ack等
2. 循环每一个子设备，做余下动作：
3. 为每个子设备，使用irq_alloc_descs函数申请irq编号
4. 使用irq_set_chip_data设置必要的cookie数据
5. 使用irq_set_chip_and_handler设置子控制器的irq_chip实例和子irq的流控处理程序，通常使用标准的流控函数，例如handle_edge_irq
6. 使用irq_set_nested_thread函数，把子设备irq的线程嵌套特性打开

子设备的驱动程序使用自身申请到的irq编号，按照正常流程申请中断服务即可  
  
因为子设备irq的线程嵌套特性被打开，使用request_threaded_irq申请子设备的中断服务时，即是是提供了handler参数，中断子系统也不会使用它，同时也不会为它创建中断线程，子设备的thread_fn回调是在父中断的中断线程中，通过handle_nested_irq调用的，也就是说，尽管子中断有自己独立的irq编号，但是它们没有独立的中断线程，只是共享了父中断的中断服务线程
