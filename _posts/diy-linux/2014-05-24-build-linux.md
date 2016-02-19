---
layout: post
title: "2-编译linux内核"
description:
category: diy-linux
tags: [linux]
mathjax: 
chart:
comments: false
---

###1. 工作区目录结构
  
在开始我们的整个工程的第一步， 我希望能使用一个整洁的目录结构来保存每一步尝试的结果  
  
	mkdir diy-linux
	cd diy-linux
	mkdir linux-src
	mkdir busybox-src
	mkdir rootfs
    
执行完之后， 看起来目录结构是这样的  
  
	diy-linux --|
		   |- linux-src
		   |- busybox-src
		   |- rootfs
                
+ linux-src : 用于保存 linux kernel 源码  
+ busybox-src : 用于保存 busybox 源码  
+ rootfs : 用于构建根文件系统的挂载点  

后续还会添加一些目录和文件用于备份
  
###2. 下载linux内核  
  
访问 www.kernel.org 下载所需要的版本，也可以直接使用wget来下载， 如 3.10.40
  
	cd linux-src
	wget --no-check-certificate http://www.kernel.org/pub/linux/kernel/v3.0/linux-3.10.40.tar.xz  
	tar Jxvf linux-3.10.40.tar.xz
    
###3. 配置linux内核  
  
在内核树的根目录中，有一个.config文件，它记录了内核的配置选项，可直接对它进行修改，再运行。在.config文件中，每个配置和选项的值只能为”y”和”m”两者之一，如果不需要这个特性不再支持她，那么可以将对应的选项用”#”注释掉。

	在ubuntu 的 /boot/config-'uname -r`中保存了系统中对应版本内核的配置 
  
在内核源码的目录中， 执行 make xxxconfig 来进行内核配置， 常用的配置方式如下：  
  
+ make config  : 基于文本的最为传统也最为枯燥的配置方式， 但是适用于任何方式， 为每一个内核特性向用户提问，回答“y”，或者“m”，或者“n”，每一次运行都会对所有问题进行提问，不管你之前有没有进行过配置。 
+ make oldconfig : 类似于“make config”， 但是在现有的.config文件的基础上建立一个配置文件，只会就当前.config文件中没有配置的内核特性向用户提问。在配置好并保存时， 会将之前的.config文件文件备份到.config.old中去, make silentoldconfig 和上面oldconfig一样，但在屏幕上不再出现已在.config中配置好的选项。    
+ make menuconfig : 基于终端的一种配置方式，提供了文本模式的图形用户界面，用户可以通过光标移动来浏览所支持的各种特性。使用这用配置方式时，系统中必须安装有ncurese库，否则会显示“Unable to find the Ncurses libraies”的错误提示 
+ make xoncifg : 基于X Winodws的一种配置方式，提供了漂亮的配置窗口，不过只有能够在X Server上使用root用户欲行X应用程序时，才能够使用，它依赖于QT，如果系统中没有安装QT库，则会出现“Unable to find the QT installation”的错误提示 
+ make gconfig : 与make xocnifg类似，不同的是make gconfig依赖于GTK库    
+ make defconfig : 按照默认的配置文件arch/i386/defconfig对内核进行配置，生成.config可以用作初始化配置，然后再使用make menuconfig进行定制化配置 
+ make allyesconfig : 尽量多地使用“y”设置内核选项值，生成的配置中包含了全部的内核特性 
+ make allmodconfig : 尽可能多的使用“m”设置内核选项值来生成配置文件 
+ make allnoconfig : 除必须的选项外,其它选项一律不选. (常用于嵌入式系统).
+ make localmodconfig : make localmodconfig 会执行 lsmod 命令查看当前系统中加载了哪些模块 (Modules)， 并最后将原来的 .config 中不需要的模块去掉，仅保留前面 lsmod 出来的这些模块，从而简化了内核的配置过程，单是该方法仅能使编译出的内核支持当前内核已经加载的模块， 如果你要为特定的机器来配置一个最简的系统， 这个命令很有用 
+ make localyesconfig : 同“make localmodconfig”一样， 但是会把选中的模块直接build进内核， 而不再是以模块的方式存在 

其实"make menuconfig"，"make xconfig" "make gconfig" 的功能相当于"make oldconfig"，只是以字符界面或者不同的图形界面来和用户交互 
  
我们要做的是  
  
	cd linux-src/linux-3.10.40
	make defconfig
	make menuconfig

这一步可能会出现如下的错误：

    scripts/kconfig/lxdialog/dialog.h:31:20: fatal error: curses.h: No such file or directory

这是因为缺少 libncurses， 执行如下命令来安装

    $ sudo apt-get install libncurses

###4. 打开Ramdisk选项  
  
使用defconfig的话，基本上无需做改动，  但是你还是需要确保以下几条选项被打开以支持ramdisk  

	General setup -->
		Initial Ram filesystem and Ram disk support
    
	Device Drivers -->
	Block devices -->
	Ram block device support  
            
你必须选择将它们 build-in kernel 而不是编译成模块
  
###5. 编译linux
  
直接执行 make -j4 来进行编译  
  
几个常用的编译命令： 
 
1. make dep  	当你更改了某一个模块的配置后， 另外的模块可能受收到影响，需要rebuild，运行make dep来解决依赖（2.6后不需要此命令了）
2. make modules	编译模块
3. make bzimage	编译内核压缩文件
  
执行`make -j4`, 编译完成后， 在内核源码下的 arch/x86_64/boot 下面(我安装的是64位版ubuntu)，会生成 bzmage 文件
