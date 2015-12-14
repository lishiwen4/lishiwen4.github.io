---
layout: post
title: "IEEE 802.11 无线网络接口的类型"
description:
category: wifi
tags: [wifi, network, linux]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 物理网卡与虚拟网络接口

在linux中， 物理上的无线网卡使用 struct wiphy 来表示， 对应 "/sys/class/ieee80211/" 目录，一个无线 网卡可以支持多个虚拟网络接口， 虚拟网络接口使用 struct net_device 来表示， 对应"/sys/class/net/" 目录 (参照iw 中的phy和dev， 分别代表物理网卡和虚拟网络接口)

在ieee80211无线网络的结构中， STA具有不同的类型， 例如 AP， STA， AD-HOC 等， 称为无线网络接口的工作模式， 也被称为无线网络接口的类型

WNIC (Wireless Network Interface Controler) 可以在下述的模式中的一种中工作， 不同的模式决定了无线连接的主要功能， 若无线网络接口能够支持多个虚拟接口， 则无线网卡还可以同时工作在不同的模式

###2. STA 

Station infrastructure mode

基本上所有的wifi driver都能支持这种模式， 因此也被称为default模式

在 使用WEXT的tool中(例如 iwconfig， iw)， STA 模式也被称为 managed 模式

###3. AP 

AccessPoint infrastructure mode

在基础型网络结构中, AP作为网络中的master设备， 所有的STA要通过AP来交换数据

###4. MON 

Monitor mode

Monitor mode是一种完全的被动模式，不发送任何帧， 并且所有接收到的帧都会发送到Host CPU， 不经过任何过滤， 通常用于抓取的sniffer log

使用mac80211, 可以为一个无线网络卡设置monitor模式， 作为正常模式(STA)的附加, 即2者共存， 当然， 不是所有的hardware都支持monitor模式， 也不是所有的hardware都支持monitor模式和正常模式共存

使用mac80211， 可以在monitor模式下发送数据包， 常用于数据帧注入，可用于在用户空间实现MLME

###5. IBSS 

Idependent Basic Service Set mode , 也被称为 Ad-Hoc mode 

Ad-Hoc 模式也被称为 IBBS (Independent Basic Service Set) 模式， 可用于建立一个无需AP的无线网络

###6. WDS 

Wireless Distribution System mode

DS (Distribution System) 是给AP提供上行链接， WDS也是实现相同的目的， 只是是以无线的形式

在无线应用领域中， WDS都是帮助无线基站与无线基站之间进行联系通讯的系统

在家庭应用中， WDS的功能是充当无线网络的中继器，通过在无线路由器上开启WDS功能，让其可以延伸扩展无线信号，从而覆盖更广更大的范围

WDS 模式涉及到 4 address

###7. Mesh

Mesh 模式用于在多个设备之间建立智能路由， 实现互相通信

###8. P2P-Client

充当p2p模式中的 client

###9. P2P-Go

充当p2p模式中的group

###10. nl80211中的虚拟网络接口类型

	enum nl80211_iftype {
		NL80211_IFTYPE_UNSPECIFIED,
		NL80211_IFTYPE_ADHOC,
		NL80211_IFTYPE_STATION,
		NL80211_IFTYPE_AP,
		NL80211_IFTYPE_AP_VLAN,	//必须绑定到一个NL80211_IFTYPE_AP类型的接口上
		NL80211_IFTYPE_WDS,
		NL80211_IFTYPE_MONITOR,
		NL80211_IFTYPE_MESH_POINT,
		NL80211_IFTYPE_P2P_CLIENT,
		NL80211_IFTYPE_P2P_GO,
		NL80211_IFTYPE_P2P_DEVICE,	//这不是一个netdev， 因此必须用NL80211_CMD_START_P2P_DEVICE
					//和NL80211_CMD_STOP_P2P_DEVICE来建立和销毁

		/* keep last */
		NUM_NL80211_IFTYPES,
		NL80211_IFTYPE_MAX = NUM_NL80211_IFTYPES - 1
	};

从 NL80211_IFTYPE_ADHOC 到 NL80211_IFTYPE_P2P_DEVICE，和前述的无线网络接口的的类型相对应

###11. wiphy支持的接口类型

wiphy 标识一个物理无线网卡， 被依次命名为 phy0，phy1，......phyn， 使用iw 命令可以查看物理网卡支持的工作模式

	# iw phy phy0  info
	Wiphy phy0
	max # scan SSIDs: 9
	max scan IEs length: 255 bytes
	Retry short limit: 7
	Retry long limit: 4
	Coverage class: 0 (up to 0m)
	Device supports roaming.
	......
	Supported interface modes:
		* IBSS
		* managed
		* AP
		* P2P-client
		* P2P-GO
	......

