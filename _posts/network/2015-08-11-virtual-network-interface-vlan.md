---
layout: post
title: "linux虚拟网络接口 —— 802.1q vlan"
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

###2. 802.1q vlan

VLAN（Virtual Local Area Network）的中文名为"虚拟局域网"

在计算机网络中，一个二层网络可以被划分为多个不同的广播域，一个广播域对应了一个特定的用户组，默认情况下这些不同的广播域是相互隔离的。不同的广播域之间想要通信，需要通过一个或多个路由器。这样的一个广播域就称为VLAN

在一台物理的交换机上， 划分不同的VLAN相当于将物理交换机分割成多个逻辑上独立的交换机， 只有处于同一个VLAN内的端口才能互相通信

1. VLAN可分离广播域， 缩小arp中间人攻击，减少广播风暴
2. VLAN只是分离广播域， 并不能隔离广播域， 不同的vlan可通过3层通信

###3. vlan的划分

vlan的划分可以分为两种情况：

+ 静态vlan划分
+ 动态vlan划分

####3.1 静态vlan划分

静态VLAN又被称为基于端口的VLAN， 需要明确指定各端口属于哪个VLAN， 由于需要一个个端口地指定，因此当网络中的计算机数目超过一定数字(比如数百台)后，设定操作就会变得烦杂无比。并且，客户机每次变更所连端口，都必须同时更改该端口所属VLAN的设定——这显然不适合那些需要频繁改变拓补结构的网络

####3.2 动态vlan划分

动态VLAN则是根据每个端口所连的计算机，随时改变端口所属的VLAN。这就可以避免上述的更改设定之类的操作。动态VLAN可以大致分为3类

1. 基于MAC地址的VLAN
2. 基于子网的VLAN
3. 基于用户的VLAN

**基于MAC地址的VLAN**

基于MAC地址的VLAN，就是通过查询并记录端口所连计算机上网卡的MAC地址来决定端口的所属。假定有一个MAC地址“A”被交换机设定为属于VLAN “10”，那么不论MAC地址为“A”的这台计算机连在交换机哪个端口，该端口都会被划分到VLAN 10中去。计算机连在端口1时，端口1属于VLAN 10;而计算机连在端口2时，则是端口2属于VLAN 10， 因此基于MAC地址的VLAN是一种在二层动态设定vlan的方法

**基于子网的VLAN**

通过所连计算机的IP地址，来决定端口所属VLAN的。不像基于MAC地址的VLAN，即使计算机因为交换了网卡或是其他原因导致MAC地址改变，只要它的IP地址不变，就仍可以加入原先设定的VLAN， 因此基于子网的VLAN是一种在三层动态设定vlan的方法

**基于用户的VLAN**

根据交换机各端口所连的计算机上当前登录的用户，来决定该端口属于哪个VLAN。这里的用户识别信息，一般是计算机操作系统登录的用户，比如可以是Windows域中使用的用户名。这些用户名信息，属于OSI第四层以上的信息， 因此基于用户的VLAN是一种在四层以上动态设定vlan的方法

决定端口所属VLAN时利用的信息在OSI中的层面越高，就越适于构建灵活多变的网络

###4. 交换机的级联

交换机的端口是有限的， 当需要将较多的网络设备划分到同一个vlan中时， 使用一个交换机是不够的， 需要多台交换机的级联， 如下图所示

![vlan与交换机的级联](images/network/vlan-exp.png)

两台交换机上划分了两个vlan ：

+ 端口1，2，5，8，10为一个vlan， 其中端口5， 8用于两台交换机的级联
+ 端口3，4，6，7，9为一个vlan， 其中端口6， 7用于两台交换机的级联

###5. trunk端口与vlan tag

从上图交换机的级联可以看出， 交换机上每一个vlan都要使用一个单独的端口来用作级联， 若一个交换机上有多个vlan， 级联时必将浪费大量的端口， 为此， 提出了 trunk端口和vlan tag的概念

