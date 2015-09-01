---
layout: post
title: "raw socket"
description:
category: network
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. raw socket

常用的网络编程都是在应用层的报文的收发操作，, 这些数据包都是由系统中的协议栈来处理，用户只需要填充应用层报文即可，由系统完成底层报文头的填充并发送， 在某些情况下需要执行更底层的操作，比如修改报文头、避开系统的协议栈， 这个时候就需要使用其他的方式来实现

在linux上， 支持2种raw socket

+ AF_INET 协议族的 SOCK_RAW 类型的socket，实现了IP 层的raw socket
+ AF_PACKET 协议族的 SOCK_RAW和 SOCK_DGRAM类型的socket， 实现了数据链路层的raw socket


###2. AF_INET raw socket

AF_INET 支持 SOCK_RAW  类型的原始套接字， 支持IP协议之上的协议类型， 例如tcp， udp， icmp等等， 创建方式类似于

	socket(AF_INET, SOCK_RAW, IPPROTO_TCP)

注意最后的 protocol 参数不能使用 IPROTO_IP, 因为如果使用IPPROTO_IP，系统根本就不知道该用什么协议

以一个udp数据包为例

	14byte            20byte      8byte        100byte  4byte
	+-----------------+-----------+------------+--------+-----+
	| Ethernet Header | ip header | udp header |  data  | fcs |
	+-----------------+-----------+------------+--------+-----+

则 “socket(AF_INET, SOCK_RAW, IPPROTO_UDP)” 可以接收到 ip header + udp header + fcs （20 + 8 + 100 byte）的数据

####2.1 AF_INET raw socket的能力

AF_INET raw socket 可以获取发往本机的ip数据包， 但是不能完成如下任务：

1. 不能收到非发往本地ip的数据包(ip软过滤会丢弃这些不是发往本机ip的数据包)
2. 不能收到从本机发送出去的数据包

关于 AF_INET 的raw socket， 还有几点如下

1. 在发送数据时， AF_INET 的raw socket需要自己组织 tcp/udp/icmp这些协议头
2. AF_INET 的raw socket 默认无法操作ip header
3. 如果用户想要自行填充ip header， 则应该使用 setsockopt来设置IP_HDRINCL选项
4. 默认会收到所有该类型的数据包， 但是可以通过bind()来指定本地的ip， connect()来指定对端的ip的方式， 进行过滤
5. 若不设置IP_HDRINCL， 如果使用bind函数绑定本地IP， 则用此IP填充源IP地址， 若不调用bind则会将源IP地址设置为外出接口的主IP地址
6. 如果使用connect函数设置目标IP，则可以使用send或者write函数发送报文，而不需要使用sendto函数
7. 端口对于SOCK_RAW而言没有任何意义
8. 不支持分片包， 需手动分片

**AF_INET 的 SOCK_PACKET 类型的套接字可能不再支持，请不要再使用， 请使用AF_PACKET协议族来代替**

**AF_INET 的raw socket处理的只是ip层及其以上的数据，比如实现SYN FLOOD攻击、处理PING报文等， 当需要操作更底层的数据的时候，就需要采用其他的方式**

####2.2 AF_INET raw socket获取数据包

数据帧进入网络层之后的处理过程如下：

	ip_rcv()
		ip_rcv_finish()
			dst_input()
				ip_local_deliver()
					raw_local_deliver()
					xxx_rcv()		/* 例如 tcp_v4_rcv()  udp_rcv() */

1. 数据包进入ip层输入例程 ip_rcv() 检查如果是非本机IP的数据包并且也非广播数据包， 则丢弃该数据包， 否则， 进行后续处理
2. ip数据包在重组(如果需要)后， 首先在raw_local_deliver()中处理raw socket
3. 处理完raw socket后， 数据包被发送到对应的协议的处理例程中 (例如 tcp_v4_rcv) 进行常规的处理

raw_local_deliver() 对raw socket的处理流程如下

	raw_local_deliver()
		raw_v4_input()
			raw_rcv()
				raw_rcv_skb()

1. raw_local_deliver() 中检查是否有通过 `socket(AF_INET, SOCK_RAW, ...)` 创建的套接字，若有，且协议符合， 则由raw_v4_input()处理
2. raw_v4_input()会遍历所有需要接收该数据包的socket， 依次拷贝一个数据包， 然后调用raw_rcv()来处理
3. raw_rcv()会调用raw_rcv_skb()将数据包拷贝socket的缓冲区中


###3. AF_PACKET raw soket

