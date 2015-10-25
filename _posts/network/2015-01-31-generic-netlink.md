---
layout: post
title: "generic netlink"
description:
category: network
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. generic netlink
   
netlink 一文中对linux的netlink的使用做了一个简单的介绍， 可以根据需要自己定义新的netlink协议类型，内核的一些子系统(例如 uevent，netfilter)定义了一些专用的netlink协议类型， 如果没有特殊需求， 不必去自定义netlink 协议类型，大可使用generic netlink  
  
netlink仅支持32种协议类型，这在实际应用中可能并不足够， 而generic netlink支持1023个子family(一个family是一堆服务的集合)    
  
generic netlink使用定义的NETLINK_GENERIC协议类型， 并提供一组数据结构和api给内核代码使用， 无需，也不要使用kernel提供的原生的netlink的API去使用NETLINK_GENERIC协议类型  
	
generic netlink工作在NETLINK_GENERIC协议类型之上，内核中不同的用户使用generic netlink，并且希望在generic netlink上监听消息时 需要注册自己的falmily和对应的operations   
  
###2. generic netlink的消息结构  

generic netlink的消息结构如下

![generic netlink 消息结构](/images/network/generic-netlink-msg-structuer.png)
  
在generic netlink中， nlmsghdr->nlmsg_type保存了family id(必须依靠它来支持多达1023个family)  

generic netlink在netlink的payload字段作出了扩展, 在头部添加了genlmsghdr和可选的user head，genlmsghdr中保存了消息的cmd，每一个消息对应有一个cmd， 并携带若干个attr  
                       
###3. 内核中使用generic netlink  
  
####3.1 注册generic netlink family  
	
内核中使用 struct genl_family 来标识一个generic netlink family, 一个family是一堆服务的集合  
	
	struct genl_family {
		unsigned int		id;
		unsigned int		hdrsize;
		char			name[GENL_NAMSIZ];
		unsigned int		version;
		unsigned int		maxattr;
		bool			netnsok;
		bool			parallel_ops;
		int			(*pre_doit)(struct genl_ops *ops,
						struct sk_buff *skb,
						struct genl_info *info);
		void			(*post_doit)(struct genl_ops *ops,
						struct sk_buff *skb,
						struct genl_info *info);
		struct nlattr **	attrbuf;	/* private */
		struct list_head	ops_list;	/* private */
		struct list_head	family_list;	/* private */
		struct list_head	mcast_groups;	/* private */
		struct module		*module;
	};  
    
+ id : family的ID， 一定要使用GENL_ID_GENERATE宏来自动生成， 不要硬编码一个ID  
+ hdrsize	 : 自定义消息头的长度， 一般没有， 置为0
+ name : family name  
+ version	 : 协议的版本， 当前还无特殊意义， 请置为1  
+ maxattr	 : 最大支持的attr数， 是最多能有多少attr， 而不是一次能传多少attr，genl使用netlink标准的attr来传输数据， 这个值可以被设为0，为0代表不区分所收到的数据的attr type）。在接收数据时，可以根据attr type，获得指定的attr type的数据在整体数据中的位置       
+ pre_doit : 在注册的operation被回调来处理消息之前被调用， 做需要做的处理， 不需要则置为NULL
+ post_doit : 在注册的operation被回调来处理消息之后被调用， 做需要做的处理， 不需要则置为NULL  
+ module : 模块所有者， 使用 THIS_MODULE 来赋值  
+ attrbuf， ops_list， family_list， mcast_groups : 私有数据， 由generic netlink来使用，维护  
  
例如， nl80211使用generic netlink时， 注册的的family为  
  
	static struct genl_family nl80211_fam = {
		.id = GENL_ID_GENERATE,	/* don't bother with a hardcoded ID */
		.name = "nl80211",	/* have users key off the name instead */
		.hdrsize = 0,		/* no private header */
		.version = 1,		/* no particular meaning now */
		.maxattr = NL80211_ATTR_MAX,
		.netnsok = true,
		.pre_doit = nl80211_pre_doit,
		.post_doit = nl80211_post_doit,
	};  
    
注册family的API：

	static inline int genl_register_family(struct genl_family *family);
	int \__genl_register_family(struct genl_family *family);
	int genl_register_ops(struct genl_family *family, struct genl_ops *ops);
	int \__genl_register_family_with_ops(struct genl_family *family, struct genl_ops *ops);
	int genl_unregister_family(struct genl_family *family);
  
####3.2 注册generic netlink operations  
  
	struct genl_ops {
		u8			cmd;
		u8			internal_flags;
		unsigned int		flags;
		const struct nla_policy	*policy;
		int		       (*doit)(struct sk_buff *skb,
						struct genl_info *info);
		int		       (*dumpit)(struct sk_buff *skb,
						struct netlink_callback *cb);
		int		       (*done)(struct netlink_callback *cb);
		struct list_head	ops_list;
	};  
    
