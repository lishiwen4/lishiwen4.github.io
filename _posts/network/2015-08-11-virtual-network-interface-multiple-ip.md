---
layout: post
title: "linux虚拟网络接口 —— 多ip地址"
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

###2. 网卡绑定多个ip

每一个物理网卡， 都有一个唯一的MAC地址， 这一点很好理解， 每一个网卡都有一个ip地址，这一点看起来也很自然，但是， 如果说一个网卡可以绑定多个ip地址， 很多人都无法理解，但是， 如果说“ip地址是主机的， 而不是网卡的，相信就比较好理解了”， 按照网络协议的分层， 下层为上层提供服务， 网络层基于ip寻址， 但是在为传输层提供服务时，tcp/udp可以使用不同的端口号， 同样的， 数据链路层基于MAC地址寻址， 为网络层服务时， 网络层可以使用不同的ip， 只要arp协议能够正确地对应MAC和ip
 
一张网卡上绑定多个ip有两种实现： 早期的 ip alias和现在的secondary ip

**网卡绑定多地址不会注册新的net_device, 因此， 严格意义来说， 并不算是虚拟网卡，但是早期的ip alias方式添加的多个ip， 列出的 ethx:y  看起来像是生成了新的网络接口一样**

###3. ip alias

ip alias在物理网卡名字后面加上后缀从而成为虚拟网络接口，例如 eth0:0 eth1:1，但是它并没有生成新的net_device， 因此， 不要去内核代码中寻找它的net_device注册过程

linux最多支持255个ip alias(注意， 是一个网卡最多255个ip alias， 不是说后缀数字最大255个)

ip alias使用ifconfig 命令来设置， ifconfig 通过ioctl来与kernel通信

以eth0接口为例

	eth0      Link encap:Ethernet  HWaddr 60:a4:4c:21:e7:4d  
		inet addr:10.42.0.1  Bcast:10.42.0.255  Mask:255.255.255.0
		inet6 addr: fe80::62a4:4cff:fe21:e74d/64 Scope:Link
		UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
		RX packets:5532032 errors:0 dropped:0 overruns:0 frame:0
		TX packets:5108136 errors:0 dropped:0 overruns:0 carrier:0
		collisions:0 txqueuelen:1000 
		RX bytes:1177246076 (1.1 GB)  TX bytes:1679653488 (1.6 GB)
		Interrupt:20 Memory:f7f00000-f7f20000 

使用 ifconfig 绑定一个ip(添加一个虚拟网卡 eth0:0 )

	$ ifconfig eth0：0 10.42.0.2 netmask 255.255.255.0 up

使用 ifconfig 查看， 可以发现新增的网络接口 eth0：0

	eth0:0    Link encap:Ethernet  HWaddr 60:a4:4c:21:e7:4d  
		inet addr:10.42.0.2  Bcast:10.42.0.255  Mask:255.255.255.0
		UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
		Interrupt:20 Memory:f7f00000-f7f20000 

若要删除绑定的ip， 可以使用 

	$ ifconfig eth0:0 down

使用ifconfig查看ip alias可以看到不同网络接口的MAC地址都是相同的

###4. secondary ip

在linux上， 只要一张网卡上配置的ip不是同一个网段的， 那么这些ip就都是primary ip， 若有相同网段的ip， 则其中第一个ip作为primary ip，其它的ip就是secondary ip，当一个primary地址被删除时，如果它有secondary地址，那么它的第一个secondary地址继承被删除的primary地址的位置成为primary地址

可以通过 “ip addr” 命令来添加 primary/secondary ip， “ip” command通过netlink与kernel通信

