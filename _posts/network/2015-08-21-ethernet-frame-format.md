---
layout: post
title: "以太网帧格式"
description:
category: network
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 以太网

以太网是局域网技术中的一种， 几种局域网技术如下：

1. 以太网(Ethernet)
2. 令牌环网
3. FDDI 网
4. ATM 网
5. 无线局域网(WLAN)

由于以太网使用普遍、发展迅速，以至于人们将"以太网"当作了"局域网"的代名词，并且**以太网是当今 TCP/IP 网络协议采用的局域网技术(注意wifi将以太网帧作为其payload来传递)**

###2. 以太网帧的历史

1. 1980年 ： DEC， Intel Xeron 制定了 Ethernet I 的标准
   + 使用Xerox PARC提出的3Mbps CSMA/CD以太网标准的封装格式
2. 1982年 ： DEC， Intel Xeron 又制定了 Ethernet II 的标准 (cisco将其命名为ARPA)
   + 相比Ethernet I， 主要更改了电气特性和物理接口， 帧格式并没有变化
3. 1982年 ： IEEE 计划制定Ethernet的国际标准 IEEE 802.3 
4. 1983年 ： Novell根据IEEE 802.3 草案， 在其Netware/86网络套件上使用了专用的Ethernet帧格式 (cisco将其命名为Novell-Ether)
   + 2年后，IEEE正式发布802.3标准时情况发生了变化—IEEE在802.3帧头中又加入了802.2 LLC(Logical Link Control)头，这使得Novell的RAW 802.3格式跟正式的IEEE 802.3标准互不兼容
5. 1985年 ： IEEE 正式推出了 IEEE 802.3规范 (cisco将其命名为SAP)
   + 这一标准被称为 802.3/802.2 LLC， 它将Ethernet V2帧头的协议类型字段替换为帧长度字段(取值为0000-05dc;十进制的1500)；并加入802.2 LLC头用以标志上层协议，LLC头中包含DSAP，SSAP以及Crontrol字段
6. IEEE 为解决 Ethernet II 与 802.3 帧格式的兼容问题推出折衷的 (cisco将其命名为SNAP)
   + 这一标准也被称为 802.3/802.2 SNAP， 这是IEEE为保证在802.2 LLC上支持更多的上层协议同时更好的支持IP协议而发布的标准，与802.3/802.2 LLC一样802.3/802.2 SNAP也带有LLC头，但是扩展了LLC属性，新添加了一个2Bytes的协议类型域（同时将SAP的值置为0xAA），从而使其可以标识更多的上层协议类型；另外添加了一个3Bytes的OUI字段用于代表不同的组织，RFC 1042定义了IP报文在802.2网络中的封装方法和ARP协议在802.2 SANP中的实现

Ethernet II 是事实上的标准

###3. Ethernet II 帧格式

Ethernet II的帧格式如下

	6byte	6byte	2byte	46～1500byte	4byte
	|	|	|	|		|
	目的地址	源地址	类型	数据		FCS

1. 目的地址和源地址均为48bit的MAC地址
2. 类型字段指明了承载的上层协议的类型， 常见的值有
   + 0x800	IP
   + 0x806	ARP
   + 0x8137	Novell IPX
   + 0x809b	Apple Talk
   + 0x86dd	IPv6
   + 在Ethernet II 中，该字段值不得小于0x05dc(即10进制的1500),否则代表不是Ethernet II帧
   + linux源码 "kernel/include/uapi/linux/if_ether.h" 中可以看到更多的定义
3. 数据字段为承载的上层协议的数据， 长度为46～1500byte， 即使数据不够46byte， 也要填充到46byte
4. 32bit CRC校验， 校验从 “目的地址” 到 “数据” 的内容

**Ethernet II帧中并不保存长度， 因此， 需要由网卡驱动来指出长度(硬件能够获取到接收的帧的长度)**

**RFC 894定义了IP报文在Ethernet II上的封装格式**

###4. Novell Etnernet 帧格式

Novell Ethernet 帧格式如下

	6byte	6byte	2byte	2byte	44～1498byte	4byte
	|	|	|	|	|		|
	目的地址	源地址	数据长度	0xffff	数据		FCS

Novell ethernet 的帧头与 Ethernet II 有所不同：

+ Ethernet II 的类型域变为 长度 域 
+ 在长度域后面添加了两个固定的字节 0xffff 用于标识帧为 Novell Ethernet
+ 由于占用了2byte 存储 0xffff 标识符， 因此 数据字段的长度为 44～1498 byte

###5. IEEE802.2 (LLC)

IEEE802.2 标准是 IEEE802.3 等标准的基础， 因此， 在细述 IEEE802.3以太网帧结构之前， 需要先解释一下IEEE802.2