+ cmd : 命令名
+ flag : 各种属性， bitmap
+ internal_flags : 自定义的flag                  
+ policy : 定义asttr的规则， 如果此指针非空，genl在触发事件处理程序之前，会使用这个字段来对帧中的attr做校验（见nlmsg_parse函数）。该字段可以为空，表示在触发事件处理程序之前，不做校验  
+ doit : 回调函数， 用于处理收到的该cmd
+ dumpit : 这是一个回调函数，当genl_ops的flag标志被添加了NLM_F_DUMP以后，每次收到genl消息即会回触发这个函数。dumpit与doit的区别是:dumpit的第一个参数skb不会携带从客户端发来的数据。相反地，开发者应该在skb中填入需要传给客户端的数据，然后，并返回skb的数据长度（可以用skb->len）return。skb中携带的数据会被自动送到客户端。只要dumpit的返回值大于0，dumpit函数就会再次被调用，并被要求在skb中填入数据。当服务端没有数据要传给客户端时，dumpit要返回0。如果函数中出错，要求返回一个负值。dumpit会在doit之前调用， dumpit一般被用于项服务端请求数据的cmd， 关于doit和dumpit的触发过程，可以查看源码中的genl_rcv_msg函数  
+ ops_list : 私有字段，由generic netlink使用和维护， 用户不要访问  
  
policy 使用nla_policy数组来表示
	
	struct nla_policy {
		u16		type;
		u16		len;
	};

其中

+ len  :	表示attr的负载数据的长度， 有些数据类型(例如U8等)无需指定      
+ type :	表示attr的负载数据的类型

type 的取值如下

	NLA_STRING           Maximum length of string
	NLA_NUL_STRING       Maximum length of string (excluding NUL)
	NLA_FLAG             Unused
	NLA_BINARY           Maximum length of attribute payload
	NLA_NESTED           Don't use `len' field -- length verification is 
				done by checking len of nested header (or empty)
	NLA_NESTED_COMPAT    Minimum length of structure payload
	NLA_U8, NLA_U16,
	NLA_U32, NLA_U64,
	NLA_S8, NLA_S16,
	NLA_S32, NLA_S64,
	NLA_MSECS            Leaving the length field zero will verify the given type fits, 
				using it verifies minimum length  
  
一个policy的实例为  
  
	static const struct nla_policy my_policy[ATTR_MAX+1] = {
		[ATTR_FOO] = { .type = NLA_U16 },
		[ATTR_BAR] = { .type = NLA_STRING, .len = BARSIZ },
		[ATTR_BAZ] = { .len = sizeof(struct mystruct) },
	};
    
doit回调的第二个参数为 struct genl_info  
  
	struct genl_info {
		u32			snd_seq;
		u32			snd_portid;
		struct nlmsghdr *	nlhdr;
		struct genlmsghdr *	genlhdr;
		void *			userhdr;
		struct nlattr **	attrs;
	#ifdef CONFIG_NET_NS
		struct net *		_net;
	#endif
		void *			user_ptr[2];
	};   
    
+ snd_seq	：发送序号                
+ snd_pid	：发送客户端的PID               
+ nlhdr	：netlink header的指针                
+ genlmsghdr	：genl头部的指针（即family头部）                
+ userhdr	：用户自定义头部指针                
+ attrs	：如果定义了genl_ops->policy，这里的attrs是被policy过滤以后的结果。在完成了操作以后，如果执行正确，返回0；否则，返回一个负数。负数的返回值会触发NLMSG_ERROR消息。当genl_ops的flag标志被添加了NLMSG_ERROR时，即使doit返回0，也会触发NLMSG_ERROR消息  
  
注册netlink operations的API  

	int genl_register_ops(struct genl_family *family, struct genl_ops *ops);
	int genl_unregister_ops(struct genl_family *, struct genl_ops *ops);
	int \__genl_register_family_with_ops(struct genl_family *family, struct genl_ops *ops);

####3.3 注册generic fmily和operation的示例  
  
以nl80211的注册为例, 有些数据只列出  
  
	static struct genl_family nl80211_fam = {
		.id = GENL_ID_GENERATE,	/* don't bother with a hardcoded ID */
		.name = "nl80211",	/* have users key off the name instead */
		.hdrsize = 0,		/* no private header */
		.version = 1,		/* no particular meaning now */
		.maxattr = NL80211_ATTR_MAX,
		.netnsok = true,
		.pre_doit = nl80211_pre_doit,
		.post_doit = nl80211_post_doit,
	};
    
	enum nl80211_attrs {
		/* don't change the order or add anything between, this is ABI! */
		NL80211_ATTR_UNSPEC,

		NL80211_ATTR_WIPHY,
		NL80211_ATTR_WIPHY_NAME,
		......
	};
    
	static const struct nla_policy nl80211_policy[NL80211_ATTR_MAX+1] = {
		[NL80211_ATTR_WIPHY] = { .type = NLA_U32 },
		[NL80211_ATTR_WIPHY_NAME] = { .type = NLA_NUL_STRING, .len = 20-1 },
		......
	};
    
	static struct genl_ops nl80211_ops[] = {
		{
			.cmd = NL80211_CMD_GET_WIPHY,
			.doit = nl80211_get_wiphy,
			.dumpit = nl80211_dump_wiphy,
			.policy = nl80211_policy,
			/* can be retrieved by unprivileged users */
			.internal_flags = NL80211_FLAG_NEED_WIPHY,
		},
		{
			.cmd = NL80211_CMD_SET_WIPHY,
			.doit = nl80211_set_wiphy,
			.policy = nl80211_policy,
			.flags = GENL_ADMIN_PERM,
			.internal_flags = NL80211_FLAG_NEED_RTNL,
		},
		......
	};
    
	err = genl_register_family_with_ops(&nl80211_fam, nl80211_ops, ARRAY_SIZE(nl80211_ops));  
        
