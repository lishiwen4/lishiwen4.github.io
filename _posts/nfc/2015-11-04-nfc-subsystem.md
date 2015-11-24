---
layout: post
title: "linux NFC subsystm"
description:
category: nfc
tags: [nfc]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux上传统的nfc软件架构

nfc chip 一般使用 I2C， SPI， USB这些接口同host连接， 在linux上， 各厂商通常各自开发driver， 然后使用不同的方式实现driver和userspace的交互(例如使用一个misc设备，并且实现用户操作), 而在userspace中， 需要适配 libnfc-nci ， 最后利用 libnfc-nci 来开发应用

以 android 为例， 其 nfc 的软件架构为:

![android nfc 架构](/images/nfc/android-nfc-stack.png)

+ nfc driver 建立一个misc设备文件， 并且实现其文件操作方法
+ Hal 层实现 libnfc-nci 定义的方法， 利用文件操作misc 设备的文件操作接口同driver通信
+ libnfc-nci 利用 Hal 层， 使用 NCI 接口来操作nfc adapter

**android中hal层被编译为动态连接库 /system/lib/hw/nfc-nci.xxx.default， 其中hal要实现的接口定义在 hardware/libhardware/include/hardware/nfc.h中**

+ open()
+ close)()
+ write()
+ core_initialized()
+ pre_discover()
+ control_granted()
+ power_cycle()

###2. linux上未来的nfc软件架构

从上一节可以看出， 在linux上， nfc 软件架构的实现比较混乱，大部分的功能都由厂商各自实现， 为此linux kernel实现了 NFC driver subsystem 来统一 nfc driver 的开发，(nfc subsystem 由 intel maintain)， 未来linux上的nfc 软件架构如下图

![nfc subsystem](/images/nfc/nfc-subsystem.png)

+ nfc subsystem 的核心是 kernel 中的 NFC core
   + NFC core 使用 generic netlink 接收 userspace 的控制命令 或者 广播事件到 userspace
   + NFC core 实现了 AF_NFC 地址族 的 NFC_SOCKPROTO_RAW 协议用于交换 NFC 数据
   + NFC core 实现了 AF_NFC 地址族 的 NFC_SOCKPROTO_LLCP 协议用于支持 NFC 的 P2P mode
+ 不同的NFC芯片厂商只需要实现nfc chip driver，并且可以使用3种不同的方式来实现driver
   + 1 直接基于 NFC core 来开发 nfc chip driver
   + 2 基于 NFC HCI layer 来开发 nfc chip driver
   + 3 若 nfc chip 支持 NCI 接口， 可以基于 NFC NCI layer 来开发 nfc chip driver
   + 最好基于 HCI 或者 NCI 来开发nfc chip driver， 这样可以重用大部分功能， 加快开发
+ userspace 运行 neard 守护进程，使用 AF_NFC socket 和 netlink 来同 kernel 中的 NFC subsystem 交互
   + neard 中使用插件来实现不同的NFC 协议， 例如 SNEP， handover 等
+ userspace 中使用 neard 提供的功能来开发 application

###3 NFC subsystem

nfc subsystem 的源码位于 kernel 的 net/nfc 目录中， 如下几个 config 选项用于控制 nfc subsystem 的编译

+ CONFIG_NFC : 控制 nfc subsystem 是否编译进kernel
   + CONFIG_NFC_NCI : 控制是否编译 NCI layer 进 kernel
   + CONFIG_NFC_HCI : 控制是否编译 HCI layer 进 kernel
      + CONFIG_NFC_SHDLC : 控制 HCI layer 是否使用 SHDLC link layer

NFC subsystem初始化入/出口为 nfc_init()

####3.1 struct nfc_dev

