---
layout: post
title: "5-在qemu中引导linux"
description:
category: diy-linux
tags: [linux]
mathjax: 
chart:
comments: false
---

###1. 使用qemu引导linux
  
现在， 我们已经有了已经build好了的kernel， 和基于ramdisk的根文件系统(存储了busybox在其中)， 现在， 可以使用qemu引导一个使用内存盘的linux系统了  
  
###2. 方法一  
  
	cp linux-src/linux-linux-3.10.40/arch/x86_64/boot/bzImage .
	qemu-system-x86_64 -kernel bzImage -initrd initrd.img -append "root=/dev/ram0 rw ramdisk_size=30720 init=/bin/sh"
    
其中， 参数的意义如下

	-kernel 指定了内核镜象  
	-initrd 指定了启动内核时的initrd  
	-append 指定了linux内核的启动参数， 
		root		指定根文件系统所在的设备， 这里我们将ramdisk作为最终的根文件系统， 因此指定“/dev/ram0”
		rw			指定以rw的方式挂载根文件系统
		ramdisk_size	因为ramdisk的默认大小为4M， 而我们制作的根文件系统大小为30M，因此需要用“ramdisk_size”来指定ramdisk的大小， 
		init			指定init程序  
        
执行命令后， 会弹出一个新的窗口， kernel起来后， 你会获得一个交互式的终端， 当鼠标被该新窗口捕获后，可以使用 alt + ctrl 键来释放它。

或者你可以 运行qemu时，加上 -curses 参数，在当前终端里面以字符模式引导linux
  
是的， 这种方式是以ramdisk来存放根文件系统来引导内核的， 在qemu上， 还可以使用硬盘来存放根文件系统来引导kernel, 见方法二
  
###3. 方法二  
  
	qemu-system-x86_64 -kernel bzImage -hda rootfs.img -append "root=/dev/sda rw init=/bin/sh"

其中参数的意义

	-kernel	指定内核镜像
	-hda	指定第一块硬盘， 使用根文件系统镜像模拟硬盘  
	-append	指定linux内核的启动参数  
		root	指定根文件系统， 放置在第一块硬盘上  
		rw		以rw的方式挂载根文件系统  
		init	指定init程序
  
###4. 两种方法的差别  
  
这两种方式还是有很大的区别的， linux系统的常见的启动过程是这样的  
  
	1.kernel启动
	
	|
	
	2.挂载ramdisk作为根文件系统(临时的)
	
	|
        
	3.挂载外存上的根文件系统
    
方法一去掉了第3步， 将ramdisk上的文件系统作为最终的根文件系统， 在该文件系统上的读写操作才断电后是会丢失的  

方法二去掉了第2步， 直接挂载外存上的根文件系统， 在该文件系统上的读写操作才断电后是不会丢失的
