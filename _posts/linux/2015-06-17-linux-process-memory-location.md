---
layout: post
title: "linux进程的内存布局"
description:
category: linux
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 32位linux下进程内存空间的经典布局


![32位linux下进程内存的经典布局](/images/linux/memory-location1.png)

**动态链接库也被映射到mmap区域**

这种布局 mmap 区域与栈区域相对增长，这意味着堆只有 1GB 的虚拟地址空间可以使用，继续增长就会进入 mmap 映射区域，这显然不是我们想要的。这是由于 32 模式地址空间限制造成的，所以 内核引入了前一种虚拟地址空间的布局形式。但是对 64 位模式，提供了巨大的虚拟地址空间，这个布局就相当好。如果要在 2.6.7 以后的内核上使用 32 位模式内存经典布局，可以使用如下任意命令来设置：

	$ sudo sysctl -w vm.legacy_va_layout=1
	$ ulimit -s unlimited 

###2. 32位linux下进程内存空间的默认布局

![32位linux下进程内存的默认布局](/images/linux/memory-location2.png)

**动态链接库也被映射到mmap区域**

从上图可以看到，栈至顶向下扩展，并且栈是有界的。堆至底向上扩展， mmap 映射区域至顶向下扩展， mmap 映射区域和堆相对扩展，直至耗尽虚拟地址空间中的剩余区域，这种结构便于 C 运行时库使用 mmap 映射区域和堆进行内存分配。上图的布局形式是在内核 2.6.7 以后才引入的

可以看到，栈和 mmap 映射区域并不是从一个固定地址开始，并且每次的值都不一样，这是程序在启动时随机改变这些值的设置，使得使用缓冲区溢出进行攻击更加困难。当然也可以让程序的栈和 mmap 映射区域从一个固定位置开始，只需要设置全局变量 randomize_v a_space 值为 0 ，这个变量默认值为 1 。用户可以通过如下的任一方式来停用进程地址随机化
	
	$ echo 0 >  /proc/sys/kernel/randomize_va_space
	$ sudo sysctl -w kernel.randomize_va_space=0

该属性有3个取值：

+ 0 ： 表示关闭内存地址随机化 
+ 1 ： 表示将mmap的基址，stack和vdso地址随机化
+ 2 ： 表示在1的基础上， heap地址随机化

###3. 64位linux上内存布局

![64位linux下进程内存布局](/images/linux/memory-location3.png)

64位系统的寻址空间比较大，所以仍然沿用了32位的经典布局，但是加上了随机的mmap起始地址，以防止溢出攻击

**目前大部分的操作系统和应用程序并不需要16EB( 264 )如此巨大的地址空间, 实现64位长的地址只会增加系统的复杂度和地址转换的成本, 带不来任何好处. 所以目前的x86-64架构CPU都遵循AMD的Canonical form, 即只有虚拟地址的最低48位才会在地址转换时被使用, 且任何虚拟地址的48位至63位必须与47位一致(sign extension). 也就是说, 总的虚拟地址空间为256TB**

256TB的虚拟内存空间划分如下:

1. 0000000000000000 - 00007fffffffffff(128TB)为用户空间
2. ffff800000000000 - ffffffffffffffff(128TB)为内核空间

内核空间中有很多空洞, 越过第一个空洞后, ffff880000000000 - ffffc7ffffffffff(64TB)才是直接映射物理内存的区域, 也就是说默认的PAGE_OFFSET为ffff880000000000

这么大的直接映射区域足够映射所有的物理内存, 所以目前x86-64架构下是不存在高端内存, 也就是ZONE_HIGHMEM这个区域的

###4. text section的起始地址

上面说到过， 在intel/amd上, text section的起始地址为：

+ x86上为 0x8048000
+ x86_64上为 0x400000

而在其它的平台上又是多少呢，对于一个确定的平台，这一值一般是固定， 可在编译器的链接脚本中看到， linux桌面上，链接脚本都放置在“/usr/lib/ldscripts”目录下，例如“/usr/lib/ldscripts/elf_x86_64.x”即为64位应用程序的链接脚本, 而对于嵌入式平台，交叉编译器自己都附带了链接脚本， 例如aarch64的交叉编译器“aarch64-linux-android-4.9”附带的链接脚本存放在“arch64-linux-android/lib/ldscripts”目录中， 从其中的链接脚本“aarch64elf.x”中可以看到， 定义的text section 存放的起始地址为

	PROVIDE (__executable_start = 0x00400000); . = 0x00400000;

###5. 查看进程的内存空间布局

查看进程的内存空间布局， 可以使用如下cmd(xxxx代表进程的pid)

	$ cat /proc/xxxx/maps

或者使用如下cmd可以查看更详细的内容

	$ cat /proc/xxxx/smaps

例如

	$  cat /proc/2390/maps
	00400000-0040b000 r-xp 00000000 08:01 9961491                            /bin/cat
	0060a000-0060b000 r--p 0000a000 08:01 9961491                            /bin/cat
	0060b000-0060c000 rw-p 0000b000 08:01 9961491                            /bin/cat
	009bb000-009dc000 rw-p 00000000 00:00 0                                  [heap]
	7fd81b83c000-7fd81bf20000 r--p 00000000 08:01 5249470                    /usr/lib/locale/locale-archive
	7fd81bf20000-7fd81c0d5000 r-xp 00000000 08:01 262155                     /lib/x86_64-linux-gnu/libc-2.15.so
	7fd81c0d5000-7fd81c2d4000 ---p 001b5000 08:01 262155                     /lib/x86_64-linux-gnu/libc-2.15.so
	7fd81c2d4000-7fd81c2d8000 r--p 001b4000 08:01 262155                     /lib/x86_64-linux-gnu/libc-2.15.so
	7fd81c2d8000-7fd81c2da000 rw-p 001b8000 08:01 262155                     /lib/x86_64-linux-gnu/libc-2.15.so
	7fd81c2da000-7fd81c2df000 rw-p 00000000 00:00 0 
	7fd81c2df000-7fd81c301000 r-xp 00000000 08:01 262267                     /lib/x86_64-linux-gnu/ld-2.15.so
	7fd81c4e3000-7fd81c4e6000 rw-p 00000000 00:00 0 
	7fd81c4ff000-7fd81c501000 rw-p 00000000 00:00 0 
	7fd81c501000-7fd81c502000 r--p 00022000 08:01 262267                     /lib/x86_64-linux-gnu/ld-2.15.so
	7fd81c502000-7fd81c504000 rw-p 00023000 08:01 262267                     /lib/x86_64-linux-gnu/ld-2.15.so
	7fff5776d000-7fff5778e000 rw-p 00000000 00:00 0                          [stack]
	7fff577fe000-7fff57800000 r-xp 00000000 00:00 0                          [vdso]
	ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

另外， 可以使用 “/proc/xxxx/mem”来dump进程的内存， 例如， 以上面的进程2390为例， dump其vdso

	$ sudo dd if=/proc/2390/mem of=~/vdso.elf bs=$[0x1000] skip=$[7fff577fe] count=2

###END