nfc subsystem中使用 struct nfc_dev 来表示一个nfc device

	struct nfc_dev {
		int idx;				// 该 nfc adapter 的编号， 由kernel中的idr系统依次分配
		u32 target_next_idx;		// 用于为该nfc adapter 发现到的target分配编号， 从0依次递增
		struct nfc_target *targets;		// 发现的tyarget的数组
		int n_targets;			// 该adapter 某一次扫描时， 扫描到的target的数目
		int targets_generation;		// 该adapter 发现target的次数， 若一次发现了多个target， 仍计为1次
		struct device dev;
		bool dev_up;			// 该 nfc device的状态， 是否被up
		u8 rf_mode;			// 该 nfc device 的rf模式，Initiator 或者 Target
		bool polling;			// 该 nfc device是否处于 polling target的状态
		struct nfc_target *active_target;	// 该 nfc device 当前激活的目标设备
		bool dep_link_up;			// 该 nfc device的 dep_link 的状态， 是否被up
		struct nfc_genl_data genl_data;
		u32 supported_protocols;		// 该 nfc device 支持的protocol

		u32 supported_se;			// 该 nfc device 使用的 SE 安全单元
		u32 active_se;			// 该 nfc device 当前使用的 SE 安全单元

		int tx_headroom;
		int tx_tailroom;

		struct timer_list check_pres_timer;
		struct work_struct check_pres_work;

		bool shutting_down;

		struct rfkill *rfkill;

		struct nfc_ops *ops;		// 该nfc device driver要实现的操作方法
	};

#####3.1.1 nfc device 的角色

struct nfc_dev.rf_mode 的取值范围如下

	#define NFC_RF_INITIATOR 0
	#define NFC_RF_TARGET    1
	#define NFC_RF_NONE      2

#####3.1.2 nfc device支持的nfc protocol

struct nfc_dev.supported_protocols 声明了该nfc device支持的nfc protocol， 需由driver来指明， 取值范围如下(可进行“or”操作)

	#define NFC_PROTO_JEWEL_MASK	(1 << NFC_PROTO_JEWEL)
	#define NFC_PROTO_MIFARE_MASK	(1 << NFC_PROTO_MIFARE)
	#define NFC_PROTO_FELICA_MASK	(1 << NFC_PROTO_FELICA)
	#define NFC_PROTO_ISO14443_MASK	(1 << NFC_PROTO_ISO14443)
	#define NFC_PROTO_NFC_DEP_MASK	(1 << NFC_PROTO_NFC_DEP)
	#define NFC_PROTO_ISO14443_B_MASK	(1 << NFC_PROTO_ISO14443_B)

####3.1.3 nfc device支持的SE安全单元

struct nfc_dev.supported_se 声明了该nfc device支持的 SE 安全单元的类型， 需由driver来指明， 取值范围如下(可进行“or”操作)

	#define NFC_SE_NONE     0x0
	#define NFC_SE_UICC     0x1		/* 即SIM卡中的SE */
	#define NFC_SE_EMBEDDED 0x2		/* nfc chip 中内置的SE */

####3.1.4 nfc device的操作方法

struct nfc_dev.ops 保存了该 nfc device 的操作方法

	struct nfc_ops {

		/* truen on/off adapter */
		int (*dev_up)(struct nfc_dev *dev);
		int (*dev_down)(struct nfc_dev *dev);

		/* start/stop 发现目标设备的过程 */
		int (*start_poll)(struct nfc_dev *dev, u32 im_protocols, u32 tm_protocols);
		void (*stop_poll)(struct nfc_dev *dev);

		/* */
		int (*dep_link_up)(struct nfc_dev *dev, struct nfc_target *target,
			u8 comm_mode, u8 *gb, size_t gb_len);
		int (*dep_link_down)(struct nfc_dev *dev);

		/* 激活/停止激活 某个目标设备 */
		int (*activate_target)(struct nfc_dev *dev, struct nfc_target *target, u32 protocol);
		void (*deactivate_target)(struct nfc_dev *dev, struct nfc_target *target);

		/* Initiator/Target 模式下的传输方法 */
		int (*im_transceive)(struct nfc_dev *dev, struct nfc_target *target,
			struct sk_buff *skb, data_exchange_cb_t cb, void *cb_context);
		int (*tm_send)(struct nfc_dev *dev, struct sk_buff *skb);

		/**/
		int (*check_presence)(struct nfc_dev *dev, struct nfc_target *target);

		/* enable/disable SE安全单元 */
		int (*enable_se)(struct nfc_dev *dev, u32 secure_element);
		int (*disable_se)(struct nfc_dev *dev, u32 secure_element);
	};

每一个nfc_dev至少要实现nfc_ops中的如下回调方法

+ start_poll()
+ stop_poll()
+ activate_target()
+ deactivate_target()
+ im_transceive()