+ vlan tag : 每一个vlan被分配一个ID， 在必要时将该TAG添加到一台网帧中， 以标识该以太网帧所属的vlan
+ trunk 端口 : 多个vlan的数据可以通过此类端口， 进入和离开该类端口的数据需打上vlan tag
+ Hybrid 端口 : 和Trunk端口在接收数据时是一样的，唯一不同之处在于发送数据时：Hybrid端口可以允许多个VLAN的报文发送时不打标签，而Trunk端口只允许缺省VLAN的报文发送时不打标签
交换机上的不同vlan共用同一个端口用于级联，
+ access 端口 : 该类端口只属于某一个vlan， 进入和离开该类端口的数据不需要打vlan tag

关于缺省端口， access 端口的缺省vlan ID为其指定的vlan id， 而trunk端口和Hybird端口的缺省vlan id为vlan1

**比较常用的是trunk端口和access端口， 以下图为例**

![trunk端口与交换机级联](images/network/vlan-exp1.png)

两台交换机上共划分了2个vlan， 共使用两个端口进行级联

+ 端口 1， 2， 10都是access端口， 并且属于同一个vlan
+ 端口 3， 4， 9都是access端口， 并且属于同一个vlan
+ 端口 5. 8 属于trunk端口

####6. vlan的tag与untag

vlan tag信息添加在以太网帧的“源MAC”和“长度/类型”字段之间， 一共4个字节, 依次为：

+ TPID ： 2字节， 固定为0x8100， 标识报文的类型为802.1q封装
+ tag control info ： 2字节， 细分为
   1. 802.1p优先级， 3bit， 值为0～7， 主要用于当交换机出端口发生拥塞时,交换机通过识别该优先级,优先发送优先级高的数据包
   2. CFI， 1bit， 以太网交换机中，规范格式指示器总被设置为0。由于兼容特性，CFI常用于以太网类网络和令牌环类网络之间，如果在以太网端口接收的帧具有CFI，那么设置为1，表示该帧不进行转发，这是因为以太网端口是一个无标签端口
   3. VLAN ID， 对VLAN的识别字段，在标准802.1q中常被使用。该字段为12bit。支持4096（2^12）VLAN的识别。在4096可能的VID中，VID＝0用于识别帧优先级，4095（FFF）作为预留值，所以VLAN配置的最大可能值为4094

**谨记一点；只有在需要vlan tag时， 才会打上tag， 否则， 能不打tag就不打tag**

仍旧以上面的图为例

**计算机A向计算机B发送数据**

1. 计算机A通过arp， 查询得到计算机B的MAC地址
2. 数据由计算机A进入端口1
3. 交换即查询MAC地址和端口的对应关系， 得到目的端口为端口2
4. 数据从端口2发出， 进入计算机B

**计算机A向计算机F发送数据**

1. 计算机A通过arp， 查询得到计算机F的MAC地址
2. 数据由计算机A进入端口1
3. 交换机1查询MAC地址和端口的对应关系， 得到目的端口为端口5
4. 进入端口2， 由于该端口巍峨日trunk端口， 因此， 数据包打上蓝色vlan的tag
5. 数据包从端口5发出， 进入端口8
7. 交换机2查询MAC地址和端口的对应关系， 得到目的端口为端口10
8. 交换机将数据发送到端口10
9. 数据进入端口10后， 去掉vlan tag， 然后从端口10发出， 进入计算机F

不同vlan之间的通信， 需要借助L3路由，下一节会详细介绍

###7. vlan之间通信

链路层的通信， 必须在数据帧头中指定通信目标的MAC地址， 而为了获取MAC地址，TCP/IP协议下使用的是ARP， ARP解析MAC地址的方法，则是通过广播， 但是vlan的目的，就是为了分离广播域，这样一个vlan的arp广播， 无法到达另外一个vlan, 也就无从解析MAC地址，亦即无法直接通信,因此，不同vlan之间的通信，需要借助L3路由 (记住arp广播是L2数据， 无法借助L3路由) 

