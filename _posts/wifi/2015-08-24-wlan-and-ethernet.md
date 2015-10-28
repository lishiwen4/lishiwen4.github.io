---
layout: post
title: "wifi 与 以太网"
description:
category: wifi
tags: [wifi, network]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 局域网技术

几种局域网技术如下：

1. 以太网(Ethernet)
2. 令牌环网
3. FDDI 网
4. ATM 网
5. 无线局域网(WLAN)

###2. 2种以太网标准

请先阅读 “network” 分类中的 “以太网帧格式” 这一章

在IEEE制定以太网标准之前， DEC， Intel Xeron 制定了 Ethernet I 和 Ethernet II的标准， 并且被 TCP/IP 选作局域网的标准

后续， IEEE退出了 IEEE802.3/802.2 LLC 以及 IEEE802.3/802.2 SNAP 以太网标准， 但是设备厂商为了考虑与旧设备的兼容性， 并未积极使用 IEEE 以太网标准， 因此， Ethernet II是事实上的以太网标准

###3. wifi MAC帧 与 Ethernet 帧的转换

wifi(IEEE 802.11)作为IEEE推出的标准， 采用 LLC SNAP 来封装IP层数据， 然后将其作为 IEEE 802.11 的MAC层数据 (即相当于将 IEEE 以太网帧的 LCC header 和 数据部分 封装到 802.11 MAC 帧中)

在 wifi 与 Ethernet 需要互通的情况下， 比如 “连接同一台无线路由器的wifi设备有线网络设备之间进行局域网通信” 的时候， 双方使用的是不同的帧， 这个时候就需要进行转换

参照 “network” 分类中的 “以太网帧格式”这一章， 里面有描述过， IP层数据包在Ethernet 上的封装由RFC 894来规定， 而IP层数据包在 IEEE以太网上的封装由 RFC 1042来规定

####3.1 RFC 1042 定义的转换方式

	              6byte    6byte   2byte  46~1500byte    4byte
	             +------+--------+------+---------------+-----+
	Ethernet II  | dest | source | type | data          | FCS |	
	             | MAC  | MAC    |      |               |     |
	             +------+--------+------+---------------+-----+


	              6byte   6byte    2byte    1byte  1byte 1byte   3byte        2byte  38 ~ 1492byte   4byte
	             +------+--------+--------+------+------+------+------------+------+---------------+-----+
	802.3 LLC    | dest | source | length | DSAP | SSAP | Ctrl | OUI        | type | data          | FCS |	 
	RFC 1042     | MAC  | MAC    |        | 0xAA | 0xAA | 0x03 | 0x00-00-00 |      |               |     |   
	             +------+--------+--------+------+------+------+------------+------+---------------+-----+
	                                      | --- LLC header --- | -- SNAP header -- |


	                 24 or 30byte 2byte    1byte  1byte 1byte   3byte        2byte  38 ~ 1492byte   4byte
	                +------------+--------+------+------+------+------------+------+---------------+-----+	
	802.11          | 802.11 MAC | length | DSAP | SSAP | Ctrl | OUI        | type | data          | FCS |
	RFC 1042        | header     |        | 0xAA | 0xAA | 0x03 | 0x00-00-00 |      |               |     |
	                +------------+--------+------+------+------+------------+------+---------------+-----+
	                                      | --- LLC header --- | -- SNAP header -- |
	                             | ---------------------- 802.11 MAC payload --------------------------- |

转换过程中 ：

1. 从Ethernet II到802.3 LLC帧的转换过程中， 因为添加了字段，所以需要重新计算FCS
2. 802.3 LLC 帧中除了 “目标地址”和“源地址” 外， 其余部分作为802.11的MAC帧的负载， 而 “目标地址” 和 “源地址” 则包含到 802.11 MAC header中

####3.2 802.1h 定义的转换方式

802.11 除了用RFC 1042 来封装 Ethernet 外， 还可以使用802.1h定义的方式， 802.1h与 802.3 LLC 的唯一差别是 **802.1h 使用了OUI字段， 其值为 0x00-00-f8**


**通常， 转换的过程由wifi chip来完成， 在外部直接给wifi chip输入Ethernet 帧， wifi chip会将其转换为wifi MAC 帧， 然后发送， 而接收到的wifi MAC帧，则会被转换为Ethernet 帧， 从wifi chip中读出**
