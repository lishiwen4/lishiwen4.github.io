---
layout: post
title: "linux 内存文件系统"
description:
category: linux
tags: [linux, build-link]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 内存文件系统
   
大部分的文件系统，建立在可持久存储的设备上， 如磁盘， U盘等， 而内存文件系统则是将一块内存空间作为存储介质， 构建在其上的文件系统， linux中最常见的两种内存文件系统是ramfs和tempfs。  
  
###2. ramfs
  
ramfs是一种简单的文件系统，它直接利用了linux的高速缓存机制（因此代码很小， 直接集成在内核代码中，不能被配置），使用系统的物理内存做成一个基于内存的文件系统。  
  
ramfs工作于虚拟文件系统层（VFS）,不能被格式化， 可以创建多个。  
  
ramfs和其它的文件系统最大的差别就是， 它没有真正的对应的文件系统设备（如磁盘，U盘等），为ramfs分配的高速也缓存和目录缓存不能被标记为clean的状态， 系统永远也不会释放ramfs所占用的高速缓存，故只有root用户有权限操作ramfs.  
  
ramfs只能在物理内存中创建(虚拟内存包括物理内存和交换空间，ramfs不能被交换到磁盘上).
  
###3. tempfs 
  
tempfs在ramfs的基础上增加了回写设备，增大了容量大小限制，允许向交换分区写入数据，因此， 普通用户也可以使用tempfs。  
  
tempfs是一种虚拟内存文件系统， 它不同于用块设备实现的ramdisk， 也不同于针对于物理内存的ramfs，它直接向虚拟内存子系统来请求页来存储文件， 自身并不清楚请求的页存在于内存上还是交换分区上。  
因此， tempfs是一种建立在虚拟内存上的文件系统。  
    
tempfs和ramfs一样， 不能被格式化， 大小固定。  

###4. rootfs  
  
  rootfs是一个ramfs或者tempfs的实例，大部分文件系统安装在rootfs之上，然后忽略它， 它是linux启动的初始化根文件系统。