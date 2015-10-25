---
layout: post
title: "1-qemu简介"
description:
category: diy-linux
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. qemu简介  
  
QEMU是一套由Fabrice Bellard所编写的以GPL许可证分发源码的模拟处理器，在GNU/Linux平台上使用广泛。Bochs，PearPC等与其类似，但不具备其许多特性，比如高速度及跨平台的特性，通过KQEMU这个闭源的加速器，QEMU能模拟至接近真实电脑的速度，目前，kqemu加速器可用于0.9.1版本或者之前版本的qemu，在qemu1.0之后的版本，都无法使用kqemu，主要利用qemu-kvm加速模块，并且加速效果以及稳定性明显好于kqemu。  
  
###2. qemu的运行模式  
  
+ user mode模拟模式 : QEMU能启动那些为不同中央处理器编译的Linux程序，类似于Wine及Dosemu。  
+ system mode模拟模式 : QEMU能模拟整个电脑系统，包括中央处理器及其他周边设备。它使得为跨平台编写的程序进行测试及除错工作变得容易。其亦能用来在一部主机上虚拟数部不同虚拟电脑。。  
  
###3. qemu能够模拟的系统  

在linux下输入 `qemu-system`再按tab键补全，能够看到qemu能够模拟的平台，如：  

	qemu-system-arm
	qemu-system-mips64      
	qemu-system-sh4
	qemu-system-cris        
	qemu-system-mips64el    
	qemu-system-sh4eb
	qemu-system-i386        
	qemu-system-mipsel      
	qemu-system-sparc
	qemu-system-m68k        
	qemu-system-ppc         
	qemu-system-sparc64
	qemu-system-microblaze  
	qemu-system-ppc64       
	qemu-system-x86_64
	qemu-system-mips        
	qemu-system-ppcemb  

使用`qemu-system-xxx -M ?`能看到该平台下能够模拟的机器， 例如， `qemu-system-arm -M ？`的输出如下：  
  
	Supported machines are:
	none                 empty machine
	beagle               Beagle board 	(OMAP3530)
	beaglexm             Beagle board XM (OMAP3630)
	collie               Collie PDA (SA-1110)
	nuri                 Samsung NURI board (Exynos4210)
	smdkc210             Samsung SMDKC210 board (Exynos4210)
	connex               Gumstix Connex (PXA255)
	verdex               Gumstix Verdex (PXA270)
	highbank             Calxeda Highbank (ECX-1000)
	integratorcp         ARM Integrator/CP (ARM926EJ-S) (default)
	kzm                  ARM KZM Emulation Baseboard (ARM1136)
	mainstone            Mainstone II (PXA27x)
	musicpal             Marvell 88w8618 / MusicPal (ARM926EJ-S)
	n800                 Nokia N800 tablet aka. RX-34 (OMAP2420)
	n810                 Nokia N810 tablet aka. RX-44 (OMAP2420)
	n900                 Nokia N900 (OMAP3)
	sx1                  Siemens SX1 (OMAP310) V2	
	sx1-v1               Siemens SX1 (OMAP310) V1
	overo                Gumstix Overo board (OMAP3530)
	cheetah              Palm Tungsten|E aka. Cheetah PDA (OMAP310)
	realview-eb          ARM RealView Emulation Baseboard (ARM926EJ-S)
	realview-eb-mpcore   ARM RealView Emulation Baseboard (ARM11MPCore)
	realview-pb-a8       ARM RealView Platform Baseboard for Cortex-A8
	realview-pbx-a9      ARM RealView Platform Baseboard Explore for 		Cortex-A9
	akita                Akita PDA (PXA270)
	spitz                Spitz PDA (PXA270)
	borzoi               Borzoi PDA (PXA270)
	terrier              Terrier PDA (PXA270)
	lm3s811evb           Stellaris LM3S811EVB
	lm3s6965evb          Stellaris LM3S6965EVB
	tosa                 Tosa PDA (PXA255)
	versatilepb          ARM Versatile/PB (ARM926EJ-S)
	versatileab          ARM Versatile/AB (ARM926EJ-S)
	vexpress-a9          ARM Versatile Express for Cortex-A9
	vexpress-a15         ARM Versatile Express for Cortex-A15
	xilinx-zynq-a9       Xilinx Zynq Platform Baseboard for Cortex-A9
	z2                   Zipit Z2 (PXA27x)

###4. ubuntu下安装qemu  
  
如果只是使用system mode的话， 使用 `sudo apt-get install qemu-system`来安装即可  
 
qemu的一般选项如下

**标准选项**
  
	-fda/-fdb	指定floppy 0/1 的镜像文件
	-hda/-hdb	指定IDE disk 0/1 的镜像文件
	-hdc/-hdd	指定IDE disk 2/3 的镜像文件
	-cdrom		指定IDE cdrom（ide1 master）的镜像文件
	-boot		启动选项，软盘（a）、硬盘（c）、光驱（D）、网卡（n），默认是从硬盘启动
	-m		指定内存大小（MB）默认为384
	-smp		cpu个数
	-full-screen	全屏
		
**linux启动选项**
  	
	-kernel		指定kernelimg
	-append		指定linux的启动参数， 使用“”括起来
	-initrd		指定init ram disk
	-dtb		指定设备树镜像文件