以eth0为例

	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    		link/ether 60:a4:4c:21:e7:4d brd ff:ff:ff:ff:ff:ff
    		inet 10.42.0.1/24 brd 10.42.0.255 scope global eth0
			valid_lft forever preferred_lft forever
		inet6 fe80::62a4:4cff:fe21:e74d/64 scope link 
       			valid_lft forever preferred_lft forever

	$ ip addr add dev eth0 10.42.0.165/24
	$ ip addr add dev eth0 10.42.1.1/24
	$ ip addr show
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    		link/ether 60:a4:4c:21:e7:4d brd ff:ff:ff:ff:ff:ff
    		inet 10.42.0.1/24 brd 10.42.0.255 scope global eth0
       			valid_lft forever preferred_lft forever
    		inet 10.42.1.1/24 scope global eth0
       			valid_lft forever preferred_lft forever
    		inet 10.42.0.165/24 scope global secondary eth0
       			valid_lft forever preferred_lft forever
    		inet6 fe80::62a4:4cff:fe21:e74d/64 scope link 
       			valid_lft forever preferred_lft forever

可以看到eth0原本的ip 10.42.0.1/24 和 后添加的  10.42.1.1/24 不在同一网段， 因此都是primary ip， 而后添加的 10.42.0.165/24 则和 原本的ip 10.42.0.1/24 是同一个网段， 因此是secondary ip

“使用ip 命令添加的ip地址， 无法通过ifconfig查看， 但是使用ifconfig添加的ip则可以使用ip命令查看”

删除添加的primary secondary ip使用“ip addr”， 例如

	$ sudo ip addr del dev eth0 10.42.1.1/24

例如当同一台机器上的不同服务需要监听同一个端口时， 可以分配不同的secondary ip，外部通过不同的ip来访问这些服务， 看起就相当于一台机器具有多张网卡 

###5. kernel中多ip的实现

kernel中， 使用net_device来表示一个网络接口， 使用 struct in_device 和 struct in_ifaddr 来存储其ip地址

	struct net_device
	{
		...
     		void                    *ip_ptr;       //指向一个in_device结构
		...
	} 

	struct in_device {
		struct net_device	*dev;
		......
		struct in_ifaddr	*ifa_list;	/* IP ifaddr chain		*/
		......
	};

	struct in_ifaddr {
		......
		struct in_ifaddr	*ifa_next;	/* 指向下一个in_ifaddr，形成链表*/
		struct in_device	*ifa_dev;		/* 指向struct in_device */
		......
		__be32			ifa_local;		/* ip地址 */
		__be32			ifa_address;
		__be32			ifa_mask;			/* 子网掩码 */
		__be32			ifa_broadcast;		/* 广播地址 */
		unsigned char		ifa_scope;
		unsigned char		ifa_flags;		/* 若是secondary ip， 则为IFA_F_SECONDARY */
		unsigned char		ifa_prefixlen;		/* */
		char			ifa_label[IFNAMSIZ];	/* 在ip alias中， 存储 ethx:y 形式字符， 在secondary ip中， 固定为ethx */
		......
	};

net_device.ip_ptr->ifa_list 链表保存了该 net_device 的 ip

ifconfig 只会返回该链表中的第一个ip， 而ip 命令则会遍历该链表， 返回所有的ip， 因此两个命令看到的结果不同相同

###6. 主机多ip时源ip的选择

当一个主机创建IP 数据包时，必须选择正确的源IP地址，这是至关重要的，因为只有源地址正确，才能让接收者正确响应。如果源地址错误，则无法得到对端主机的任何回应

如果一个主机绑定有多个IP地址， 当客户机向主机发送数据包后， 主机在回应时， 数据包的ip地址源地址为客户机请求的地址， 但是如果主机主动向外发送数据包时， 源地址如何选择呢？

选择源地址使用如下的步骤：

1. 应用程序可以通过并setsockopt() 传递 IP_PKTINFO ，从而显式指定源ip地址(只对数据报套接字有效)，在这种情况下，操作系统内核仅仅检查其源ip地址是否正确，否则产生相应的错误
2. 如果应用程序没有指定源ip地址，包含源ip的路由表项可以指定数据包源ip地址，通过设置 “ip route” 命令的 src 参数，从而指定源ip地址
3. 其它情况下， 内核选择数据包路由的网络接口上的ip地址， 若该网络接口上绑定了多个ip地址， 则尽量选择与目标ip处于同一子网的ip作为源ip， 当然， 只会使用primary ip，不考虑secondary ip

**最后，可以指定iptable规则在不同的过滤点修改源ip**
