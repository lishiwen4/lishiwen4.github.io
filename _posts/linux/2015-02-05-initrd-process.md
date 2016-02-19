---
layout: post
title: "initrd 处理流程"
description:
category: linux
tags: [linux, build-link]
mathjax: 
chart:
comments: false
---

###1. initrd处理流程  
  
从linux 2.4到linux 2.6，inird发展了不同的版本， kernel启动时，对initrd的处理流程也不相同。  
  
###2. linux 2.4对initrd的处理  
  
1.bootloader 将 kernel 以及 /dev/initrd（由bootloader初始化，存放initrd） 中的内容加载到内存  
2.在内核初始化的过程中，将 /dev/initrd 中的内容解压缩并拷贝到 /dev/ram0 （ramdisk）上  
3.内核以刻度写的方式将 /dev/ram0 挂载为原始的根文件系统  
4.如果 /dev/ram0 被指定为真正的根文件系统，那么内核跳至最后一步正常启动  
5.执行 initrd 上的 /linuxrc 文件，linuxrc 通常是一个脚本文件，负责加载内核访问根文件系统必须的驱动， 以及加载根文件系统  
6./linuxrc 执行完毕，真正的根文件系统被挂载  
7.如果真正的根文件系统存在 /initrd 目录，那么 /dev/ram0 将从 / 移动到 /initrd。否则如果 /initrd 目录不存在， /dev/ram0 将被卸载  
8.在真正的根文件系统上进行正常启动过程 ，执行 /sbin/init。 linux2.4 内核的 initrd 的执行是作为内核启动的一个中间阶段，也就是说 initrd 的 /linuxrc 执行以后，内核会继续执行初始化代码，我们后面会看到这是 linux2.4 内核同 2.6 内核的 initrd 处理流程的一个显著区别  
  
###3. linux 2.6对initrd的处理流程  
  
linux2.6 内核支持两种格式的 initrd，一种是 linux2.4 内核那种传统格式的文件系统镜像－image-initrd，它的制作方法同 Linux2.4 内核的 initrd 一样，其核心文件就是 “/linuxrc”，另外一种格式的 initrd 是 cpio 格式的，这种格式的 initrd 从 linux2.5 起开始引入，使用 cpio 工具生成，其核心文件不再是 “/linuxrc”，而是 “/init”，本文将这种 initrd 称为 cpio-initrd。尽管 linux2.6 内核对 cpio-initrd和 image-initrd 这两种格式的 initrd 均支持，但对其处理流程有着显著的区别，下面分别介绍 linux2.6 内核对这两种 initrd 的处理流程。  
  
####3.1 初始化rootfs  
  
先从古老的start_kernel() 函数说起。
  
	asmlinkage void __init start_kernel(void)
	{
		......
	[1]	vfs_caches_init(totalram_pages);
		......
	[2]    	rest_init();
		......
	}  
    
	void __init vfs_caches_init(unsigned long mempages)
	{
		......
		[3]    	mnt_init();
		......
	}  
    
	void __init mnt_init(void)
	{
    		......
	[4]    	init_rootfs();
	[5]	init_mount_tree();
	}
    
	int __init init_rootfs(void)
	{
		......
	[6]	err = register_filesystem(&rootfs_fs_type);
		......
	}

	static void __init init_mount_tree(void)
	{
		......
	[7]		mnt = vfs_kern_mount(type, 0, "rootfs", NULL);
		......
	[8]		set_fs_pwd(current->fs, &root);
	[9]		set_fs_root(current->fs, &root);
	}

+ 代码[1]: 调用vfs_caches_init来初始化虚拟内存系统
+ 代码[6]: [1] -> [3] -> [4] -> [6] 注册类型为rootfs（其实就是RAMFS），名为rootfs的文件系统， rootfs是一个基于内存的文件系统，是linux在初始化时加载的第一个文件系统。   
+ 代码[7]: [1] -> [3] -> [5] -> [7] 挂载上一步注册的rootfs到根目录“/”（该函数会为VFS系统建立根目录“/”）  
+ 代码[8]: [1] -> [3] -> [5] -> [8] 将"/"目录设置为当前进程的当前目录(会被其fork出的子进程所继承)  
+ 代码[9]: [1] -> [3] -> [5] -> [9] 将"/"目录设置为当前进程的根目录(会被其fork出的子进程所继承)   
  