交换机与路由的连接也有两种方式， 如下图所示

![vlan之间的通信](images/network/vlan-route.png)

无论是那一种方式， 都需要路由器支持vlan

**一般的， 同一vlan指定同一个网段的ip， 不同vlan指定不同的ip， 因为如果同一vlan内指定不同网段的ip，则他们之间的通信还需要通过路由， 若不同的vlan指定相同的网段， 则它们之间不会经过路由， 但是又存在vlan的隔离， 无法通信**

###7.1 交换机通过access端口连接路由器

如上图左的方式， 交换机上划分了两个vlan， 且路由器上也划分了两个vlan， 分别使用access端口连接， 将路由器设定为gateway

以计算机A发送数据到计算机C为例

1. 计算机A和计算机C不在同一网段， 因此计算机A需要将数据发送到网关(即路由器)
2. 计算机A通过ARP协议查找到路由器的MAC
3. 计算机A发出的数据包进入端口1
4. 交换机查找MAC和端口的对应关系， 将数据包转发到端口5
5. 数据包流出端口5， 进入路由器的端口
6. 路由器查询内部的路由表项，修改数据包的目标MAC为计算机C的MAC， 由于目标ip属于白色vlan， 因此从负责白色vlan的接口中发出
7. 数据包进入端口6， 交换机查找MAC和端口， 将数据包转发到端口3
8. 数据包从端口3进入计算机C
 
这一方式因占用较多的端口， 因此应用不多， 实际上交换机和路由都是使用trunk端口互联

####7.2 交换机通过trunk端口连接路由器

如上图右的方式， 交换机上划分了两个vlan， 且路由器上也划分了两个vlan， 分别使用trunk端口连接， 将路由器设定为gateway

以计算机A发送数据到计算机C为例

1. 计算机A和计算机C不在同一网段， 因此计算机A需要将数据发送到网关(即路由器)
2. 计算机A通过ARP协议查找到路由器的MAC
3. 计算机A发出的数据包进入端口1
4. 交换机查找MAC和端口的对应关系， 将数据包转发到端口5
5. 交换机的端口5属于trunk端口， 因此进入的数据包打上蓝色vlan tag
6. 数据包流出端口5， 进入路由器的trunk端口
7. 路由器的trunk端口接收到数据包后， 检查到蓝色的vlan tag， 交由负责蓝色的端口处理(事实上， 路由器端会生成对应的vlan的端口)
8. 路由器查询内部的路由表项，修改数据包的目标MAC为计算机C的MAC
9. 路由器内交换端口查询MAC和端口对应关系， 选定trunk口
10. 由于目标ip属于白色vlan， 因此数据包进入trunk端口后打上白色vlan tag
11. 数据包进入交换机的trunk端口后， 查询交换机内的MAC和端口的对应关系后， 讲数据包转发到端口3上
12. 数据包进入端口3后， 去掉vlan tag， 流出端口3， 进入计算机C

###8. 3层交换机

交换机使用被称为ASIC(ApplicationSpecified Integrated Circuit)的专用硬件芯片处理数据帧的交换操作，在很多机型上都能实现以缆线速度(Wired Speed)交换。而路由器，则基本上是基于软件处理的。即使以缆线速度接收到数据包，也无法在不限速的条件下转发出去，因此会成为速度瓶颈。就VLAN间路由而言，流量会集中到路由器和交换机互联的汇聚链路部分，这一部分尤其特别容易成为速度瓶颈。并且从硬件上看，由于需要分别设置路由器和交换机，在一些空间狭小的环境里可能连设置的场所都成问题

在这一场景下， 3层交换机应运而生

