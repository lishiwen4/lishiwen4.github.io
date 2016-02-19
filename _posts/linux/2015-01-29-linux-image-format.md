---
layout: post
title: "linux 内核镜像文件种类"
description:
category: linux
tags: [linux, build-link]
mathjax: 
chart:
comments: false
---

###1. kernel 镜像文件种类  
  
linux kernerl镜像文件按照格式可以分为如下几种：

+ vmlinux
+ zImage
+ bzImage
+ uImage
+ vmlinuxz  
  
###2. vmlinux：  
  
编译生成的最原始的内核文件， 没有经过压缩。  
  
###3. zImage：  
  
vmlinux经过gzip压缩后的文件， 包含自解压代码。  
  
###4. bzImage：  
  
bz表示“big zImage”，不是用bzip2压缩的。两者的不同之处在于，zImage解压缩内核到低端内存(第一个640K)，bzImage解压缩内核到高端内存(1M以上)。如果内核比较小，那么采用zImage或bzImage都行，如果比较大应该用bzImage。  
  
###5. uImage：  
  
U-boot专用的映像文件，它是在zImage之前加上一个长度为0x40的tag(64个字节，说明这个映像文件的类型、加载位置、生成时间、大小等信息)。其实就是一个自动跟手动的区别,有了uImage头部的描述,u-boot就知道对应Image的信息,如果没有头部则需要自己手动去搞那些参数。换句话说，如果直接从uImage的0x40位置开始执行，zImage和uImage没有任何区别。  
  
###6. vmlinuxz：  
  
vmlinuxz是bzImage/zImage文件的拷贝或指向bzImage/zImage的链接
