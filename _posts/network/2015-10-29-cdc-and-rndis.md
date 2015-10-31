---
layout: post
title: "CDC-ECM 和 RNDIS"
description:
category: network
tags: [network, linux, wifi]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. USB-CDC

USB协议中的的CDC类是USB通信设备类 (Communication Device Class) 的简称, CDC类是USB组织定义的一类专门给各种通信设备（电信通信设备和中速网络通信设备）使用的USB子类, 而USB-cdc又可以分为几个子类

+ CDC-ACM : Abstract Control Model
   + ACM driver将usb设备视为一个虚拟的modem或者com口， driver能够通过ACM来发送data和AT command (使用不同的channel)， 或者通过模拟串口发送data和AT command
+ CDC-ECM : Ethernet Networking Control Model
   + ECM 协议能够在device和host之间交换ethernet frame， 符合ECM规范的设备， 认为自己是注解的一个虚拟的网络接口， 可以被分配MAC和IP, 通常使用ECM的设比是 LAN/WAN适配器
+ CDC-NCM : Network Control Model
   + NCM 协议用于在device和host之间， 交换高速的ethernet frame， 符合ECM规范的设备， 认为自己是注解的一个虚拟的网络接口， 可以被分配MAC和IP，常见的使用NCM的设备是 支持3.5G/4G网络(例如 HSPA+ 和 LTE)的无线适配器

每一个子类都由一个通信接口类和一个数据接口类组成

**定义了这样的一系列协议的好处是： usb 主/从 机都支持这些协议后， 就可以使用通用的设备驱动**

###2. CDC-ECM

以太网控制模型是用在主从设备间的以太网帧数据的交换, 其中， 

+ 通信类接口用于配置和管理以太网程序
+ 数据接口则用于在USB总线上交换USB数据包，这些USB数以的包封装了完整的以太网包。CRC校验和不能包含在以太网收发数据包中。检验失败的帧数据不能再发送到主机。这意味着设备必须能够缓冲至少一个完整的以太网帧数据

![cdc-ecm](/images/network/cdc-ecm.png)

###3. RNDIS

RNDIS (Remote network Driver Interface Specification) 协议是微软对于 CDC-ECM 的变种实现，主要用于简化windows平台中usb网络设备的驱动开发， RNDIS网络协议栈如图所示

RNDIS数据传输模型很复杂，每个USB消息都包含了多个以太网包, RNDIS默认期待自己作为USB配置中的唯一功能, **所以对于USB复合设备需要注意，RNDIS它期待自己是第一个usb配置**

Linux支持它仅仅是因为微软不支持 CDC以太网标准

**在Android手机中， RNDIS几乎是标配**

###4. linux使能CDC-ECM/RNDIS功能

若要在linux中支持RNDIS，需要修改内核的编译选项， 与以太网不同， 在usb网络中， 各端是不对等的，USB host 和 USB device端所需要的软硬件是不同的， 

####4.1 linux 作为USB host时

linux作为usb host时，要支持RNDIS，依赖 rndis_host.ko 模块， 需要打开kernel的config
 
	Device Drivers   --->
		Network Device Support --->
			Usb Network Adapters --->
				Multi-purpose USB Networking Framework --->
					CDC Ethernet Support
					Host For RDNIS and ActiveSync Devices

这两个config， 分别用于支持 CDC-ECM 和 RNDIS

####4.2 linux作为从机时

linux作为usb device时， 要支持RNDIS， 依赖 g_ether.ko 模块， 需要打开kernel的config

	Device Drivers   --->
		USB support --->
			USB Gadget Support --->
				USB Gadget Driver --->
					Ethernet Gadget (with CDC Ethernet Support)
					RNDIS support

这两个config， 分别用于支持 CDC-ECM 和 RNDIS

将运行linux的usb从机连接到运行linux的usb主机后，打开usb共享网络(即配置usb从机的功能为RNDIS), 则在主机和从机上都会出现一个名为 “usbx” (x为数字编号) 的网络接口