三层交换机，本质上就是“带有路由功能的交换机”，路由属于OSI参照模型中第三层网络层的功能，因此带有第三层路由功能的交换机才被称为“三层交换机”，其内部分别设置了交换机模块和路由器模块， 而内置的路由模块与交换模块相同，使用ASIC硬件处理路由。因此，与传统的路由器相比，可以实现高速路由。并且，路由与交换模块是汇聚链接的，由于是内部连接，可以确保相当大的带宽

三层交换机与路由器的差别：

+ 三层交换机采用硬件实现转发， 速度较快， 非常适用于数据交换频繁的局域网中， 而路由器的转发较三层交换机慢， 更适合于数据交换不是很频繁的不同类型网络的互联
+ 三层交换机虽然具有路由功能， 但是3层功能较为简单， 不如路由器的功能强大， 路由器的转发功能基于软件实现， 较为灵活
+ 三层交换机一般只支持tcp/ip网络， 对于IPX AppleTalk等网络协议， 很多三层交换机还不支持， 需要使用路由器

###9. linux上的vlan

在物理设备上， 都是先有lan， 然后才划分vlan， 而在linux上则刚好相反， 因为构建lan的所需的bridge是一个逻辑设备， 因此， 在linux上需要先构建一个vlan设备， 然后将其绑定到一个bridge上去， 所有绑定到该bridge上的设备构成一个vlan， 而构建该vlan设备时指定的网络设备则作为trunk端口

例如

	/* 构建vlan   vlan 10  */
	ifconfig eth0 up
	vconfig eth0 10
	ifconfig eth0.10 up

	/* 创建网桥 */
	brctl addbr brvlan10

	/* 添加 vlan10 到网桥中 */
	brctl addif brvlan10 eth0.10

	/* 添加其它网络接口到网桥中 */
	ifconfig eth1 0.0.0.0 up
	brctl addif brvlan10 eth1

	ifconfig eth2 0.0.0.0 up
	brctl addif brvlan10 eth2

执行完成之后， eth0， eth1， eth2 构成了一个vlan， 其中eth0 为trunk端口， eth1和eth2为access端口， vlan的id为10

**上面的示例中， 创建vlan时使用eth0作为宿主设备， 事实上， 还可以使用虚拟网络设备作为vlan的宿主设备**

###10. linux上vlan的实现

linux上， vlan部分的代码位于“kernel/net/8021q”目录中

####10.1 vlan初始化

	void vlan_ioctl_set(int (*hook) (struct net *, void __user *))
	{
		......
		vlan_ioctl_hook = hook;
		......
	}

	static int __init vlan_proto_init(void)
	{
		......
		vlan_ioctl_set(vlan_ioctl_handler);
		......
	}
	module_init(vlan_proto_init);

在vlan部分的代码初始化时， 会将一个函数指针 vlan_ioctl_hook 指向函数 vlan_ioctl_handler()

sock_ioctl在将处理vlan相关的ioctl时， 会调用vlan_ioctl_hook

	static long sock_ioctl(struct file *file, unsigned cmd, unsigned long arg)
	{
		......
		case SIOCGIFVLAN:
		case SIOCSIFVLAN:
			......
			if (vlan_ioctl_hook)
				err = vlan_ioctl_hook(net, argp);
			......
			break;
			......
	}

####10.2 添加vlan

添加vlan通过发送ioctl(SIOCSIFVLAN)， 且sub cmd为 ADD_VLAN_CMD

	/* userspace */
	ioctl(SIOCSIFVLAN)

	/**/
	sock_ioctl()
		vlan_ioctl_handler()
			register_vlan_device()



	static const struct net_device_ops vlan_netdev_ops = {
		......
		.ndo_start_xmit =  vlan_dev_hard_start_xmit,
		......
	}

	void vlan_setup(struct net_device *dev)
	{
		......
		dev->netdev_ops		= &vlan_netdev_ops;
		......
	}


	static int register_vlan_device(struct net_device *real_dev, u16 vlan_id)
	{
		......
		new_dev = alloc_netdev(sizeof(struct vlan_dev_priv), name, vlan_setup);
		......
		vlan_dev_priv(new_dev)->real_dev = real_dev;
		......
		err = register_vlan_dev(new_dev);
		......
	}