#####3.1.5 nfc device class

nfc subsystem 为nfc device 注册了名为 “nfc” 的 device class 对应  “/sys/class/nfc” 目录

####3.2 struct nfc_target

配置为 Poll模式的NFC device 会打开射频场，并根据配置模式进行发现过程，来发现周围的NFC设备。在NFC规范中，发现的顺序为NFC-A -> NFC-B -> NFC-F -> 私有技术， 在一次发现过程中， 可能会发现多个目标设备， nfc subsystem 使用 struct nfc_target 来表示一个被发现的目标设备

	struct nfc_target {
		u32 idx;				//id, 由对应的 struct nfc_dev.target_next_idx 赋值
		u32 supported_protocols;		//支持的 nfc protocol， 同struct nfc_dev的该字段
		u16 sens_res;			
		u8 sel_res;			
		u8 nfcid1_len;
		u8 nfcid1[NFC_NFCID1_MAXSIZE];
		u8 sensb_res_len;
		u8 sensb_res[NFC_SENSB_RES_MAXSIZE];
		u8 sensf_res_len;
		u8 sensf_res[NFC_SENSF_RES_MAXSIZE];
		u8 hci_reader_gate;
		u8 logical_idx;
	};

被发现的目标设备的信息被保存在对应的 `struct nfc_dev.targets` 和 `struct nfc_dev.n_targets` 中

####3.3 generic netlink “nfc”

nfc subsystem注册了一个名为 “nfc” 的 generic netlink， 用于同userspace之间交换控制信息，linxu kernel 中的 枚举类型 enum nfc_commands 为 generic netlink 的 “nfc” family 定义了哟一些 cmd 和 event

#####3.3.1 generic netlink cmd

+ NFC_CMD_GET_DEVICE : 获取某一个或者所有的 nfc adapter 的信息
   + 可以在消息中附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_CMD_DEV_UP : turn on 指定的nfc adapter
   + 需要在消息中附带 NFC_ATTR_DEVICE_INDEX 属性来指定nfc adapter
+ NFC_CMD_DEV_DOWN : turn off 指定的nfc adapter
   + 需要在消息中使用 NFC_ATTR_DEVICE_INDEX 属性来指定nfc adapter
+ NFC_CMD_DEP_LINK_UP : 
   + 需要在消息中附带 NFC_ATTR_DEVICE_INDEX 和 NFC_ATTR_COMM_MODE 属性，可以选择附带 NFC_ATTR_TARGET_INDEX 属性
+ NFC_CMD_DEP_LINK_DOWN : 
   + 需要在消息中附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_CMD_START_POLL :  指定某一个nfc adapter 使用指定的协议开始 polling nfc target 
   + 需要在消息中附带 NFC_ATTR_DEVICE_INDEX 和 NFC_ATTR_PROTOCOLS 属性
+ NFC_CMD_STOP_POLL :  stop polling nfc target 
   + 需要在消息中附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_CMD_GET_TARGET : dump 指定的nfc adapter polling到的nfc target
   + 需要在消息中附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_CMD_LLC_GET_PARAMS : 获取一个 nfc adapter 的 LTO， RW， MIUX 参数
   + 返回的消息中会附带 NFC_ATTR_DEVICE_INDEX， NFC_ATTR_LLC_PARAM_LTO， NFC_ATTR_LLC_PARAM_RW， NFC_ATTR_LLC_PARAM_MIUX 属性
+ NFC_CMD_LLC_SET_PARAMS : 设置一个 nfc adapter 的 LTO， RW， MIUX 参数
   + 消息中应该附带 NFC_ATTR_DEVICE_INDEX， NFC_ATTR_LLC_PARAM_LTO， NFC_ATTR_LLC_PARAM_RW， NFC_ATTR_LLC_PARAM_MIUX 属性
+ NFC_CMD_ENABLE_SE : enable 指定的nfc adapter与指定的SE单元的物理连接，即意味着在nfc emulation mode时， 使用基于SE的卡模拟
   + nfc subsystem 暂时还不会处理该cmd， 可能后续的更新 中会加入支持
+ NFC_CMD_DISABLE_SE : disable 指定的nfc adapter与指定的SE单元的物理连接，即意味着在nfc emulation mode时， 使用基于软件的卡模拟
   + nfc subsystm 暂时还不会处理该cmd， 可能后续的更新 中会加入支持
