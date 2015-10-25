---
layout: post
title: "7-在真机上引导linux"
description:
category: diy-linux
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 使用U盘在真机上引导内核  
  
我们已经能够从qemu中引导linux内核了，这一次， 我们使用一个u盘在一台真实的计算机上引导自己build的内核  
  
###2. 使用grub来引导  
  
####2.1 安装grub  
  
你需要一台安装了linux的机器， 大部分发行版都是使用grub来进行引导，里面都自带了grub的tool  
  
插上u盘， 确认是哪个块设备设备， 可以使用 `lsblk`命令来查看，我的pc上只有一块硬盘， 当我只插上一个U盘时， 它是 /dev/sdb  
  
格式化U盘， 建立ext4文件系统  
  
	sudo fdisk /dev/sda
    
删除原有的分区， 并建立一个主分区，那么现在就有“/dev/sdb”和“/dev/sdb1”（当然你的情况可能有所不同,一定要确认好， 否则可能破坏你的本地的引导） 然后格式化为ext4文件系统  
  
	sudo mkfs.ext4 /dev/sdb1
    
然后，挂载U盘  

	sudo mount -t ext4 /mnt /dev/sdb1
    
安装grub到U盘

	sudo grub-install --root-directory /mnt /dev/sdb
    
注意其中的 “--root-directory” 选项， 在安装grub的时候，需要拷贝grub的代码， 默认是到 “/boot/grub”， 但是我们需要安装到u盘上， 因此，使用“--root-directory”来指定根目录， grub-install会自动建立目录并拷贝所需的文件。  
  
“/dev/sdb”则指定写u盘的MBR  
  
grub的安装有多种方式，比如还可以进入grub的交互模式，使用install命令或者setup命令， 但是使用 grub-install 是最为简单的方式
  
####2.2 编辑grub配置文件  
  
在grub2中， /boot/grub/grub.cfg是唯一的配置文件， 当缺少此文件时， 开机时直接进入grub的交互模式。 我们需要编辑此文件，在里面添加引导选项  
  
	default=0
	timeout=10
	menuentry usb linux{
		linux /boot/bzImage ro ramdisk_size=65536 root=/dev/ram0 init=/bin/sh
		initrd /boot/initrd.img
	}
    
是的， 我们需要把build 出来的kernel 和 制作的initrd 拷贝到U盘的 /boot 目录， 然后插上upan到pc， 选择从u盘启动
  
###3. 使用syslinux来引导  
  
准备一个U盘， 假设为/dev/sdb  
  
	sudo mkdosfs -F 32 -I /dev/sdb
	sudo syslinux -s /dev/sdb
    
将bzImage和initrd.img拷贝到U盘根目录下， 并添加syslinux.cfg文件  
  
	TIMEOUT 20
	DEFAULT Linux
	LABEL Linux
	KERNEL bzImage
	APPEND root=/dev/ram0 ramdisk_size=30720 rw initrd=initrd.img init=/bin/sh
    
插上U盘， 开机并且选择从U盘启动。