linux中添加了 AF_PACKET 协议族， 用于取代旧的 AF_INET中的SOCK_PACKET 套接字， AF_PACKET raw soket 可以操作链路层数据

AF_PACKET 协议族支持2种类型的套接字

+ SOCK_RAW ： 用户接收的数据包将包含链路层的协议头(即Ethernet 帧头)， 发送数据时， 用户需要填充链路层的协议头(即Ethernet 帧头)
+ SOCK_DGRAM ： 用户接收/发送的数据包都不包含链路层的协议头(即Ethernet 帧头)， 发送数据时， 用户指定的物理层地址只是作为参考， kernel将自动填充合适的物理层地址

AF_PACKET 协议族的socket需要指定protocol， 例如

	socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP))
	socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ARP))
	socket(PF_PACKET, SOCK_RAW, htons(ETH_P_EAPOL))
	socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL))

ETH_P_ALL可以匹配所有类型的以太网帧

以一个udp数据包为例

	14byte            20byte      8byte        100byte  4byte
	+-----------------+-----------+------------+--------+-----+
	| Ethernet Header | ip header | udp header |  data  | fcs |
	+-----------------+-----------+------------+--------+-----+

+ SOCK_RAW 可以接收到 Ethernet header + ip header + udp header + data  即 (14 + 20 + 8 + 100 byte)
+ SOCK_DGRAM 可以接收到 ip header + udp header + data  即 (20 + 8 + 100 byte)

####3.1 AF_PACKET raw soket的能力

AF_PACKET raw soket的能力如下：

+ 能接收发往本地mac的数据帧
+ 能接收从本机发送出去的数据帧(**第3个参数需要设置为ETH_P_ALL**才行)
+ 接收非发往本地mac的数据帧(网卡需要设置为promisc混杂模式)

**AF_PACKET 协议族的socket， 还可以使用bind() 指定网络接口， 只接收该接口上的数据帧**

####3.2 获取远端发送给本机的数据

首先， 任何网络协议的代码， 都可以通过 dev_add_pack() 来注册一个 struct packet_type， 来声明其感兴趣的以太网帧，感兴趣的网络设备接口， 以及对应的处理函数， 例如

+ ip协议栈注册了一个名为“ip_packet_type” 的结构体，关注类型为 ETH_P_IP的以太网帧, 处理处理函数为 ip_rcv()
+ arp协议注册了一个名为“arp_packet_type” 的结构体，关注类型为 ETH_P_ARP的以太网帧, 处理函数为arp_rcv()
					
关注 ETH_P_ALL 的packet_type会被添加到 ptype_all 链表中， 关注 ETH_P_ALL 的packet_type会被添加到 ptype_all 链表中， 而其它的的packet_type会被添加到 ptype_base 数组中对应的链表中， AF_PACKET 族的socket在创建时， 会注册一个对应的packet_type

	netif_rx()
		netif_receive_skb()
			__netif_receive_skb()
				__netif_receive_skb_core()

在 __netif_receive_skb_core() 中：

1. 先遍历 ptype_all 链表中的所有packet_type, 若数据帧由其感兴趣的网络接口所接收， 则复制一份数据帧交给其处理函数去处理
2. 继续遍历 ptype_base 对应的链表中的所有packet_type, 若数据帧由其感兴趣的网络接口所接收，且数据帧类型符合，则复制一份数据帧交给其处理函数去处理
		
####3.3 获取本机发出的数据

数据包进入数据链路层的过程如下

	dev_queue_xmit()

	dev_hard_start_sxmit()
		dev_queue_xmit_nit()
			

在dev_hard_start_sxmit()中， 会遍历 ptype_all 链表中的 packet_type, 若若数据帧由其感兴趣的网络接口所发送， 则则复制一份数据帧交给其处理函数去处理

**因为在发送过程中， 只会遍历 ptype_all 链表， 因此， 只有在创建时， 使用 ETH_P_ALL 的raw socket才能接收本机发送出去的数据帧**


###4. 总结

总的来说， AF_PACKET raw socket功能更为强大和全面，不但可以获取本机收到的数据帧， 还可以获取本机发出的数据帧， 常被用于本机网络数据的监控， 抓包， 例如， tcpdump就使用了 AF_PACKET raw socket， 而AF_INET raw socket 的功能相对较弱， 只能获取本机接收到的数据包， 不能获取本机发送的数据包， 但是可以直接过滤IP层之上的协议， 因此， 常用于开发 ICMP IGMP相关的工具