在ISO的OSI参考模型中, 数据链路层又划分成两个子层：

+ 逻辑链路控制LLC（Logic Line Control）子层
+ 介质访问控制MAC（Media Access Control）子层

**IEEE802委员会为局域网制订了一系列标准，统称为802标准. 其中IEEE802.2标准定义了逻辑链路控制LLC子层的功能与服务，并且是 IEEE802.3，IEEE802.4和 IEEE802.5等标准的基标准**

LLC 为上层提供了处理任何类型 MAC 层的方法

LLC 定义了2种数据通信操作类型：

+ LLC1 即类型1, 无连接, 该方式不保证发送的信息一定可以收到。
+ LLC2 即类型2, 面向连接, 该方式提供了四种服务：
   1. 连接的建立
   2. 确认和数据到达响应
   3. 差错恢复（通过请求重发接收到的错误数据实现）
   4. 滑动窗口, 滑动窗口用来提高数据传输速率。

LLC1是应用于以太网中，而LLC2应用在IBM SNA网络环境中

LLC header 格式为

	1byte	1byte	1byte
	|	|	|
	DSAP	SSAP	Ctrl

+ DSAP ： 目标服务存取点（Destination Service Access Point）
+ SSAP ： 源服务存取点（Source Service Access Point）
+ Ctrl ： 无连接或面向连接的LLC， 基本不使用， 一般被设为0x03，指明采用无连接服务的802.2无编号数据格式

**服务存取点(SAP)用于标识以太网帧所携带的上层数据类型**

在 “kernel/include/uapi/linux” 中可以看到SAP类型

	#define LLC_SAP_NULL	0x00		/* NULL SAP. 			*/
	#define LLC_SAP_LLC	0x02		/* LLC Sublayer Management. 	*/
	#define LLC_SAP_SNA	0x04		/* SNA Path Control. 		*/
	#define LLC_SAP_PNM	0x0E		/* Proway Network Management.	*/	
	#define LLC_SAP_IP		0x06		/* TCP/IP. 			*/
	#define LLC_SAP_BSPAN	0x42		/* Bridge Spanning Tree Proto	*/
	#define LLC_SAP_MMS	0x4E		/* Manufacturing Message Srv.	*/
	#define LLC_SAP_8208	0x7E		/* ISO 8208			*/
	#define LLC_SAP_3COM	0x80		/* 3COM. 			*/
	#define LLC_SAP_PRO	0x8E		/* Proway Active Station List	*/
	#define LLC_SAP_SNAP	0xAA		/* SNAP. 			*/
	#define LLC_SAP_BANYAN	0xBC		/* Banyan. 			*/
	#define LLC_SAP_IPX	0xE0		/* IPX/SPX. 			*/
	#define LLC_SAP_NETBEUI	0xF0		/* NetBEUI. 			*/
	#define LLC_SAP_LANMGR	0xF4		/* LanManager. 			*/
	#define LLC_SAP_IMPL	0xF8		/* IMPL				*/
	#define LLC_SAP_DISC	0xFC		/* Discovery			*/
	#define LLC_SAP_OSI	0xFE		/* OSI Network Layers. 		*/
	#define LLC_SAP_LAR	0xDC		/* LAN Address Resolution 	*/
	#define LLC_SAP_RM		0xD4		/* Resource Management 		*/
	#define LLC_SAP_GLOBAL	0xFF		/* Global SAP. 

###6. IEEE802.3/802.2  LLC 帧格式

IEEE802.3/802.2 LLC 帧的格式为

	6byte	6byte	2byte	3byte	43～1497byte	4byte
	|	|	|	|	|		|
	目的地址	源地址	数据长度	LLC	数据		FCS

IEEE802.3/802.2 LLC 帧和Ethernet II的帧的差别为：

1. Ethernet II 中的类型字段变成了长度字段
2. 引入802.2协议(LLC)， 在802.3帧头后面添加了一个LLC header

###7. IEEE802.3/802.2  SNAP 帧格式

IEEE802.3/802.2  SNAP  在 IEEE802.3/802.2  SAP 的基础上， 扩充了 LLC header

	6byte	6byte	2byte	8byte	38～1492byte	4byte
	|	|	|	|	|		|
	目的地址	源地址	数据长度	LLC	数据		FCS

而LLC header的扩展为

	1byte	1byte	1byte	3byte	2byte
	|	|	|	|	|
	DSAP	SSAP	Ctrl	Org Code	type