+ NFC_CMD_LLC_SDREQ : 
   + 消息中需要附带 NFC_ATTR_DEVICE_INDEX 和 NFC_ATTR_LLC_SDP 属性

这些netlink cmd由userspace发送到kernel中， nfc subsystem 在 nfc_genl_ops[] 数组中定义了上述 cmd 的 handler

	static struct genl_ops nfc_genl_ops[] = {
		{
			.cmd = NFC_CMD_GET_DEVICE,
			.doit = nfc_genl_get_device,
			.dumpit = nfc_genl_dump_devices,
			.done = nfc_genl_dump_devices_done,
			.policy = nfc_genl_policy,
		},
		{
			.cmd = NFC_CMD_DEV_UP,
			.doit = nfc_genl_dev_up,
			.policy = nfc_genl_policy,
		},
		
		......

		{
			.cmd = NFC_CMD_LLC_SDREQ,
			.doit = nfc_genl_llc_sdreq,
			.policy = nfc_genl_policy,
		},
	};

**这些cmd handler中，与nfc device相关的操作最终会调用 struct nfc_dev.ops 中对应的方法**

#####3.3.2 generic netlink event

+ NFC_EVENT_TARGETS_FOUND : 当有一个新的nfc target被发现时， 广播该event
   + 消息中会附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_EVENT_DEVICE_ADDED : 当有新的nfc adapter 被注册到nfc core时， 发送该event
   + 消息中会附带 NFC_ATTR_DEVICE_NAME, NFC_ATTR_DEVICE_INDEX 和 NFC_ATTR_PROTOCOLS 属性
+ NFC_EVENT_DEVICE_REMOVED : 当有新的nfc adapter从nfc core注销时， 发送该event
   + 消息中会附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_EVENT_TARGET_LOST : 当target lost (比如操作还未完成时移开nfc tag) 时， 发送该event
   + 消息中会附带带 NFC_ATTR_DEVICE_NAME 属性 或者 NFC_ATTR_TARGET_INDEX 属性中的一个
+ NFC_EVENT_TM_ACTIVATED : 当某一个nfc adapter被激活为 target mode时， 发送该event
   + 消息中会附带 NFC_ATTR_DEVICE_INDEX 和 NFC_ATTR_TM_PROTOCOLS 属性
+ NFC_EVENT_TM_DEACTIVATED : 当某一个 nfc adapter被取消激活时， 发送该event
   + 消息中会附带 NFC_ATTR_DEVICE_INDEX 属性
+ NFC_EVENT_LLC_SDRES :
   + 消息中会附带 NFC_ATTR_DEVICE_INDEX 和 NFC_ATTR_LLC_SDP 属性

这些 event 则由 nfc subsystem 广播到 userspace 中， nfc subsystem 为 generic netlink family “nfc” 注册了一个名为 “events” 的广播组用于广播event

	#define NFC_GENL_MCAST_EVENT_NAME "events"

	static struct genl_multicast_group nfc_genl_event_mcgrp = {
		.name = NFC_GENL_MCAST_EVENT_NAME,
	};

####3.4 AF_NFC 族 socket

nfc subsystem 使用socket接口同userspace进行数据交换， 因此， 定义了 AF_NFC 地址族

	#define AF_NFC		39	/* NFC sockets			*/

在 AF_NFC 地址族上， nfc subsystem 实现了2种socket 协议

	/* NFC socket protocols */
	#define NFC_SOCKPROTO_RAW	0	// for R/W and C/E mode
	#define NFC_SOCKPROTO_LLCP	1	// for P2P mode

**在阅读这一节之前， 建议先阅读 “network” 分类中的 “socket 接口的网络协议无关性” 一文**

#####3.4.1 NFC_SOCKPROTO_RAW 协议