在添加vlan时， 构建一个net_device设备， 设置其发送函数为 vlan_dev_hard_start_xmit()

####10.3 删除vlan

添加vlan通过发送ioctl(SIOCSIFVLAN)， 且sub cmd为DEL_VLAN_CMD

	/* userspace */
	ioctl(SIOCSIFVLAN)

	/**/
	sock_ioctl()
		vlan_ioctl_handler()
			unregister_vlan_dev()

unregister_vlan_dev()中会删除vlan设备和其宿主设备的关联， 然后unregister vlan设备 对应的net_device接口

####10.4 vlan设备的数据发送流程

Vlan设备对上层协议栈而言，和实际设备时平等的，所以它也会参与路由选择，若vlan设备被选中为出口设备， 那么上层最终会调用dev_queue_xmit(vlan_dev)来发送数据， vlan设备的tx_queue被初始化为0，所以发送流程会直接调用hard_dev_start_xmit()函数

	int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,
			struct netdev_queue *txq)
	{
		......
		if (vlan_tx_tag_present(skb) &&
		    !vlan_hw_offload_capable(features, skb->vlan_proto)) {
			skb = __vlan_put_tag(skb, skb->vlan_proto,
					     vlan_tx_tag_get(skb));
			......
		}
		......
		rc = ops->ndo_start_xmit(skb, dev);
		......
	}

首先， 给skb中的数据包打上vlan tag，然后， 调用 net_device.ndo_start_xmit()来发送数据包

前面讲到过， vlan设备的发送函数被设置为 vlan_dev_hard_start_xmit()

	static netdev_tx_t vlan_dev_hard_start_xmit(struct sk_buff *skb,
					    struct net_device *dev)
	{
		......
		skb->dev = vlan->real_dev;
		......
		ret = dev_queue_xmit(skb);
		......
	}

其中所做的是， 将skb的发送设备重置为vlan接口的宿主设备， 然后调用dev_queue_xmit()进行发送， 在这种情况下， 数据包不会经过绑定vlan的bridge来处理

####10.5 vlan设备的数据接收流程

网络设备的接收流程，不管采用中断方式，还是NAPI方式，最终都会准备好skb，并在一个内核线程中调用netif_receive_skb(skb)函数来处理

	netif_receive_skb()
		__netif_receive_skb()
			__netif_receive_skb_core()

	
	static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
	{
		......
		if (skb->protocol == cpu_to_be16(ETH_P_8021Q) ||
	    		skb->protocol == cpu_to_be16(ETH_P_8021AD)) {
			skb = vlan_untag(skb);
			......
		}
		.....
		if (vlan_tx_tag_present(skb)) {
			......
			if (vlan_do_receive(&skb))
				goto another_round;
			......
		}
		......
		rx_handler = rcu_dereference(skb->dev->rx_handler);
			......
	}

在__netif_receive_skb_core()中

+ 首先调用vlan_untag() 脱去vlan tag
+ vlan_do_receive（）中会重定位接收设备(skb->dev)为vlan dev, 并且将低层代码误以为vlan报文的目的mac不是本机的错误纠正过来(vlan dev和其宿主dev的mac不同)
+ 现在， skb->dev 指向vlan dev，因vlan dev被绑定到网桥上， 因此，rx_handler为br_handle_frame(), 这一步会进行bridge的处理，根据需要在二层转发或者传递到3层去处理

####10.6 linux上不同的vlan间通信

Linux 的网络协议栈并不依赖硬件cache转发，因此， 所谓的三层交换其实就是路由， 要实现linux上的躲过vlan之间的通信， 只需要保证能够将数据包路由到linux上由三层来转发即可