关于 usb gadget : 常用的usb设备， 比如 U盘， 鼠标， 键盘等等， 都比较简单， 使用较小的控制芯片，在其上面运行编译好的固件， 而一些linux嵌入式设备， 通常都配备有性能较好的cpu，并且， 很多这一类设备都支持OTG功能， 可以在同一接口上扮演usb host和usb device的功能， 对于将linux设备作为usb device端时， linux提出了Gadget驱动框架， 其定义了一套API， UDC(usb device controller)需要实现这些API (主流的SoC的usb controller driver基本上都实现了这些API， 利用这些API， Gadget 框架实现了一套与硬件无关的功能(基本上可以对应到 USB 协议里 的各种 USB Class), 当然，Gadget 驱动还是受限于底层提供的功能的。比如 某些 Class 需要 USB Isochronous  端点，这时我们就不能支持该 Class， 基于Gadget驱动框架， 可以开发driver，实现多种USB device， 比如

+ Gadget Zero, 类似于 dummy hcd, 该驱动用于测试 udc 驱动。它会帮助您通过 USB-IF 测试。
+ Ethernet over USB， 该驱动模拟以太网网口，它支持多种运行方式：
+ CDC Ethernet: usb 规范规定的 Communications Device Class “Ethernet Model” protocol。
+ CDC Subset： 对硬件要求最低的一种方式，主要是 Linux 主机支持该方式。
+ RNDIS： 微软公司对 CDC Ethernet 的变种实现。
+ File-backed Storage Gadget最常见的 U 盘功能实现。
+ Serial Gadget 实现，包括：
+ Generic Serial 实现（只需要Bulk-in/Bulk-out端点+ep0）
+ CDC ACM 规范实现。
+ Gadget Filesystem, 将 Gadget API 接口暴露给应用层，以便在应用层实现user mode driver。
+ MIDI: 暴露ALSA接口，提供 recording 以及 playback 功能。 

###5. USB共享网络模型

无论是使用 CDC-ECM 还是 RNDIS， 通过USB共享网络时， 模型如下：

![usb共享网络模型](/images/network/usb-network-share-model.png)

**在android设备上， 使用USB共享网络时， 虚拟的网络接口通常命名为rndis0， 而不是usb0**

在移动设备上， 应用RNDIS时， 移动设备通常作为 USB deivce， 其它的设备作为USB host， 虽然可以从任一端设备共享网络给另一端设备， 对于移动设备来说， 这两种方式有着不同的命名：

+ USB tethering ： 移动设备共享网络给其它的设备
+ USB reserse-tethering ： 其它的设备共享网络给移动设备

事实上，对于移动设备来说， 应用最多场景还是USB tethering(如果要进行reserse-tethring的话， 还需要在移动设备上关闭其它联网方式， 然后在另一端的设备上设置其它网卡的网络共享给其虚拟的usb网卡)

####5. usb网络共享与tcp/ip转发

**usb tethering 本质上是利用tcp/ip协议栈的转发功能， 和softap以及 bluetooth tethering 类似 (tcp/ip 网络协议栈的数据包转发来实现网络共享， 通常是使用iptables 命令来操作 其nat表来配置的)**

例如， 一台android手机， 通过wifi连接网络, 然后通过usb tethering 将网络共享给 ubuntu pc, 在android手机上连接wifi，在其开启usb tetehering之前，dump 其iptables的nat表的配置如下：

	# iptables -S -t nat
	-P PREROUTING ACCEPT
	-P INPUT ACCEPT
	-P OUTPUT ACCEPT
	-P POSTROUTING ACCEPT
	-N natctrl_nat_POSTROUTING
	-N oem_nat_pre
	-A PREROUTING -j oem_nat_pre
	-A POSTROUTING -j nactrl_nat_POSTROUTING

而在开启usb tethering之后， dump 其iptables的nat表的配置如下：

	# iptables -S -t nat
	-P PREROUTING ACCEPT
	-P INPUT ACCEPT
	-P OUTPUT ACCEPT
	-P POSTROUTING ACCEPT
	-N natctrl_nat_POSTROUTING
	-N oem_nat_pre
	-A PREROUTING -j oem_nat_pre
	-A POSTROUTING -j nactrl_nat_POSTROUTING
	-A natctrl_nat_POSTROUTING -o wlan0 -j MASQUERADE

正是最后多出的一行命令设置了将 wifi 接口的网络共享给其它的接口
	