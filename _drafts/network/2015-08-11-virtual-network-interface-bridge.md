---
layout: post
title: "linux虚拟网络接口 —— bridge"
description:
category: wifi
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux 虚拟网络接口

在linux的网络设备驱动框架里面， 使用一个net_device来代表一个网络设备接口， 因此， 一个物理网卡对应着一个net_device结构

虚拟网络接口是指一个网络接口的net_device没有直接对应的物理设备

###2. bridge

Bridge 设备是基于内核软件实现的二层数据交换设备，其作用类似于现实世界中的二级交换机

Linux内核通过一个虚拟的网桥设备来实现桥接的，这个设备可以绑定若干个以太网接口设备，从而将它们桥接起来， 假设网桥设备br0绑定了eth0和eth1。对于网络协议栈的上层来说，只看得到br0，因为桥接是在数据链路层实现的，上层不需要关心桥接的细节。于是协议栈上层需要发送的报文被送到br0，网桥设备的处理代码再来判断报文该被转发到eth0或是eth1，或者两者皆是；反过来，从eth0或从eth1接收到的报文被提交给网桥的处理代码，在这里会判断报文该转发、丢弃、或提交到协议栈上层，当然， 有时候eth0、eth1也可能会作为报文的源地址或目的地址，直接参与报文的发送与接收（从而绕过网桥）

**注意一个网络接口只能被一个网桥绑定, 并且网桥不能被其它的网桥绑定， 但是网桥可以绑定其它类型的虚拟网络接口**

###3. bridge相关数据结构

net_device 中添加了一个 net_bridge的结构， 其中， 几个重要的成员如下：

+ net_bridge.port_list : 保存bridge绑定的网络接口(例如 eth0， eth1......)
+ net_bridge.hash		: 用来处理地址学习的, hash表中的net_bridge_fdb_entry， 可以通过MAC地址索引到一个网络接口

###3. bridge网络接口的添加

bridge的添加可以使用brctl命令 (包含在bridge-utils软件包中), 例如

	$ brctl addbr br0

brctl通过ioctl发送 SIOCBRADDBR cmd来完成bridge的添加， 其执行过程为

	/* userspace */
	ioctl( SIOCBRADDBR )

	/* kernel */
	sock_ioctl()
		br_ioctl_hook()	/* bridge 初始化时， 注册br_ioctl_hook为br_ioctl_deviceless_stub() */
			br_add_bridge()


	int br_add_bridge(struct net *net, const char *name)
	{
		struct net_device *dev;
		int res;

		dev = alloc_netdev(sizeof(struct net_bridge), name, br_dev_setup);

		if (!dev)
			return -ENOMEM;

		......
		res = register_netdev(dev);
		if (res)
			free_netdev(dev);
		return res;
	}

每次添加一个bridge， 就会注册一个对应的net_device结构体, 且net_device.priv_flags会设置IFF_EBRIDGE标记， 注册的net_device的 net_device.netdev_ops为

	static const struct net_device_ops br_netdev_ops = {
		.ndo_open		 = br_dev_open,
		.ndo_stop		 = br_dev_stop,
		.ndo_init		 = br_dev_init,
		.ndo_start_xmit		 = br_dev_xmit,
		......
	};

**一个bridge网络接口有自己的MAC， 可以为其设置ip，并且可以使用该接口收发数据, 通常来说ip地址是三层协议的内容，不应该出现在二层设备bridge 上。但是Linux里bridge 是通用网络设备抽象的一种，只要是网络设备就能够设定ip地址**

**linux中可以配置多个bridge， 不同的bridge之间的通信需要依赖L3**

###4. bridge网络接口的删除

bridge的删除可以使用brctl命令, 例如

	$ brctl delbr br0

brctl通过ioctl发送 SIOCBRDELBR cmd来完成bridge的添加， 其执行过程为

	/* userspace */
	ioctl( SIOCBRDELBR )

	/* kernel */
	sock_ioctl()
		br_ioctl_hook()	/* bridge 初始化时， 注册br_ioctl_hook为br_ioctl_deviceless_stub() */
			br_del_bridge()
				br_dev_delete()

+ 在br_del_bridge() 会检查该网络接口是否是bridge (net_device.priv_flags是否有IFF_EBRIDGE标记)，并且该网络接口是否是up状态
+ br_dev_delete() 中解除该bridge上绑定的网络接口， 然后unregister其对应的netdevice

###5. bridge绑定网络接口

一个交换机至少要2个以上的端口才有意义， 因此， 单独添加一个bridge接口并不起作用， 还需要给它绑定网络接口，添加网络接口由可以使用brctl命令来完成

	$ brctl addif br0 eth0

brctl 通过ioctl发送 SIOCBRADDIF 来绑定一个指定的网络接口到指定的bridge， 其处理过程为

	/* userspace */
	ioctl( SIOCBRADDIF )

	/* kernel */
	br_dev_ioctl()
		add_del_if()
			br_add_if()

**需要说明的是， net_device可在net_device.priv_flags中设置IFF_DONT_BRIDGE标记， 表示不希望被绑定到网桥上，br_add_if()中会忽略对该类网络接口的绑定**

net_device 中添加了一个 net_bridge的结构， 其中， 几个重要的成员如下：

+ net_bridge.port_list	: 保存bridge绑定的网络接口(例如 eth0， eth1......)
+ net_bridge.hash		: 用来处理地址学习的, hash表中的net_bridge_fdb_entry， 可以通过MAC地址索引到一个网络接口

br_add_if()中会

