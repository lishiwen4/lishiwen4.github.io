---
layout: post
title: "linux虚拟网络接口 —— tun/tap"
description:
category: network
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux 虚拟网络接口

在linux的网络设备驱动框架里面， 使用一个net_device来代表一个网络设备接口， 因此， 一个物理网卡对应着一个net_device结构

虚拟网络接口是指一个网络接口的net_device没有直接对应的物理设备

###2. TUN/TAP

TUN/TAP 会生成虚拟网络接口， 配合“/dev/net/tun”设备文件， 所有通过 tun/tap 网络接口发送的数据包， 都会进入“/dev/net/tun”设备文件的缓冲区， 用户空间可以从“/dev/net/tun”设备文件中读出读出这些数据包， 而写入“/dev/net/tun”设备文件的数据包， 则会进入到linux的网络协议栈进行处理

tun/tap的区别如下：

+ tun表示虚拟的是点对点设备，工作在三层网络上，具有处理IP包的能力
+ tap表示虚拟的是以太网设备，工作在二层网络上，具有直接处理以太帧的能力

tun/tap的内核代码位于 “driver/net/tun.c”

###3. /dev/net/tun

"/dev/net/tun"是一个misc 设备， minor id固定为200

	crw-rw-rwT  1 root root 10, 200  7月 31 09:11 tun

	static const struct file_operations tun_fops = {
		......
    		.aio_read  = tun_chr_aio_read,
    		.aio_write = tun_chr_aio_write,
    		.unlocked_ioctl = tun_chr_ioctl,
    		.open   = tun_chr_open,
		.release = tun_chr_close,
		......
	};

	 struct miscdevice tun_miscdev = {
    		.minor = TUN_MINOR,
    		.name = "tun",
    		.nodename = "net/tun",
    		.fops = &tun_fops,
	};

对于“/dev/net/tun”设备来说， 最重要的操作就是 read / write / ioctl了， 其中ioctl用于添加/删除 tun/tap 接口

###4. 添加/删除 tun/tap 网络接口

tun/tap 的添加通过使用ioctl 发送 TUNSETIFF 命令来实现