####3.4 发送generic netlink消息    
  
netlink的介绍中已经介绍过， 内核中使用 struct sk_buff 来存放要发送的消息， generic netlink提供如下宏来分配sk_buff  
  
	struct sk_buff *genlmsg_new(size_t payload, gfp_t flags)  
    
+ payload ： 为分配内存的大小， 会自动在此基础上添加netlink 消息头和family头的大小  
+ flags ：  为内核内存分配类型，一般地为GFP_ATOMIC(用于原子的上下文)或GFP_KERNEL(用于非原子上下文)  
  
接下来， 需要创建一个消息负载， 这一步明显根据提供的服务而不同， 没有什么明显的规范， 一个示例如下：  

	int rc;
	void *msg_head;

	/* create the message headers */
	msg_head = genlmsg_put(skb, pid, seq, type, 0, flags, DOC_EXMPL_C_ECHO, 1);
	if (msg_head == NULL) {
		rc = -ENOMEM;
		goto failure;
	}

	/* add a DOC_EXMPL_A_MSG attribute */
	rc = nla_put_string(skb, DOC_EXMPL_A_MSG, "Generic Netlink Rocks");
	if (rc != 0)
     		goto failure;
	/* finalize the message */
	genlmsg_end(skb, msg_head);

示例中几个主要的函数  
  
	void *genlmsg_put(struct sk_buff *skb, u32 portid, u32 seq,
			struct genl_family *family, int flags, u8 cmd)
                
用于在一个sk_buff中添加netlink的消息头和generic netlink的family头到sk_buff中  
  
	int nla_put_string(struct sk_buff *skb, int attrtype, const char *str)
                 
netlink提供的辅助API， 用于给sk_buf中的消息负载添加一个attr， 类型为string  
  
	int genlmsg_end(struct sk_buff *skb, void *hdr)
    
每一个消息对应有一个cmd， 和若干个attr， 在为负载消息添加完所有的attr后， 调用子函数， 修正netlink消息头中的nlmsg_len(消息的长度)  
  
发送单播消息使用：  
  
	int genlmsg_unicast(struct net *net, struct sk_buff *skb, u32 portid)
    
+ net ： network namespace， 网路命名空间， 是为了支持网络协议栈的多个实例(例如协议栈里面的全局数据就能够通过命名空间来区分) 以实现用户空间虚拟化   
+ skb ： 要发送的数据  
+ portid ： 指定接受者  
  
发送组播数据可以使用  
  
	int genlmsg_multicast_netns(struct net *net, struct sk_buff *skb,
				u32 portid, unsigned int group, gfp_t flags)
                      
前3个参数和发送组播消息使用的数据相同， group指定要发送的组， flags为内核内存分配类型，一般地为GFP_ATOMIC(用于原子的上下文)或GFP_KERNEL(用于非原子上下文)  
  
include/net/genetlink.h中提供了大量的辅助函数， 用于获取/设置generic netlink的消息头以及消息数据   
  
关于network namespace， 涉及到linux 的namespace的概念， 用于实现linux container虚拟化技术， 详细可以参见 https://code.csdn.net/lishiwen4/linux-notes/file/linux-namespace.md#linux-namespace 需要注意的是， 不同的namespace之间无法通信， 单虚拟化常用于服务器领域，大部分普通用都只使用系统初始化的默认namespace， 因此都处于同一个namespace中

####3.5 内核中接受netlink消息  
  
和netlink一样， generic netlink也是在注册operations时指定对应cmd的回调处理函数， genl_rcv_msg()中会处理所有的NETLINK_GENERIC子协议类型的消息， 并且根据消息中的family 头来进行消息的处理和分发  
  
###4. 用户空间使用generic netlink  
  
用户空间可以使用标准的socket API来接受/发送netlink消息， 但是，必须自己处理netlink消息头以及generic netlink的family头， 工作较为繁琐， 最好使用libnl-genl来处理较为方便

[libnl官网链接](http://www.infradead.org/~tgr/libnl/) 可以下载源码和文档