对于一个网络协议来说， 我们需要关注其对应的 struct proto 和 struct proto_ops， 对于NFC_SOCKPROTO_RAW 则有

	static struct proto rawsock_proto = {
		.name     = "NFC_RAW",
		.owner    = THIS_MODULE,
		.obj_size = sizeof(struct nfc_rawsock),
	};

	static const struct proto_ops rawsock_ops = {
		.family         = PF_NFC,
		.owner          = THIS_MODULE,
		.release        = rawsock_release,
		.bind           = sock_no_bind,
		.connect        = rawsock_connect,
		.socketpair     = sock_no_socketpair,
		.accept         = sock_no_accept,
		.getname        = sock_no_getname,
		.poll           = datagram_poll,
		.ioctl          = sock_no_ioctl,
		.listen         = sock_no_listen,
		.shutdown       = sock_no_shutdown,
		.setsockopt     = sock_no_setsockopt,
		.getsockopt     = sock_no_getsockopt,
		.sendmsg        = rawsock_sendmsg,
		.recvmsg        = rawsock_recvmsg,
		.mmap           = sock_no_mmap,
	};

可见， rawsock_ops中只是实现了几种操作方式， 该socket协议的名称为“NFC_RAW”, protocol id 为 NFC_SOCKPROTO_RAW

NFC_SOCKPROTO_RAW 协议使用 struct nfc_rawsock 来表示该协议的socket

	struct nfc_rawsock {
		struct sock sk;
		struct nfc_dev *dev;		//nfc device 的id
		u32 target_idx;			//目标设备的id
		struct work_struct tx_work;		//tx workqueue
		bool tx_work_scheduled;		//tx workqueue是否正在被调度
	};

该协议仅支持 SOCK_SEQPACKET 类型，因此，建立该类型的socket时， `socket()调用的参数如下`

	socket(AF_NFC, SOCK_SEQPACKET, NFC_SOCKPROTO_RAW)

调用后， 在kernel中将分配一个 nfc_rawsock, 并且有

	struct nfc_rawsock.sk.sk_protocol = NFC_SOCKPROTO_RAW
	struct nfc_rawsock.sk.ops = rawsock_ops
	
在userspace使用该类型的socket的一般步骤为

	socket()
	connect()
	sendmsg()
	recvmsg()
	......
	close()

在使用 `connect()`系统调用时， 需要指定远端地址, AF_NFC地址族中为 NFC_SOCKPROTO_RAW 协议定义了 地址struct sockaddr_nfc

	struct sockaddr_nfc {
		sa_family_t sa_family;	//family为 AF_NFC
		__u32 dev_idx;		//nfc device 的编号
		__u32 target_idx;		//目标设备的编号， 由发现它的nfc设备分配
		__u32 nfc_protocol;	//要使用的nfc 协议
	};

地址中需要指定nfc device 和 nfc target 的 id， 其中nfc device 的为其driver注册时由nfc subsystem分配， 而nfc target的id则由发现它的nfc device来分配， 在userspace中可以使用上一节介绍的generic netlink cmd来获取

1. 通过netlink发送 `NFC_CMD_START_POLL` cmd 来发起发现nfc target的过程
2. 当 nfc device 发现nfc target后， 其driver会通过nfc core的接口发送 netlink 广播 `NFC_EVENT_TARGETS_FOUND`
3. userspace 通过netlink接收到 `NFC_EVENT_TARGETS_FOUND` 事件后，发送 netlink cmd `NFC_CMD_GET_TARGET` 来获取nfc target的信息

**在userspace调用 connect() 时， 会自动调用nfc_ops.activate_target() 来激活nfc target, 而在userspace调用 close() 时， 会自动调用nfc_ops.deactivate_target() 来取消激活的nfc target**

使用 sendmsg()/recvmsg() 来交换数据时， 具体的数据内容和使用的nfc协议相关， 每一次调用 sendemsg() 后， 都有一个response， 可以通过 recvmsg() 来接收， 为了允许empty response， NFC_SOCKPROTO_RAW 协议传输数据时会添加一字节的header，由于 sendmsg() 是异步的，userspace() 在调用 sendmsg() 后 可调用poll() 来等待可读写

####3.4.2 NFC_SOCKPROTO_LLCP 协议

