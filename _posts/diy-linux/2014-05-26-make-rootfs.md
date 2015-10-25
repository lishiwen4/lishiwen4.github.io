---
layout: post
title: "4-建立根文件系统"
description:
category: diy-linux
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 建立根文件系统  

我们将使用ramdisk来作为根文件系统
  
###2. 建立ext4文件系统镜像  

	dd if=/dev/zero of=rootfs.img bs=1M count=30
	mkfs.ext4 initrd.img
    
###3. 拷贝busybox到根文件系统中
  
	sudo mount -o loop initrd.img rootfs
	cp busybox-1.20.0/_install/* rootfs -r
    
###4. 修改init程序配置  
  
linux在加载ramdisk中的根文件系统后， 先运行/linuxrc 程序，如果有真正的根文件系统的话， 加载它的存储设备的驱动，然后挂载正真的根文件系统， 执行/init程序。  

busybox中linuxrc其实是指向 /bin/busybox的链接， busybox会按照配置文件“/etc/inittable”来建立子进程， 如果找不到配置文件， 则会使用默认配置

建立根文件系统后， 你需要一个init程序，否则，内核启动后什么也不做就退出了， 你可以直接使用busybox里面的shell来作为init程序， 这样，开机就会打开一个shell用来交互