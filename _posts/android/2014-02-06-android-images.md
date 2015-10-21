---
layout: post
title: "android image 文件"
description:
category: android
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. boot.img  & recovery.img  

boot.img 和 recovery.img 并不是一个完整的文件系统的镜像，而是android自定义的一种文件格式，从头至尾依次为： 
 
1. 1page大小的文件头  
2. gzip压缩过的kernel.img(zImage，由vmlinux和解压程序组成)  
3. ramdisk内存盘（android采用的根文件系统）  
4. boot阶段stage 2的程序（可选的）  
  
合并起来的boot.img会放到一个单独的分区中.  
  
也就是说：  

	boot.img = zImage + ramdisk.img  
	recovery.img = zImage + recover-ramdisk.img(比正常的ramdisk.img多一些额外的资源)  
  
boot.img recovery.img的生成, mkbootimg工具(out/host/linux-x86/bin/mkbootimg， 源码位于 system/core/mkbootimg)
例如:

	$ mkbootimg --cmdline 'no_console_suspend=1 console=null' --kernel zImage --ramdisk boot/boot.img-ramdisk.gz -o boot.img --base 02e00000
  
官方源码并未附带 boot.img 和 recovery.img 的unpac工具， 但是存在许多的第三方工具。  
  
###2. ramdisk.img  
  
ramdisk.img 是一个cpio格式的压缩文件，有多unpack种方式， 如：  

	$ gunzip -c ../ramdisk.img | cpio -i

ramdisk.img的pack方式（out/host/linux-x86/bin/mkbootfs 和 out/host/linux-x86/bin/minigzip）  

	/out/host/linux-x86/bin/mkbootfs root/ | ./out/host/linux-x86/bin/minigzip > ramdisk.img
  

###3. system.img文件  

在android2.3之后， 默认的文件系统是ext4，之前版本为yaffs2。虽然说默认的文件系统为ext4， 但是， 没有规范强制要求一定是ext4， 设备厂商也可以选择其它的文件系统，但是大部分还是使用默认的ext4. 
  
虽然说linux原生支持ext4文件系统， 但是，编译android源码生成的img文件经过了转换， 已经不能被正确识别为ext4文件系统了，因此， 必须按照 unpack-modify-pack 的动作来进行修改：  
  
	//unpack (out/host/linux-x86-/bin/Simg2img)  
	# simg2img system.img system.img.raw
  
	//mount  
	# mount -t ext4 system.img.raw /mnt
  
	//modify  
    
	//umount  
	# umount /mnt
  
	// pack (out/host/linux-x86/bin/make_ext4fs)  
	# make_ext4fs -s -l 512M -a system system.img system.img.raw

make_ext4fs的参数为：

+  -s : 表示除去分区中的空数据，即生成的img为实际数据的大小， 而不是分区的大小  
+  -l : 指定分区的大小  
+  -a : 指定该分区在android上的挂载点