+ DSAP ： 参见第6节中的SAP， 类型定义， SNAP为0xAA， 因此该字段固定为0xAA
+ SSAP ： 参见第6节中的SAP， 类型定义， SNAP为0xAA， 因此该字段固定为0xAA
+ Ctrl ： 固定为0x03
+ Org Code : 通常与源mac地址的前三个bytes相同， 为厂商代码
+ type 域与Ethernet II 中的 “类型” 字段相同

###8. 不同帧格式的区分方式

1. 检查帧中“源MAC”字段后面的2byte(在Ethernet II中为协议类型， 在其它帧中为 数据长度)， 若大于 0x05dc(即10进制的1500)， 则为 Ethernet 帧
2. 继续比较后面的2个byte, 如果为0xFFFF则为Novell Ether 类型的帧
3. 如果为 0xAAAA 则为 IEEE802.3/802.2 SNAP 帧， 否则， 为 IEEE 802.3/802.2 LLC 帧

这些以太网帧可以共存于一个网络中，但互不兼容，当用不同封装类型的工作站要交换信息时，必须通过 支持的路由器来通信

###9, Ethernet 与 IEEE的以太网帧的关系

在 DLL 和 PHY层， Ethernet 与 IEEE 以太网的关系可以使用下图表示


	+-----+	+----------------------------------------------------+
	|     |	|                IEEE 802.2 LLC                      |   DLL 之 LLC Sublayer
	|  E  |	+----------------------------------------------------+
	|  t  |	
	|  h  |	+------------+  +-----------+  +---------------------+
	|  e  |	| 802.3 MAC  |  | 802.5 MAC |  | 802.11 MAC          |   DLL 之 MAC Sublayer                                                     
	|  r  |	+------------+  +-----------+  +---------------------+
	|  n  |	                
	|  e  |
	|  t  |	+------------+  +-----------+  +---------------------+
	|     |	| 802.3 PHY  |  | 802.5 PHY |  | 802.11 a/b/g/ac PHY |   PHY
	+-----+	+------------+  +-----------+  +---------------------+

TCP/IP 使用Ethernet 帧作为局域网的基础， 而无线局域网 wifi， 则是使用LLC封装， 因此， 其会将Ethernet帧转换为802.3帧， 然后再转换为802.11帧

###10. 以太网帧前导码

前导码其实是在物理层发送以太网帧之前， 添加到以太网帧前面的的， 并不是正式帧的一部分， 添加前导码的目的是物理层可以在接收到实际的帧之前， 开始检测载波， 并且与接收到的帧时序达到同步稳定

前导码的结构为

	7byte	1byte
	|	|
	前导码	帧起始鉴定符

+ 前导码 ： 7byte， 内容为 循环的 010101..... bit
+ 帧起始鉴定符 ： 1byte， 内容为 10101011 最后的两个bit “11” 表示后续的字段是 “目的地址”

###11. IP数据包在 Ethernet II 以太网上的封装

IP数据包在 Ethernet II 以太网上的封装由RFC894规定

Ethernet II 帧格式为

	6byte	6byte	2byte	46～1500byte	4byte
	|	|	|	|		|
	目的地址	源地址	类型	数据		FCS

IP数据包

	2byte	46～1500byte	4byte
	|	|		|
	类型	IP数据包		FCS
	0x8000

ARP 数据包

	2byte	28byte	18byte	4byte
	|	|	|	|
	类型	ARP数据包	padding	FCS
	0x8006

RARP 数据包

	2byte	28byte		18byte	4byte
	|	|		|	|
	类型	RARP数据包	padding	FCS
	0x8035

###12. IP数据包在 IEEE 以太网上的封装

IP数据包在 IEEE 以太网上的封装由RFC1042规定

IEEE 802.3/802.2 SNAP 帧格式为

	6byte	6byte	2byte	8byte	38～1492byte	4byte
	|	|	|	|	|		|
	目的地址	源地址	数据长度	LLC	数据		FCS

IP 数据包

	2byte	38～1492byte	4byte
	|	|		|
	LLC之类型	IP数据包		FCS
	0x8000

ARP 数据包

	2byte	28byte	10byte	4byte
	|	|	|	|
	LLC之类型	ARP数据包	padding	FCS
	0x8006

RARP 数据包

	2byte	28byte		10byte	4byte
	|	|		|	|
	LLC之类型	RARP数据包	padding	FCS
	0x8035

**RFC894和RFC1042的主要差别在于， IEEE以太网相对于Ethernet II以太网， 其数据字段中有一部分分配给LLC头， 因此， IEEE以太网封装IP数据包时， 实际的数据长度上限要小一些， 并且对于ARP和RARP数据包所需的padding字节数也要小一些**

**RFC1024中规定， LLC。header 中的 Org code 值固定为 0x00-00-00 **