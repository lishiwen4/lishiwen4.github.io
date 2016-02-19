---
layout: post
title: "linux per-cpu 变量"
description: 
category: linux-kernel
tags: [linux, driver]
mathjax: 
chart:
comments: false
---

###1. per-cpu 变量

per-cpu data 是内核为smp系统中不同CPU之间的数据保护方式，系统为每个CPU维护一段私有的空间用于存放per-cpu  data，在这段空间中的数据只有这个CPU能访问， 即per-cpu data 在每一个cpu上都有一份副本， 每一个cpu只会访问自己的副本， 因此， 对per-cpu data的访问几乎不需要锁

可以看出， per-cpu data基本上只能在特殊情况下， 也就是当它确定在系统的各cpu上的逻辑上独立的时候才适合使用

per-cpu data常用于计数， 队列， tasklet、timer_list等机制都使用了per-CPU技术

**虽然对per-cpu data的访问不许要加锁， 但是避免它被切换到另一个CPU上被处理， 因此， 在访问时， 还需要禁止抢占**

###2. 静态定义per-cpu data

相关的API定义在 “linux/percpu-defs.h” 中

	// 没有特殊要求的 per-cpu data 的定义接口
	DECLARE_PER_CPU(type, name)
	DEFINE_PER_CPU(type, name)

	// 通过该接口定义的per-cpu存放在对应的section的最前面 (per-cpu data 存放在一些特殊的section中)
	DECLARE_PER_CPU_FIRST(type, name)
	DEFINE_PER_CPU_FIRST(type, name)

	// 无论是SMP还是UP上， per-cpu data会对其带 L1 cache line
	DECLARE_PER_CPU_ALIGNED(type, name)
	DEFINE_PER_CPU_ALIGNED(type, name)

	// 在SMP上， per-cpu data 会对齐到 L1 cache line， 而在UP上则不会对齐
	DECLARE_PER_CPU_SHARED_ALIGNED(type, name)
	DEFINE_PER_CPU_SHARED_ALIGNED(type, name)

	// per-cpu data 会page align
	DECLARE_PER_CPU_PAGE_ALIGNED(type, name)
	DEFINE_PER_CPU_PAGE_ALIGNED(type, name)

	// per-cpu data 是read mostly的
	DECLARE_PER_CPU_READ_MOSTLY(type, name)
	DEFINE_PER_CPU_READ_MOSTLY(type, name)

示例

	DEFINE_PER_CPU(int,my_percpu); //声明一个变量
    	DEFINE_PER_CPU(int[3],my_percpu_array); //声明一个数组

###3. 访问静态定义的 per-cpu data

静态定义的per cpu变量不能象普通变量那样进行访问，需要使用特定的接口函数

可以直接获取per-cpu data， 执行的结果可以作为左值(即可以进行取地址操作)

	#define get_cpu_var(var) (*({				\
		preempt_disable();				\
		&__get_cpu_var(var); }))

	#define put_cpu_var(var) do {				\
		(void)&(var);					\
		preempt_enable();				\
	} while (0)

这两个接口函数已经内嵌了锁的机制（preempt disable），用户可以直接调用该接口进行本CPU上该变量副本的访问, 如果用户确认当前的执行环境已经是preempt disable，那么可以使用lock-free版本(只需要get， 不需要put)

	__get_cpu_var(var)

访问指定cpu上per-cpu data 的副本

	per_cpu(var, cpu)		

示例

	DEFINE_PER_CPU(int,my_percpu);

	int *ptr；
	ptr = get_cpu_var(my_percpu);
	......
	put_cpu_var(my_percpu);

###4. 动态分配/释放per-cpu data

	alloc_percpu(type)

	void free_percpu(void __percpu *ptr)
	
###5. 访问动态分配的per-cpu data

动态分配的 per-cpu data 需要通过指针来操作

	#define get_cpu_ptr(var) ({				\
		preempt_disable();				\
		this_cpu_ptr(var); })

	#define put_cpu_ptr(var) do {				\
		(void)(var);					\
		preempt_enable();				\
	} while (0)

这两个接口函数已经内嵌了锁的机制（preempt disable），用户可以直接调用该接口进行本CPU上该变量副本的访问, 如果用户确认当前的执行环境已经是preempt disable，那么可以使用lock-free版本(只需要get， 不需要put)

	__this_cpu_ptr(ptr)

获取指定的cpu上的per-data 副本的指针， 使用

	per_cpu_ptr(ptr, cpu)