同样， 先来看对应的 struct proto 和 struct proto_ops

	static struct proto llcp_sock_proto = {
		.name     = "NFC_LLCP",
		.owner    = THIS_MODULE,
		.obj_size = sizeof(struct nfc_llcp_sock),
	};

	static const struct proto_ops llcp_sock_ops = {
		.family         = PF_NFC,
		.owner          = THIS_MODULE,
		.bind           = llcp_sock_bind,
		.connect        = llcp_sock_connect,
		.release        = llcp_sock_release,
		.socketpair     = sock_no_socketpair,
		.accept         = llcp_sock_accept,
		.getname        = llcp_sock_getname,
		.poll           = llcp_sock_poll,
		.ioctl          = sock_no_ioctl,
		.listen         = llcp_sock_listen,
		.shutdown       = sock_no_shutdown,
		.setsockopt     = nfc_llcp_setsockopt,
		.getsockopt     = nfc_llcp_getsockopt,
		.sendmsg        = llcp_sock_sendmsg,
		.recvmsg        = llcp_sock_recvmsg,
		.mmap           = sock_no_mmap,
	};

	static const struct proto_ops llcp_rawsock_ops = {
		.family         = PF_NFC,
		.owner          = THIS_MODULE,
		.bind           = llcp_raw_sock_bind,
		.connect        = sock_no_connect,
		.release        = llcp_sock_release,
		.socketpair     = sock_no_socketpair,
		.accept         = sock_no_accept,
		.getname        = llcp_sock_getname,
		.poll           = llcp_sock_poll,
		.ioctl          = sock_no_ioctl,
		.listen         = sock_no_listen,
		.shutdown       = sock_no_shutdown,
		.setsockopt     = sock_no_setsockopt,
		.getsockopt     = sock_no_getsockopt,
		.sendmsg        = sock_no_sendmsg,
		.recvmsg        = llcp_sock_recvmsg,
		.mmap           = sock_no_mmap,
	};

可以看到，NFC_SOCKPROTO_LLCP 提供了两个 struct proto_ops 结构， 这是因为， 与 NFC_SOCKPROTO_RAW 不同， NFC_SOCKPROTO_LLCP 支持3种类型的socket，提供2种服务

+ SOCK_DGRAM / SOCK_STREAM : 提供面向连接的服务， 对应的 proto_ops 为 llcp_sock_ops
+ SOCK_RAW : 提供无连接的服务， 对应的 proto_ops 为 llcp_rawsock_ops

因此， 在 userspace 调用 `socket()` 系统调用来建立socket时， 可用的参数组合有

	socket(AF_NFC, SOCK_RAW, NFC_SOCKPROTO_LLCP);

	/* 这两种socket type相同 */
	socket(AF_NFC, SOCK_STREAM, NFC_SOCKPROTO_LLCP);
	socket(AF_NFC, SOCK_DGRAM, NFC_SOCKPROTO_LLCP);

NFC_SOCKPROTO_LLCP协议 使用 struct nfc_lcp_sock 来表示该协议的socket

	struct nfc_llcp_sock {
		struct sock sk;
		struct nfc_dev *dev;
		struct nfc_llcp_local *local;
		u32 target_idx;
		u32 nfc_protocol;

		/* Link parameters */
		u8 ssap;
		u8 dsap;
		char *service_name;
		size_t service_name_len;
		u8 rw;
		__be16 miux;


		/* Remote link parameters */
		u8 remote_rw;
		u16 remote_miu;

		/* Link variables */
		u8 send_n;
		u8 send_ack_n;
		u8 recv_n;
		u8 recv_ack_n;

		/* Is the remote peer ready to receive */
		u8 remote_ready;

		/* Reserved source SAP */
		u8 reserved_ssap;

		struct sk_buff_head tx_queue;
		struct sk_buff_head tx_pending_queue;

		struct list_head accept_queue;
		struct sock *parent;
	};

在userspace 中， 调用 `connect()` 或者 `bind()` 系统调用时， 需要指定地址， AF_NFC地址族中为 NFC_SOCKPROTO_LLCP 协议定义了 地址struct sockaddr_nfc_llcp

	struct sockaddr_nfc_llcp {
		sa_family_t sa_family;
		__u32 dev_idx;
		__u32 target_idx;
		__u32 nfc_protocol;
		__u8 dsap; /* Destination SAP, if known */
		__u8 ssap; /* Source SAP to be bound to */
		char service_name[NFC_LLCP_MAX_SERVICE_NAME]; /* Service name URI */;
		size_t service_name_len;
	};

**NDEF和Handover协议都是使用 NFC_SOCKPROTO_LLCP 提供的面向连接的服务**

