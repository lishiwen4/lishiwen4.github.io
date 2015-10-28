---
layout: post
title: "linux虚拟网络接口 —— 环回接口 lo"
description:
category: network
tags: [network, linux]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 虚拟网络接口

在linux的网络设备驱动框架里面， 使用一个net_device来代表一个网络设备接口， 因此， 一个物理网卡对应着一个net_device结构

虚拟网络接口是指一个网络接口的net_device没有直接对应的物理设备

###2. 环回接口 lo

linux上的回环网络接口也是一个虚拟网络接口， 所有通过该设备发送的数据都会被该接口接收返回

###3， 环回接口的实现

“lo” 回环网络接口的实现位于linux源码中的 “driver/net/loopback.c”， 其实现比较简单， 只有220行左右

	static netdev_tx_t loopback_xmit(struct sk_buff *skb,
                 struct net_device *dev)
	{
		......
    		skb->protocol = eth_type_trans(skb, dev);
		......
    		if (likely(netif_rx(skb) == NET_RX_SUCCESS)) {
			......
    		}
    		return NETDEV_TX_OK;
	}

	static const struct net_device_ops loopback_ops = { 
    		.ndo_init      = loopback_dev_init,
    		.ndo_start_xmit= loopback_xmit,
    		.ndo_get_stats64 = loopback_get_stats64,
	};

	static void loopback_setup(struct net_device *dev)
	{
		......
    		dev->netdev_ops     = &loopback_ops;
		......
	}

	static __net_init int loopback_net_init(struct net *net)
	{
    		struct net_device *dev;
    		......
    		dev = alloc_netdev(0, "lo", loopback_setup);
		......
	}


其中注册了一个名为“lo”的net_device， 且在其“.ndo_start_xmit”方法中会调用 netfi_rx()将要发送的数据原路返回到 L3， 由此可见，虽然通过“lo”发送的目的端地址是环回地址,可以省略部分传输层和所有网络层的逻辑操作， 但是linux中还是照样完成传输层和网络层的所有过程

**在linux的网络协议栈的L2中， 会处理网桥和VLAN， 但是对于127.x.x.x的地址则不会进行这些处理， 因此， 在设置桥接的时候，不要将地址设置为 127.xxx.xxx.xxx网段**

在不同的net namespace中， 都有一个独立的“lo”接口

###4. localhost/127.0.0.1/本机ip 的差别

根据惯例，大多数系统把IP地址127.0.0.1分配给环回接口， 并且将其命名为locahhost，在访问本机时， 可以使用localhost， 127.0.0.1， 以及本机网卡的ip， 那么， 这3者有何差别呢？

+ localhost

一个保留的DNS域名， 会被转换为127.0.0.1， 例如， 在linux上“/etc/hosts”中会定义
	
	127.0.0.1 localhost

+ 127.0.0.1

事实上 127.0.0.1 ～ 127.255.255.254 都是 loopback 地址

linux上会由系统维护一张 名为“local”的路由表， 并且优先级最高， 其中有

	local 127.0.0.0/8 dev lo  proto kernel  scope host  src 127.0.0.1
	local 127.0.0.1 dev lo  proto kernel  scope host  src 127.0.0.1

所有目的地址在 127.0.0.0/8 网段的数据包都通过“lo”网络接口收发

使用ping配合tcpdump抓包， 可得

	/* terminate 1 中执行 */
	$ tcpdump icmp -i lo

	/* terminate 2 中执行 */
	$ ping localhost -c 1

	/* terminate 1 中输出 */
	19:14:05.999400 IP localhost > localhost: ICMP echo request, id 8211, seq 1, length 64
	19:14:05.999412 IP localhost > localhost: ICMP echo reply, id 8211, seq 1, length 64


+ 本机某一网卡的ip

kernel的__ip_route_output_key()中， 若认为目标ip是本机ip后， 无论在路由过程中， 选定哪一个网络接口作为出口网络设备， 会强制将出口网络设备设置为 loopback_dev (即“lo”)

以本机网卡eth0 (ip10.42.0.1) 为例， 使用ping配合tcpdump抓包， 可得

	/* terminate 1 中执行 */
	$ tcpdump icmp -i lo

	/* terminate 2 中执行 */
	$ ping 10.42.0.1 -c 1

	/* terminate 1 中输出 */
	19:16:44.710902 IP 10.42.0.1 > 10.42.0.1: ICMP echo request, id 8284, seq 1, length 64
	19:16:44.710924 IP 10.42.0.1 > 10.42.0.1: ICMP echo reply, id 8284, seq 1, length 64
	
可见， 目的地址为本机ip的数据包， 都是通过lo口来收发(即使eth0没有接网线)

####3.1 考虑net namespace情况下的差异

namespace是linux上轻量级的虚拟化的容器技术， 用于提供独立的用户空间以及系统资源， 而其中的net name space则用于分离网络资源，一个net namespace有自己独立的路由表，iptables策略， 网络协议栈以及网络接口， 和其它的net namespace完全独立

因此， 若两个网络接口分配不同的ip， 并且加入不同的net namespace， 若从本机中的另一个net namespace中ping另外一个net namespace中的ip， 则数据包不会通过“lo”接口来loopback， 需要使用线缆来连接两个网卡， 因此， 若需要在一台机器上测试网卡的性能， 需要将其加入不同的net namespace