####3.2 解压缩initrd  

	static noinline void __init_refok rest_init(void)
	{
	[10]	kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
		......
	[11]	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	}
    
	static int __ref kernel_init(void *unused)
	{
	[12]	kernel_init_freeable();
		......
	}
    
	static noinline void __init kernel_init_freeable(void)
	{
		......
	[13]	do_basic_setup();       
		......
	}
    
	static void __init do_basic_setup(void)
	{
		......
	[14]	do_initcalls();
		......
	}
    
	static void __init do_initcalls(void)
	{
		......
	[15]	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
			do_initcall_level(level);
	}
    
	rootfs_initcall(populate_rootfs);
    
代码[15]: [2] -> [10] -> [12] -> [13] -> [14] -> [15] 调用各个level的module_init函数， 包括initramfs的初始化函数populate_rootfs（）  

	static int __init populate_rootfs(void)
	{
	[16]	char *err = unpack_to_rootfs(__initramfs_start, __initramfs_size);
			if (err)
				panic(err);	/* Failed to decompress INTERNAL initramfs */
	[17]	if (initrd_start) {
			#ifdef CONFIG_BLK_DEV_RAM
				int fd;
				printk(KERN_INFO "Trying to unpack rootfs image as initramfs...\n");
	[18]		err = unpack_to_rootfs((char *)initrd_start,
				initrd_end - initrd_start);
				if (!err) {
					free_initrd();
					goto done;
				} else {
					clean_rootfs();
	[19]			unpack_to_rootfs(__initramfs_start, __initramfs_size);
				}
					printk(KERN_INFO "rootfs image is not initramfs (%s)"
						"; looks like an initrd\n", err);
				fd = sys_open("/initrd.image",
			      		O_WRONLY|O_CREAT, 0700);
				if (fd >= 0) {
	[20]			sys_write(fd, (char *)initrd_start,
							initrd_end - initrd_start);
					sys_close(fd);
					free_initrd();
				}
			done:
			#else
				printk(KERN_INFO "Unpacking initramfs...\n");
	[21]		err = unpack_to_rootfs((char *)initrd_start,
					initrd_end - initrd_start);
				if (err)
					printk(KERN_EMERG "Initramfs unpacking failed: %s\n", err);
				free_initrd();
			#endif
			load_default_modules();
		}
			return 0;$
	}
  
+ 代码[16]: 假设initrd和内核链接在一起（内核的”.init.ramfs“数据段），将其解压到rootfs中。  
+ 代码[17]: 判断是否有单独的initrd文件，无论是cpio-initrd还是image-initrd,bootloader都会把initrd加载到init_start处  
+ 代码[18]: 试图把initrd文件作为initramfs解压到rootfs， 如果成功则跳到代码[21]  
+ 代码[19]: [18]失败， 将内核的”.init.ramfs“数据段解压缩到rootfs
+ 代码[20]： 单独的initrd文件是image-initrd， 将其写入到“/initrd.image”文件中  

继续看 __init kernel_init_freeable()

	static noinline void __init kernel_init_freeable(void)
	{
		......
	[13]	do_basic_setup();       
		......
            
	[22]	if (!ramdisk_execute_command)
			ramdisk_execute_command = "/init";

	[23]	if (sys_access((const char __user *) ramdisk_execute_command, 0) != 0) {
				ramdisk_execute_command = NULL;
				prepare_namespace();
		}
	}
  