###6. 静态 per-cpu data 的实现

	#define DEFINE_PER_CPU(type, name)					\
		DEFINE_PER_CPU_SECTION(type, name, "")

	#define DEFINE_PER_CPU_SECTION(type, name, sec)				\
		__PCPU_DUMMY_ATTRS char __pcpu_scope_##name;			\
		extern __PCPU_DUMMY_ATTRS char __pcpu_unique_##name;		\
		__PCPU_DUMMY_ATTRS char __pcpu_unique_##name;			\
		__PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES __weak			\
		__typeof__(type) name

	#define __PCPU_DUMMY_ATTRS						\
		__attribute__((section(".discard"), unused))

	#define __PCPU_ATTRS(sec)						\
		__percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))	\
		PER_CPU_ATTRIBUTES

假设使用 DEFINE_PER_CPU() 来静态定义一个per-cpu data

	DEFINE_PER_CPU(int, my_percpu)

那么展开后可得

	__attribute__((section(".discard"), unused))  char __pcpu_scope_my_percpu;
	extern __attribute__((section(".discard"), unused)) char __pcpu_uinique_my_percpu;
	__attribute__((section(".discard"), unused)) char __pcpu_uinique_my_percpu;
	__percpu __attribute__((section(PER_CPU_BASE_SECTION ""))) PER_CPU_ATTRIBUTES __weak __typeof__(type) my_percpu

继续看其中的一些定义

	#define PER_CPU_BASE_SECTION ".data..percpu"
	#define PER_CPU_DEF_ATTRIBUTES
	# define __percpu

而 __typeof__(var) 是gcc对C语言的一个扩展的关键字，用于声明变量类型,var可以是数据类型（int， char*..),也可以是变量表达式， 继续展开， 可得

	__attribute__((section(".discard"), unused))  char __pcpu_scope_my_percpu;
	extern __attribute__((section(".discard"), unused)) char __pcpu_uinique_my_percpu;
	__attribute__((section(".discard"), unused)) char __pcpu_uinique_my_percpu;
	__attribute__((section(".data..percpu" ""))) __weak int my_percpu

__attribute__( section("xxxxx") ) 表示在编译时将变量或者函数存储在指定的section中， __attribute__( unused ) 表示符号可能不会被使用， 用于消除编译器的警告， __weak 表示该符号是一个弱符号， 若其它位置定义了相同的强符号， 则该处会被覆盖， 最终， 静态声明一个 int 型的 per-cpu data  my_percpu时， 完成了如下的工作:

1. 定义一个char变量 __pcpu_scope_my_percpu， 存放在 ".discard" section
2. 定义一个char变量 __pcpu_uinique_my_percpu， 存放在 ".discard" section
3. 定义一个int变量 my_percpu， 存放在 ".data..percpu" section

再来看其它的 DEFINE_PER_CPU_xxx 宏

	#define DEFINE_PER_CPU_FIRST(type, name)				\
		DEFINE_PER_CPU_SECTION(type, name, PER_CPU_FIRST_SECTION)

	#define DEFINE_PER_CPU_ALIGNED(type, name)				\
		DEFINE_PER_CPU_SECTION(type, name, PER_CPU_ALIGNED_SECTION)	\
		____cacheline_aligned

	......

可见， 它们在调用 DEFINE_PER_CPU_SECTION 宏时第3个参数不同， 因此， 会被放置在不同的section里面， 总结如下

	/* DEFINE_PER_CPU()	*/
	--------------------------------------------------------
			|	SMP			|	UP
	--------------------------------------------------------
	built-in	| ".data..precpu"		|	".data"
	--------------------------------------------------------
	module		| ".data..precpu"		|	".data"
	--------------------------------------------------------


	/* DEFINE_PER_CPU_FIRST() */
	--------------------------------------------------------
			|	SMP			|	UP
	--------------------------------------------------------
	built-in	| ".data..precpu..first"	|	".data"
	--------------------------------------------------------
	module		| ".data..precpu..first"	|	".data"
	--------------------------------------------------------


	/* DEFINE_PER_CPU_SHARED_ALIGNED() */
	-----------------------------------------------------------------
			|	SMP			|	UP
	-----------------------------------------------------------------
	built-in	| ".data..precpu..shared_aligned"	|	".data"
	-----------------------------------------------------------------
	module		| ".data..precpu"			|	".data"
	-----------------------------------------------------------------

	/* DEFINE_PER_CPU_ALIGNED() */
	---------------------------------------------------------------------------------------
			|	SMP			|	UP
	---------------------------------------------------------------------------------------
	built-in	| ".data..precpu..shared_aligned"	|	".data..shared_aligned"
	---------------------------------------------------------------------------------------
	module		| ".data..precpu"			|	".data..shared_aligned"
	---------------------------------------------------------------------------------------

	/* DEFINE_PER_CPU_PAGE_ALIGNED() */
	---------------------------------------------------------------------------------------
			|	SMP			|	UP
	---------------------------------------------------------------------------------------
	built-in	| ".data..precpu..page_aligned"	|	".data..page_aligned"
	---------------------------------------------------------------------------------------
	module		| ".data..precpu..page_aligned"	|	".data..shared_aligned"
	---------------------------------------------------------------------------------------

	/* DEFINE_PER_CPU_READ_MOSTLY() */
	---------------------------------------------------------------------------------------
			|	SMP			|	UP
	---------------------------------------------------------------------------------------
	built-in	| ".data..precpu..readmostly"	|	".data..readmostly"
	---------------------------------------------------------------------------------------
	module		| ".data..precpu..readmostly"	|	".data..readmostly"
	---------------------------------------------------------------------------------------