1. 将要绑定的网络接口对应的net_device添加到bridge对应的net_bridge.port_list中去
2. 将要绑定的网络接口的net_device.priv_flags添加IFF_BRIDGE_PORT标记
3. 设置要绑定的网络接口的 net_device.rx_handler为br_handle_frame()

**需要注意的是， 当一个设备被绑定到到bridge上时，那个设备的ip会变的无效，Linux不再使用那个ip在三层接受数据，无法ping通该地址， 因此需要为bridge设置ip， 使用bridge的ip来通信**

###6. bridge解除绑定网络接口

解除绑定网络接口由可以使用brctl命令来完成

	$ brctl delif br0 eth0

brctl 通过ioctl发送 SIOCBRDELIF 来解除绑定一个指定的网络接口到指定的bridge， 其处理过程为

	/* userspace */
	ioctl( SIOCBRDELIF )

	/* kernel */
	br_dev_ioctl()
		add_del_if()
			br_del_if()

br_del_if()中所做的动作与br_del_if()相反

1. 将要绑定的网络接口对应的net_device从bridge对应的net_bridge.port_list中删除
2. 将要绑定的网络接口的net_device.priv_flags取消IFF_BRIDGE_PORT标记
3. 设置要绑定的网络接口的 net_device.rx_handler为NULL

###7. bridge的数据发送流程

同其它的网络接口一样， birdge可以使用ifconfig和ip命令配置ip地址和路由

若路由的结果选择以网桥作为数据包的发送设备， 则进行如下处理

bridge工作在L2， linux中L3的数据通过dev_queue_xmit()发送给L2的发送队列来处理

	dev_queue_xmit()
		dev_hard_start_xmit()
			br_dev_xmit()


	netdev_tx_t br_dev_xmit(struct sk_buff *skb, struct net_device *dev)
	{
		......
		if (is_broadcast_ether_addr(dest))
			br_flood_deliver(br, skb);
		else if (is_multicast_ether_addr(dest)) {
			if (unlikely(netpoll_tx_running(dev))) {
				br_flood_deliver(br, skb);
				goto out;
			}
			if (br_multicast_rcv(br, NULL, skb)) {
				kfree_skb(skb);
				goto out;
			}

			mdst = br_mdb_get(br, skb, vid);
			if (mdst || BR_INPUT_SKB_CB_MROUTERS_ONLY(skb))
				br_multicast_deliver(mdst, skb);
			else
				br_flood_deliver(br, skb);
		} else if ((dst = __br_fdb_get(br, dest, vid)) != NULL)
			br_deliver(dst->dst, skb);
		else
			br_flood_deliver(br, skb);
		......
	}

+ 对于广播数据包， 发送到网桥上绑定的每一个网络接口上
+ 如果是多播数据包
   1. 如果能够查找到多播地址对应的网络接口， 则发送到这些网络接口上
   2. 如果不能够查找到多播地址对应的网络接口， 则发送到网桥上绑定的每一个网络接口上
+ 对于单播数据包
   1. 若能找到目的MAC地址对应的网络接口， 则发送到该网络接口上
   2. 若不能找到目的MAC地址对应的网络接口，则发送到网桥上绑定的每一个网络接口上

**可以直接通过bridge绑定的网络接口发送数据， 这种情况下， 发送的数据不会经过brdge处理**

###8. bridge的数据接收流程

首先， 数据包由网桥绑定的物理网卡接收到， 在其接收处理例程中， 最终会调用netif_rx()将数据包放入接收队列中， 而netif_receive_skb()则会从接收队列中取出数据包， 进行L2的处理

	netif_receive_skb()
		__netif_receive_skb_core()
			rx_handler()

在绑定一个网络接口到bridge时， 该网络接口对应的 net_device.rx_handlser 会被设置为 br_handle_frame()

+ 若目的MAC为链路本地多播地址， 则进行netfilter的NF_BR_LOCAL_IN过滤， 若未被过滤掉， 则继续处理
+ 若网桥处于forwarding模式(正常工作模式)， 若通过ebtabl配置规则需要转发，则将其传递到L3， 否则， 同下一步处理
+ 若网桥处于leasrning模式， 且数据包的目的mac地址为虚拟网桥设备的mac地址，则先进行 netfilter的NF_BR_PRE_ROUTING过滤， 再由br_handle_frame_finish处理
+ br_handle_frame_finish()中对于单播或者多播的数据， 发送到对应的网络接口上， 或者广播到所有绑定的网络接口上
+ br_handle_frame_finish()中对于本地网卡的数据包， 进行netfilter的NF_BR_LOCAL_IN过滤之后， 然后调用netif_receive_skb()将数据包传递给上层处理

网桥的几种工作状态：

	#define BR_STATE_DISABLED 0
	#define BR_STATE_LISTENING 1
	#define BR_STATE_LEARNING 2
	#define BR_STATE_FORWARDING 3
	#define BR_STATE_BLOCKING 4

+ disable ： 禁用状态， 不参与生成树,不转发任何数据帧
+ listening ： 监听状态,能够决定根,可以选择根端口、指定端口和非指定端口O在监昕状态的过程中,端口不能学 习任何接收帧的单播地址
+ learning ： 学习状态,端口能学习流入帧的MAC地址，不能转发帧
+ forwarding ： 正常工作， 转发状态,接口能够转发帧。端口学习到接收帧的源MAC地址，并可根据目标MAC地址进行恰当地转发， 不参与帧转发,监听流人的BPDU,不能学习接收帧的任何MAC地址
+ blocking ： 不参与数据包转发

###9. bridge 生成树协议

关闭生成树协议

	$ brctl stp br0 off

http://blog.csdn.net/kaoa000/article/details/8991716

http://blog.chinaunix.net/uid-21275705-id-224340.html

###未完