+ 代码[22]:	当cmdline中存在”rdinit=xxx“时， ramdisk_execute_command才会被设置， 否则将“/init”设置为第一个要执行的程序  
+ 代码[23]:	检查是否能获取"/init"，对于cpio-initrd, 第一个执行的程序为”/init“, 对于image-initrd, 第一个执行的程序为"/linuxrc", 通过这个也能够判断使用的是image-initrd还是cpio-initrd，对于image-initrd,调用prepare_namespace()进一步处理  

###4. image-initrd额外的处理步骤

	void __init prepare_namespace(void)
	{
		......
	[24]   	if (initrd_load())
			goto out;
		......
	}
    
	int __init initrd_load(void)
	{
		if (mount_initrd) {
	[25]		create_dev("/dev/ram", Root_RAM0);
				/*
		 		* Load the initrd data into /dev/ram0. Execute it as initrd
		 		* unless /dev/ram0 is supposed to be our actual root device,
		 		* in that case the ram disk is just set up here, and gets
		 		* mounted in the normal path.
		 		*/
				if (rd_load_image("/initrd.image") && ROOT_DEV != Root_RAM0) {
					sys_unlink("/initrd.image");
	[26]			handle_initrd();
					return 1;
				}
			}
	[27]	sys_unlink("/initrd.image");
			return 0;
	}

+ 代码[25]:	创建"/dev/ram0"  
+ 代码[26]: 如果cmdline传递了正确的root=/dev/ramx的话， 说明你将“/dev/ram0”作为真正的根设备（有些嵌入式设备就这么做）作为真正的文件系统来处理， 否则，作为中间过度的文件系统， 此时，调用handle_initrd()来处理, 而在 handle_initrd()中会执行“/linuxrc”， 并挂载正真的文件系统.
+ 代码[27]: 删除"/initrd.img"文件.   

###5. cpio-initrd和image-initrd的处理步骤  

	static int __ref kernel_init(void *unused)
	{
	[12]	kernel_init_freeable();
	......
	[28]    if (ramdisk_execute_command) {
				if (!run_init_process(ramdisk_execute_command))
					return 0;
				pr_err("Failed to execute %s\n", ramdisk_execute_command);
			}
	[29]        if (execute_command) {
				if (!run_init_process(execute_command))
					return 0;
				pr_err("Failed to execute %s.  Attempting defaults...\n",
				execute_command);
			}
	[30]	if (!run_init_process("/sbin/init") ||
	    		!run_init_process("/etc/init") ||
	    		!run_init_process("/bin/init") ||
	    		!run_init_process("/bin/sh"))
				return 0;

	[31]	panic("No init found.  Try passing init= option to kernel. "
	      		"See Linux Documentation/init.txt for guidance.");
	}       
    
+ 代码[28]:如果 ramdisk_execute_command(cmdline中使用‘rdinit=xxx指定’)存在且执行成功则返回   
+ 代码[29]:如果 execute_command(cmdline中使用'init=xxx指定')存在且执行成功则返回  
+ 代码[30]:如果 “/sbin/init”, "/etc/init"， “/bin/init”， “/bin/sh”中有任一个存在且执行成功则返回  
+ 代码[31]:如果上述命令都不存在或执行失败,则打出“panic”信息，并hang住系统  

###6. image-initrd和cpio-initrd的处理差异  

1．cpio-initrd并没有使用额外的ramdisk,而是将其内容输入到rootfs中，其实rootfs本身也是一个基于内存的文件系统。这样就省掉了ramdisk的挂载、卸载等步骤。
2．cpio-initrd启动完/init进程，内核的任务就结束了，剩下的工作完全交给/init处理；而对于image-initrd，内核在执行完/linuxrc进程后，还要进行一些收尾工作，并且要负责执行真正的根文件系统的/sbin/init。cpio-initrd不再象image-initrd那样作为linux内核启动的一个中间步骤，而是作为内核启动的终点，内核将控制权交给cpio-initrd的/init文件后，内核的任务就结束了，所以在/init文件中，我们可以做更多的工作，而不比担心同内核后续处理的衔接问题.  