在 vmlinux.lds.h文件 中有

	#define PERCPU_INPUT(cacheline)                     \
    		VMLINUX_SYMBOL(__per_cpu_start) = .;                \
    		*(.data..percpu..first)                     \
    		. = ALIGN(PAGE_SIZE);                       \
    		*(.data..percpu..page_aligned)                  \
    		. = ALIGN(cacheline);                       \
    		*(.data..percpu..readmostly)                    \
    		. = ALIGN(cacheline);                       \
    		*(.data..percpu)                        \
    		*(.data..percpu..shared_aligned)                \
    		VMLINUX_SYMBOL(__per_cpu_end) = .;

这些section位于 __per_cpu_start 和 __per_cpu_end 之间


配合链接脚本， linux kernel定义了

	#define PERCPU_VADDR(cacheline, vaddr, phdr)				\
	VMLINUX_SYMBOL(__per_cpu_load) = .;				\
	.data..percpu vaddr : AT(VMLINUX_SYMBOL(__per_cpu_load)		\
				- LOAD_OFFSET) {			\
		PERCPU_INPUT(cacheline)					\
	} phdr								\
	. = VMLINUX_SYMBOL(__per_cpu_load) + SIZEOF(.data..percpu);

**无论静态还是动态per cpu变量的分配，其机制都是一样的，只不过，对于静态per cpu变量，需要在系统初始化的时候，对应per cpu section，预先动态分配一个同样size的per cpu chunk**

###7. 动态分配的per-cpu data的实现

per-cpu的总体的内存管理信息被收集在struct pcpu_alloc_info结构中

	struct pcpu_alloc_info {
    		size_t static_size;	//静态定义的percpu变量占用内存区域长度
    		size_t reserved_size;	//预留区域，在percpu内存分配指定为预留区域分配时，将使用该区域
    		size_t dyn_size;		//动态分配的percpu变量占用内存区域长度
    		size_t unit_size;		//每颗处理器的percpu虚拟内存递进基本单位，每个cpu的percpu空间所占得内存空间为一个unit, 每个unit的大小记为unit_size
    		size_t atom_size;		//PAGE_SIZE
    		size_t alloc_size;		//要分配的percpu内存空间
    		size_t __ai_size;		//整个pcpu_alloc_info结构体的大小
    		int nr_groups;		//该架构下的处理器分组数目
    		struct pcpu_group_info groups[];	//该架构下的处理器分组信息
	};
内核使用struct pcpu_group_info结构来表示处理器的分组信息

	struct pcpu_group_info {
    		int nr_units; 		//该组的处理器数目
    		unsigned long base_offset;	//组的percpu内存地址起始地址，即组内处理器数目×处理器percpu虚拟内存递进基本单位
    		unsigned int *cpu_map; 	//组内cpu对应数组，保存属于该分组的cpu的id
	};


	
	struct pcpu_chunk {
    		struct list_head list;		//用来把chunk链接起来形成链表， chunk中空闲空间的大小链表又被放到pcpu_slot数组中
    		int free_size; 			//chunk中的空闲大小
    		int contig_hint; 			//该chunk中最大的可用空间的map项的size
    		void *base_addr; 			//percpu内存开始基地值
    		int map_used; 			//该chunk中使用了多少个map项
    		int map_alloc; 			//记录map数组的项数，为PERCPU_DYNAMIC_EARLY_SLOTS=128
    							//若map项>0,表示该map中记录的size是可以用来分配percpu空间的。
    							//若map项<0,表示该map项中的size已经被分配使用。
    		int *map; 			//map数组，记录该chunk的空间使用情况
    		void *data; 			//chunk data
    		bool immutable; 			/* no [de]population allowed */
    		unsigned long populated[]; 		/* populated bitmap */
	};