即该物理网卡支持 IBSS， STA， AP， P2P-Client， P2P-Go 这些工作模式

同样， iw 还能够查看一个无线网络接口的工作模式， 例如

	# iw dev wlan0 info                                         
	Interface wlan0
		ifindex 43
		wdev 0x700000001
		addr 00:0a:f5:0c:33:d8
		type managed
		wiphy 7

	# iw dev p2p0 info                                            	
		Interface p2p0
		ifindex 44
		wdev 0x700000002
		addr 02:0a:f5:0c:33:d8
		ssid DIRECT-0Y-Android_c359
		type P2P-client
		wiphy 7

	# iw dev p2p0 info                                            
		Interface p2p0
		ifindex 30
		wdev 0x2
		addr 1e:b7:2c:9e:1e:26
		ssid DIRECT-0Y-Android_c359
		type P2P-GO
		wiphy 0

###12. wiphy支持的接口类型组合

wiphy可以支持多个工作模式同时操作， 即一个物理无线网卡上可以支持多个虚拟网卡，受限于hw， fw， driver的实现， 不同的网卡能够支持的组合有所不同

nl80211中使用如下的数据结构来描述wiphy能够支持的接口类型组合：

	struct ieee80211_iface_limit {
		u16 max;		//该类型接口的最大数目
		u16 types;	//接口的类型， 如果有多个不同类型的接口有总数限制， 则因该对这些类型做or操作
	};

	struct ieee80211_iface_combination {
		const struct ieee80211_iface_limit *limits;	// ieee80211_iface_limit 结构体的元素个数
		u32 num_different_channels;			// 可以同时工作的channel的数目
		u16 max_interfaces;			// 允许的接口数目的最大值
		u8 n_limits;				// limits 中 ieee80211_iface_limit 结构体的数目
		bool beacon_int_infra_match;		// STA模式和AP模式下的beacon interval是否要匹配
		u8 radar_detect_widths;			// 雷达探测所支持的chennel宽度的位图
	};

相关的信息保存在 wiphy 结构体中

	struct wiphy {
		......

		// 存放 ieee80211_iface_combination 结构体数组， 若wiphy支持多种不同的组合， 
		// 则需要使用多个 ieee80211_iface_combination ， 大部分情况下， 只支持一种组合
		const struct ieee80211_iface_combination *iface_combinations;	

		// iface_combinations 成员中 ieee80211_iface_combination 结构体的个数
		int n_iface_combinations;					
		....
	}

例如：

	struct ieee80211_iface_limit wlan_hdd_iface_limit[] = {
		{
			.max = 3,
			.types = BIT(NL80211_IFTYPE_STATION),
		},
		{
			.max = 1,
			.types = BIT(NL80211_IFTYPE_ADHOC) | BIT(NL80211_IFTYPE_AP),
		},
		{
			.max = 1,
			.types = BIT(NL80211_IFTYPE_P2P_GO) |
			BIT(NL80211_IFTYPE_P2P_CLIENT),
		},
	};

	struct ieee80211_iface_combination wlan_hdd_iface_combination = {
		.limits = wlan_hdd_iface_limit,
		.num_different_channels = 1,
		.max_interfaces = 3,
		.n_limits = ARRAY_SIZE(wlan_hdd_iface_limit),
		.beacon_int_infra_match = false,
	};

即该wiphy最大支持

+ 3个STA类型的接口
+ 1个类型的 AD-HOC 或者 AP 类型的接口
+ 1个类型的 P2P-GO 或者 P2P-Client 类型的接口

当然， 该wiphy最大支持3个虚拟网络接口

使用iw tool能够查看wiphy 支持的接口组合

	# iw phy phy7 info
	Wiphy phy7
		......
		valid interface combinations:
			* #{ managed } <= 3, #{ IBSS, AP } <= 1, #{ P2P-client, P2P-GO } <= 1,
			  total <= 3, #channels <= 1

即最大支持3个 STA 接口， 最大支持1个IBSS接口或者AP接口， 最大支持1个 P2P-Go 接口或者 P2P-client 接口， 并且接口类型的总数最大为3个， 且只能工作在一个channel

###13. 使用iw添加网络接口

在支持多个虚拟网络接口的无线网卡上面， 可以使用 iw 来添加无线网络接口， iw可以用于添加如下类型的网络接口

+ monitor
+ managed
+ wds
+ mesh
+ ibss

例如：

	iw phy phy0 interface add moni0 type monitor

	iw phy phy0 interface add wlan10 type managed

	// mesh id 为 “mymesh”
	iw phy phy0 interface add mesh0 type mp mesh_id mymesh

	iw phy phy0 interface add wds0 type wds

删除iw添加的网络接口则需要使用“iw dev xx del”， 例如

	iw dev moni0 del