相较于 NFC_SOCKPROTO_RAW， NFC_SOCKPROTO_LLCP 要复杂许多，所以会有单独的章节来介绍 nfc subsystem 之 NFC_SOCKPROTO_LLCP

###4. NFC subsystem 提供的接口

在基于nfc subsystem开发driver时， 可以使用nfc subsystem提供的接口

####4.1 nfc device相关的接口

nfc subsystem 如下的接口用于 alloc/free nfc_dev

	struct nfc_dev *nfc_allocate_device(struct nfc_ops *ops,
					u32 supported_protocols,
					u32 supported_se,
					int tx_headroom, int tx_tailroom)

	static inline void nfc_free_device(struct nfc_dev *dev);

在alloc nfc_dev 之后， 还需要向nfc core注册该  nfc_dev，使用如下的接口来 注册/注销 nfc_dev

	int nfc_register_device(struct nfc_dev *dev);
	void nfc_unregister_device(struct nfc_dev *dev);

如下接口用于通过 nfc_dev 的 id 来获取对应的 nfc_dev 指针并且维护引用计数

	struct nfc_dev *nfc_get_device(unsigned int idx)

如下的接口用于解除对 nfc_dev 的引用

	static inline void nfc_put_device(struct nfc_dev *dev)

如下的3个接口配合使用， 可用于遍历所有的 nfc_dev

	static inline void nfc_device_iter_init(struct class_dev_iter *iter)
	static inline struct nfc_dev *nfc_device_iter_next(struct class_dev_iter *iter)
	static inline void nfc_device_iter_exit(struct class_dev_iter *iter)

####4.2 sk_buff 相关的接口

当nfc device接收到数据， 需要通过 AF_NFC socket 接口返回给userspace时， 分配缓冲区 

	struct sk_buff *nfc_alloc_recv_skb(unsigned int size, gfp_t gfp);

会自动预留1byte的空header以支持返回空数据

####4.3 generic netlink event 相关接口

nfc core 和 nfc device driver 使用如下的接口来广播不同的事件

	/* 由 nfc device driver 调用以广播 NFC_EVENT_TARGETS_FOUND 事件 */
	int nfc_targets_found(struct nfc_dev *dev, struct nfc_target *targets, int n_targets)

	/* 由 nfc core 或者 nfc device driver 调用以广播 NFC_EVENT_TARGET_LOST 事件 */
	int nfc_target_lost(struct nfc_dev *dev, u32 target_idx)

	/* 由 nfc device driver 调用以广播 NFC_EVENT_TM_ACTIVATED 事件 */
	int nfc_tm_activated(struct nfc_dev *dev, u32 protocol, u8 comm_mode, u8 *gb, size_t gb_len)

	/* 由 nfc device driver 调用以广播 NFC_EVENT_TM_DEACTIVATED 事件 */
	int nfc_tm_deactivated(struct nfc_dev *dev)
	
	/* 由 nfc core 调用以广播 NFC_EVENT_DEVICE_ADDED 事件 */
	int nfc_genl_device_added(struct nfc_dev *dev)

	/* 由 nfc core 调用以广播 NFC_EVENT_DEVICE_REMOVED 事件 */
	int nfc_genl_device_removed(struct nfc_dev *dev)

	/* 由 调用以广播 NFC_EVENT_LLC_SDRES 事件  */
	int nfc_genl_llc_send_sdres(struct nfc_dev *dev, struct hlist_head *sdres_list)


	用于广播  NFC_CMD_DEP_LINK_UP
	int nfc_dep_link_is_up(struct nfc_dev *dev, u32 target_idx, u8 comm_mode, u8 rf_mode)

####4.4 llcp 相关的接口

用于llcp 且处于 target 模式时， 通知 nfc subsystem llcp 有数据到来

	int nfc_tm_data_received(struct nfc_dev *dev, struct sk_buff *skb)

	int nfc_set_remote_general_bytes(struct nfc_dev *dev, u8 *gb, u8 gb_len)
	u8 *nfc_get_local_general_bytes(struct nfc_dev *dev, size_t *gb_len)

####4.5 其它接口

用于通知nfc subsystem driver错误， 不能继续去完成目标设备发现过程
	void nfc_driver_failure(struct nfc_dev *dev, int err)
