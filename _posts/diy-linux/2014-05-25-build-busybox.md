---
layout: post
title: "3-编译buxybox"
description:
category: diy-linux
tags: [linux]
mathjax: 
chart:
comments: false
---
  
###1. 下载busybox  
  
前往 http://www.busybox.net/ 下载，或者：  
  
	cd busybox-src
	wget http://www.busybox.net/downloads/busybox-1.20.0.tar.bz2  
	tar jxvf busybox-1.20.0.tar.bz2
    
###2. 编译busybox
  
使用默认配置来编译busybox
  
	cd busybox-src/busybox-1.20.0
	make defconfig
	make menuconfig
    
	busybox settings -->  
	build options -->  
		[]build busybox as a static binary （no shared libs）  

按空格选中， 配置完成后开始执行make
    
make 过程中可能会出如下错误   
  
	networking/lib.a(inetd.o): In function `unregister_rpc':
	inetd.c:(.text.unregister_rpc+0x20): undefined reference to `pmap_unset'
	networking/lib.a(inetd.o): In function `register_rpc':
	inetd.c:(.text.register_rpc+0x6a): undefined reference to `pmap_unset'
	inetd.c:(.text.register_rpc+0x86): undefined reference to `pmap_set'
	networking/lib.a(inetd.o): In function `prepare_socket_fd':
	inetd.c:(.text.prepare_socket_fd+0x7e): undefined reference to `bindresvport'
	collect2: error: ld returned 1 exit status
	make: *** [busybox_unstripped] Error 1
  
可以看出是build inetd demon进程时出现了问题， 最简单的解决方法是  
  
	make menuconfig
	
	Networking Utilities -->
		[]inetd
        
按空格去掉选中  
  
	make
		make install  
    
我在ubuntu12.10上build时没有遇到其他的问题， 但是在ubuntu14.04上有  
  
	loginutils/passwd.c: In function ‘passwd_main’:
	loginutils/passwd.c:104:16: error: storage size of ‘rlimit_fsize’ isn’t known
	loginutils/passwd.c:188:2: warning: implicit declaration of function ‘setrlimit’ [-Wimplicit-function-declaration]
	loginutils/passwd.c:188:12: error: ‘RLIMIT_FSIZE’ undeclared (first use in this function)
	loginutils/passwd.c:188:12: note: each undeclared identifier is reported>for each function it appears in
	loginutils/passwd.c:104:16: warning: unused variable ‘rlimit_fsize’ [-Wunused-variable]
    
解决的办法是:
  
	cd busybox-src/busybox-1.20.0/include
	vim libbb.h
  
添加一行 “#include < sys/resource.h >    
  
make install 后会生成 _install 目录， build出来的文件都在 _install 目录里面
