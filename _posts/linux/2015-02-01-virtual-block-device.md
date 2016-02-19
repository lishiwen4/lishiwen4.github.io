---
layout: post
title: "linux 虚拟块设备"
description:
category: linux
tags: [linux, build-link]
mathjax: 
chart:
comments: false
---

###1. 虚拟块设备
  
虚拟块设备是指不存在真实的物理块设备， 而是在内存上模拟（ramdisk）或者使用文件模拟（loop device）的块设备。  
  
###1. ramdisk  

ramdisk是一种内存中的一块区域当作物理磁盘来使用的技术，它在内存中创建一个快设备， 用来存放文件系统，掉电后文件内容会消失， 不能用于长期保存文件爱你的介质。为了使用ramdisk必须在编译内核时配置选项“CONFIG_BLK_DEV_RAM”。为了在内核的加载阶段就使用ramdisk， 还必须配置"CONFIG_BLK_DEV_INITRD".  

ramdisk的大小固定， 安装在上面的文件系统的大小也是固定的。ramdisk在使用的时候，和高速缓存之间有数据拷贝，还需要文件系统的驱动来格式化和解时这些数据，使用ramdisk浪费了内存，污染了cache， 加重了cpu的负担，而且所有的文件访问都需要通过页和目录缓存来进行，这些工作都是ramfs需要执行的，另外， 回环设备提供了一个根为灵活和方便的方式（通过文件而不是大块内存）来创建一个块设备。因此， linux2.6以后已经废弃了ramdisk.  可以使用 loop device
  
**ramdisk与ramfs的区别: ramdisk有一块内存作为后端， 因此可以 挂载/回写/格式化**

###2. loop device    
  
loop device是linux上的一种特殊的块设备，它通过映射操作系统上正常的文件来虚拟你个块设备。这种机制为我们创建一个特定文件系统的镜像提供了条件。  
   
创建文件系统镜像的步骤：  
   
1.使用`dd`命令创建一个指定大小的空白文件  
  `dd if=/dev/zero of=img_file bs=1k count=10000`  
  
2.使用·`losetup`命令关联循环设备和img_file文件  
  `losetup /dev/loop0 img_file`   
  
3.使用`mkfs`命令创建一个文件系统  
  `mkfs.ext4 /dev/loop0`  
  
4.使用mount命令挂载文件系统  
  `mount /dev/loop0 /mnt`  
  
5.向挂载的文件系统中添加所需的文件  
  
6.使用`umount`命令卸载文件系统  
  `umount /mnt`  
  
7.使用`losetup`命令解除镜像文件和循环设备的关联  
  `losetup -d /dev/loop0`  
  
现在， 得到了一个包含指定文件的文件系统的镜像  
  