linux kernel的初始化的过程中

	start_kernel()
		setup_per_cpu_areas()
			pcpu_embed_first_chunk()
			

	void __init setup_per_cpu_areas(void)
	{
		......
		// 建立第一个chunk， PERCPU_MODULE_RESERVE 是为模块中的 per-cpu data预留的空间
		rc = pcpu_embed_first_chunk(PERCPU_MODULE_RESERVE,
				    PERCPU_DYNAMIC_RESERVE, PAGE_SIZE, NULL,
				    pcpu_dfl_fc_alloc, pcpu_dfl_fc_free);
		......

		// __per_cpu_start 是存储静态分配的per-cpu data的section在kernel镜像中的起始地址
		// pcpu_base_addr是kernel为per-cpu分配的一大段内存的起始地址， 这一大段内存划分为若干小块，每一个cpu拥有一个独立的小块 
		// 计算每一个cpu的独立的per-cpu 内存块相对于 __per_cpu_start 的偏移 
		// pcpu_unit_offsets[cpu]保存了对应的cpu的独立的per-cpu 内存块相对于pcpu_base_addr的偏移
		// __per_cpu_offset[cpu]保存了对应的cpu的独立的per-cpu 内存块相对于__per_cpu_start的偏移
		delta = (unsigned long)pcpu_base_addr - (unsigned long)__per_cpu_start;
		for_each_possible_cpu(cpu)
			__per_cpu_offset[cpu] = delta + pcpu_unit_offsets[cpu];
	}


	int __init pcpu_embed_first_chunk(size_t reserved_size, size_t dyn_size,
				  size_t atom_size,
				  pcpu_fc_cpu_distance_fn_t cpu_distance_fn,
				  pcpu_fc_alloc_fn_t alloc_fn,
				  pcpu_fc_free_fn_t free_fn)
	{
		......

		// 收集该架构下的per-cpu data的相关信息， 存储在 struct pcpu_alloc_info结构中
		ai = pcpu_build_alloc_info(reserved_size, dyn_size, atom_size,
				   cpu_distance_fn);
		......

		// 计算每一个cpu所需的per-cpu data内存空间的大小， 包括静态分配所需的空间， 模块静态分配预留的空间， 动态分配所需的空间
		size_sum = ai->static_size + ai->reserved_size + ai->dyn_size;

		// areas用来保存每个group的percpu内存起始地址，为其分配空间，做临时存储使用，用完释放掉
		areas_size = PFN_ALIGN(ai->nr_groups * sizeof(void *));		
		areas = alloc_bootmem_nopanic(areas_size);
	
		......

		// 为每一个cpu group分配内存区域
		for (group = 0; group < ai->nr_groups; group++) {
			struct pcpu_group_info *gi = &ai->groups[group];
			unsigned int cpu = NR_CPUS;
			void *ptr;

			......

			// 为每一个cpu group分配一段内存用于存储per-cpu data， 大小为 group中cpu数目 * 每一个cpu的per-cpu data 数目
			ptr = alloc_fn(cpu, gi->nr_units * ai->unit_size, atom_size);
			if (!ptr) {
				rc = -ENOMEM;
				goto out_free_areas;
			}
			......

			// 所有group 的per-cpu内存的起始地址存放在 areas数组中
			areas[group] = ptr;

			// 将所有group 的per-cpu内存的起始地址的最小值保存在base中
			base = min(ptr, base);
		}

		// 将为每一个cpu group 分配的内存空间划分给其中的每一个cpu
		for (group = 0; group < ai->nr_groups; group++) {
			struct pcpu_group_info *gi = &ai->groups[group];
			void *ptr = areas[group];

			for (i = 0; i < gi->nr_units; i++, ptr += ai->unit_size) {
				if (gi->cpu_map[i] == NR_CPUS) {
					// NR_CPUS 是系统支持的cpu的最大数目， 分配内存时按最大数目来分配， 实际上可能有一部分内存不会被使用，释放掉
					free_fn(ptr, ai->unit_size);
					continue;
				}
				
				// 将静态定义的per-cpu变量拷贝到每个cpu的per-cpu内存起始地址
				memcpy(ptr, __per_cpu_load, ai->static_size);

				// 释放每一个cpu的per-cpu内存中的多余的空间
				// 所需的空间为ai->static_size + ai->reserved_size + ai->dyn_size， 实际分配了ai->unit_size
				free_fn(ptr + size_sum, ai->unit_size - size_sum);
			}
		}

		//计算每一个cpu group的per-cpu内存区的起始地址的偏移量(相对于起始地址最小的那一个)
		max_distance = 0;
		for (group = 0; group < ai->nr_groups; group++) {
			ai->groups[group].base_offset = areas[group] - base;
			max_distance = max_t(size_t, max_distance,
				     ai->groups[group].base_offset);
		}

		// 计算出per-cpu 内存区中的最大偏移量
		max_distance += ai->unit_size;

		// 检查最大偏移量是否超过vmalloc空间的75%
		......

		// 为per-cpu data建立第一个chunk
		rc = pcpu_setup_first_chunk(ai, base);
		.....
		return rc;
	}

per-cpu 内存被使用struct pcpu_chunk来描述， 不同大小的chunk被放在不同的链表中，在动态分配per-cpu data时， 从合适的chunk中分配所需的空间