例如，添加一个tun网络接口的示例

	int tun_alloc(char *dev)  
	{  
  		struct ifreq ifr;  
  		int fd, err;  
  
  		if ((fd = open("/dev/net/tun", O_RDWR)) < 0)  
			......
  
  		memset(&ifr, 0, sizeof(ifr));  
  		ifr.ifr_flags = IFF_TUN | IFF_NO_PI;  
  
  		if (*dev)  
    			strncpy(ifr.ifr_name, dev, IFNAMSIZ);   
  
  		if ((err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0)  
  			......

  		return fd;  
	}

ioctl的执行流程

	/* userspace */
	ioctl( TUNSETIFF )

	/* kernel */
	tun_chr_ioctl()
		__tun_chr_ioctl()
			tun_set_iff()
				tun_net_init()

	static void tun_net_init(struct net_device *dev)
	{
		......
		switch (tun->flags & TUN_TYPE_MASK) {
			case TUN_TUN_DEV:
				dev->netdev_ops = &tun_netdev_ops;
				.......
				break;

			case TUN_TAP_DEV:
				dev->netdev_ops = &tap_netdev_ops;
				break;
			......
		}		
	}

	static const struct net_device_ops tun_netdev_ops = {
		.ndo_uninit		= tun_net_uninit,
		.ndo_open		= tun_net_open,
		.ndo_stop		= tun_net_close,
		.ndo_start_xmit		= tun_net_xmit,
		.ndo_change_mtu		= tun_net_change_mtu,
		.ndo_fix_features	= tun_net_fix_features,
		.ndo_select_queue	= tun_select_queue,
	#ifdef CONFIG_NET_POLL_CONTROLLER
		.ndo_poll_controller	= tun_poll_controller,
	#endif
	};

	static const struct net_device_ops tap_netdev_ops = {
		.ndo_uninit		= tun_net_uninit,
		.ndo_open		= tun_net_open,
		.ndo_stop		= tun_net_close,
		.ndo_start_xmit		= tun_net_xmit,
		.ndo_change_mtu		= tun_net_change_mtu,
		.ndo_fix_features	= tun_net_fix_features,
		.ndo_set_rx_mode	= tun_net_mclist,
		.ndo_set_mac_address	= eth_mac_addr,
		.ndo_validate_addr	= eth_validate_addr,
		.ndo_select_queue	= tun_select_queue,
	#ifdef CONFIG_NET_POLL_CONTROLLER
		.ndo_poll_controller	= tun_poll_controller,
	#endif
	};

根据 ioctl 的 TUNSETIFF cmd 附带的IFF_TUN 或者 IFF_TAP flag来添加一个 tun或者tap网络接口， TUNSETIFF 支持如下的flag

1. IFF_TUN		： 创建一个点对点设备， 和 IFF_TAP互斥
2. IFF_TAP		： 创建一个以太网设备， 和 IFF_TUN互斥
3. IFF_NO_PI		： 不包含包信息，默认的每个数据包当传到用户空间时，都将包含一个附加的包头来保存包信息， 可和其它flag组合
4. IFF_ONE_QUEUE	： 采用单一队列模式，即当数据包队列满的时候，由虚拟网络设备自已丢弃以后的数据包直到数据包队列再有空闲， 可和其它flag组合

对于上述的 IFF_NO_PI, 当其没有开启时， 附加的包头信息为：

	struct tun_pi {
		unsigned short flags;
		unsigned short proto;
	};

添加的tun/tap网络接口按照 tun0， tun1 ... 或者 tap0 tap1 ...的形式命名, 同其它的网络接口一样， 可以使用ifconfig  ip等命令来配置ip地址， 路由等

tun/tap 网络接口的删除则是通过释放打开 “/dev/net/tun” 设备文件时获取的fd来实现的， 调用流程为

	/* userspace */
	close()

	/* kernel */
	tun_chr_close()
		tun_detach()
			__tun_detach()
				
在__tun_detach()中会 unregister tun/tap网络接口对应的net_device

还可以使用tunctl命令管理tun/tap网络接口

	/* 创建名为tap0的虚拟网卡， 拥有者为sven */
	$ tunctl -t tap0 -u sven
	
	/* 删除名为tap0 的虚拟网卡 */
	$ tunctl -d tap0

	/* tun/tap 虚拟网卡同其它网卡一样， 使用ifconfig进行设置, 例如 */
	$ ifocnfig tap0 192.168.1.165 netmask 255.255.255.0 up

###5. tun/tap 网络接口上的数据发送

数据包经过路由后， 若选择 tun/tap 网络接口作为出口， 最终需要调用  net_device.ndo_start_xmit() 来发送数据， 即调用tun_net_xmit(), 在其中会

+ 将skb加入到readq链表中， 然后唤醒等待的read进程,("/dev/net/tun" 的read进程会在tun_do_read()中阻塞， 等待被数据唤醒)
+ "/dev/net/tun" 的read进程在tun_do_read()中被唤醒后， 从skb 数据包队列中， 取出数据， 调用tun_put_user()将数据拷贝到用户空间buffer (对“/dev/net/tun” 调用read()时指定的buffer)

###6. tun/tap 网络接口上的数据接收

向 “/dev/net/tun”写入数据包

	/* userspace */
	write()
	
	/* kernel */
	tun_chr_aio_write()
		tun_get_user()
			tun_alloc_skb()
			skb_copy_datagram_from_iovec()
			netif_rx_ni()

可见， tun_get_user()中， 分配skb结构，从用户空间拷贝数据，最后调用netif_rx_ni()， 将数据包发往上层的网络协议栈处理， 最后， 应用程序在用户空间通过socket接口接收数据

###7. tun/tap 网络接口的应用

从上面的tun/tap网络接口的数据包收发流程可以看到：

+ 数据包写入“/dev/net/tun”设备文件后， 到达linux网络协议栈， 最后通过socket接口被应用程序接收， 这一过程中， 可以逐层剔除各层的协议头， 得到应用层数据
+ 应用层数据通过socket接口进入网络协议栈， 经路由后， 选择tun/tap接口作为出口设备， 到达“/dev/net/tun”设备文件的缓冲区， 可在用户空间读出， 得到以太帧或者ip数据包， 这一过程中，逐层添加协议头

上面的两个相反的过程， 可以应用于实现网络隧道， vtun和openvpn等项目都使用了 tun/tap, 另外， 一些虚拟机的虚拟化网络也使用到了 tun/tap 技术
