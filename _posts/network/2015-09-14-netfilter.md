---
layout: post
title: "netfilter 框架"
description:
category: network
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. netfilter

netfilter是linux中的一个网络子系统， 它提供了一套框架用于注册钩子函数， 用于实现诸如数据包过滤、网络地址转换(NAT)以及网络连接跟踪等功能


###2. netfilter 支持的协议类型

在 “uapi/netfilter.h” 中定义了netfilter支持的网络协议

	enum {
		NFPROTO_UNSPEC =  0,
		NFPROTO_IPV4   =  2,
		NFPROTO_ARP    =  3,
		NFPROTO_BRIDGE =  7,
		NFPROTO_IPV6   = 10,
		NFPROTO_DECNET = 12,
		NFPROTO_NUMPROTO,
	};

可见， netfilter 支持 ipv4, arp, bridge, ipv6, decnet 这些网络协议

###3. netfilter支持的hook点

netfilter 可以在支持的网络数据包的处理路径上的多个注册点上注册钩子函数， 在 “linux/netfilter.h” 中定义了netfilter支持的hook点

	enum nf_inet_hooks {
		NF_INET_PRE_ROUTING,
		NF_INET_LOCAL_IN,
		NF_INET_FORWARD,
		NF_INET_LOCAL_OUT,
		NF_INET_POST_ROUTING,
		NF_INET_NUMHOOKS
	};

一共5个hook点：

+ PRE_ROUTING
+ LOACL_IN
+ FORWARD
+ LOCAL_OUT
+ POST_ROUTING

**针对不同的网络协议， 这些hook点的位置也不相同**

###4. netfilter的钩子函数

在linux kernel的“linux/netfilter.h” 中定义了netfilter钩子函数的原型

	typedef unsigned int nf_hookfn(unsigned int hooknum,
			       struct sk_buff *skb,
			       const struct net_device *in,
			       const struct net_device *out,
			       int (*okfn)(struct sk_buff *));

其中各参数的意义为

+ hooknum ： 当前的网络协议， 可取值有(NFPROTO_IPV4,NFPROTO_ARP,NFPROTO_BRIDGE,NFPROTO_IPV6,NFPROTO_DECNET)
+ sk_buff :  当前要处理的skb指针
+ in : skb 的入口的net_device， 通常在NF_IP_PRE_ROUTING, NF_IP_LOCAL_IN时指定， 其它时候为NULL (NF_IP_FORWARD时也可能指定)
+ out : skb 的出口的net_device， 通常在NF_IP_LOCAL_OUT和NF_IP_POST_ROUTING时指定， 其它时候为NULL (NF_IP_FORWARD时也可能指定)
+ okfn : 整个hook chain有一个返回值：返回成功netfilter会调用一个okfn回调函数来处理sb，通常okfn是协议栈的处理函数；反之释放skb

钩子函数的返回值在linux 源码中的 “uapi/netfilter.h” 中定义

	#define NF_DROP 0			
	#define NF_ACCEPT 1		
	#define NF_STOLEN 2		
	#define NF_QUEUE 3			
	#define NF_REPEAT 4		
	#define NF_STOP 5			

	#define NF_MAX_VERDICT NF_STOP

+ NF_DROP		: skb将会被skb_free(), 即既不跑剩下的hook chain， 也不被网络协议栈继续处理
+ NF_ACCEPT	: 继续跑剩下的hook chain
+ NF_STOLEN	: skb的生命周期由该hook函数来接管(hook函数可以free它， 也可以交给网络协议栈处理)， 即skb不再进行剩下的hook chain处理， 也不会由netfilter交给网络协议栈来处理
+ NF_QUEUE	: 暂缓hook chain的执行，将skb交由用户空间的hook来处理，处理完毕后，使用nf_reinject返回内核空间继续执行后面的hook
+ NF_REPEAT	: 重新跑一次该hook函数， 然后继续跑剩下的hook chain
+ NF_STOP		: 不再跑剩余的hook chain，由网络协议栈继续处理

###5. 注册/注销hook函数

####5.1 定义nf_hook_ops实例

在向netfilter注册hook函数之前， 需要定义一个 nf_hook_ops实例

	struct nf_hook_ops {
		struct list_head list;	//用于将同一类型的hook点中的hook函数形成hook chain

		nf_hookfn *hook;		//hook函数的指针
		struct module *owner;	//模块owner 通常为 THIS_MODULE
		u_int8_t pf;		//网络协议 NFPROTO_xxx
		unsigned int hooknum;	//hook点的类型 NF_INET_xxxx
		int priority;		//该hook函数的优先级
	};

####5.2 注册 nf_hook_ops

注册一个 nf_hook_ops

	int nf_register_hook(struct nf_hook_ops *reg)

注册多个 nf_hook_ops

	int nf_register_hooks(struct nf_hook_ops *reg, unsigned int n)

####5.3 注消 nf_hook_ops

注消一个 nf_hook_ops

	void nf_unregister_hook(struct nf_hook_ops *reg)	

注消多个 nf_hook_ops

	void nf_unregister_hooks(struct nf_hook_ops *reg, unsigned int n)

###6. netfilter 框架的实现

####6.1 hook chain的实现

在netfilter框架的源码中， 定义了一个全局数组

	struct list_head nf_hooks[NFPROTO_NUMPROTO][NF_MAX_HOOKS] __read_mostly;

一个二维数组， 用于存储链表头， 其中， 第一维通过 网络协议的类型(NFPROTO_xxx)来索引, 第2维通过hook点的类型(NF_INET_xxx)来索引， 在注册 nf_hook_ops时， 会根据 nf_hook_ops.pf 和 nf_hook_ops.hooknum, 选择对应的链表头， 将其添加到链表中，添加的过程中， 按照优先级大小排列， 优先级越大， 在链表中越靠前， 相同优先级时， 先添加的靠前

	int nf_register_hook(struct nf_hook_ops *reg)
	{
		struct nf_hook_ops *elem;
		int err;

		err = mutex_lock_interruptible(&nf_hook_mutex);
		if (err < 0)
			return err;
	l	ist_for_each_entry(elem, &nf_hooks[reg->pf][reg->hooknum], list) {
			if (reg->priority < elem->priority)
				break;
		}
		list_add_rcu(&reg->list, elem->list.prev);
		mutex_unlock(&nf_hook_mutex);
	#if defined(CONFIG_JUMP_LABEL)
		static_key_slow_inc(&nf_hooks_needed[reg->pf][reg->hooknum]);
	#endif
		return 0;
	
相同网络协议类型和hook点类型的nf_hook_ops实例被放置在同一个链表中， 形成一个hook chain

####6.2 放置hook点

netfilter框架的实现需要在不同的网络协议代码中不同的位置放置hook点， 当数据包流过该hook点时， 运行对应的hook chani来进行过滤， 有3个函数于放置hook点

	int NF_HOOK_THRESH(uint8_t pf, unsigned int hook, struct sk_buff *skb, struct net_device *in, 
			struct net_device *out, int (*okfn)(struct sk_buff *), int thresh);
	
	int NF_HOOK_COND(uint8_t pf, unsigned int hook, struct sk_buff *skb, struct net_device *in,
			struct net_device *out, int (*okfn)(struct sk_buff *), bool cond);

	int NF_HOOK(uint8_t pf, unsigned int hook, struct sk_buff *skb, struct net_device *in, 
			struct net_device *out, int (*okfn)(struct sk_buff *));

其中的参数为
	
+ pf ： 当前的网络协议， 可取值有(NFPROTO_IPV4,NFPROTO_ARP,NFPROTO_BRIDGE,NFPROTO_IPV6,NFPROTO_DECNET)
+ hook ： hook点类型， 可选的值有(PRE_ROUTING, LOACL_IN, FORWARD, LOCAL_OUT, POST_ROUTING)
+ sk_buff :  当前要处理的skb指针
+ in : skb 的入口的net_device， 通常在NF_IP_PRE_ROUTING, NF_IP_LOCAL_IN时指定， 其它时候为NULL (NF_IP_FORWARD时也可能指定)
+ out : skb 的出口的net_device， 通常在NF_IP_LOCAL_OUT和NF_IP_POST_ROUTING时指定， 其它时候为NULL (NF_IP_FORWARD时也可能指定)
+ okfn : 整个hook chain有一个返回值：返回成功netfilter会调用一个okfn回调函数来处理sb，通常okfn是协议栈的处理函数；反之释放skb
+ thresh ： 用于指定一个优先级， 只有优先级大于该值的hook函数才会被执行
+ cond ： 指定条件， 只有当其为真的时候， 才会跑hook chain， 否则直接运行okfn

这3个函数的差别如下：

+ NF_HOOK_THRESH() 是最基础的函数
+ NF_HOOK_COND() 是 NF_HOOK_THRESH()的特例， 它们使用 INT_MIN 作为 thresh 参数来调用 NF_HOOK_THRESH()， 因此， hook chain中任何hook函数都能被执行， 另外， 它还会检查一个条件， 只有当条件满足时， 才会去执行hook chain中的hook函数
+ NF_HOOK() 是 NF_HOOK_THRESH()的特例， 它们使用 INT_MIN 作为 thresh 参数来调用 NF_HOOK_THRESH()， 因此， hook chain中任何hook函数都能被执行

这3个函数最终都是调用 nf_hook_thresh()来完成的

**若kernel配置为不支持netfilter时， 则这三个函数会直接被替换为调用okfn**

示例

	NF_HOOK(NFPROTO_ARP, NF_ARP_OUT, skb, NULL, skb->dev, dev_queue_xmit);
	NF_HOOK(NFPROTO_BRIDGE, NF_BR_LOCAL_OUT, skb, NULL, skb->dev, br_forward_finish);
	NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,ip_rcv_finish);

####6.3 hook chain的执行

上述的3个放置hook点的函数都是通过 这3个函数最终都是调用 nf_hook_thresh()来完成的， 那么， 当数据包流经一个hook点时， hook chain的处理过程如下

	NF_HOOK_xxx()
		if (nf_hooks_active(pf, hook))  return nf_hook_slow
			nf_iterate()

当该hook点为活动时， 调用nf_hook_slow() 其中会寻找对应的 nf_hook_ops 链表， 然后调用nf_iterate()来运行该hook chain

	unsigned int nf_iterate(struct list_head *head,
			struct sk_buff *skb,
			unsigned int hook,
			const struct net_device *indev,
			const struct net_device *outdev,
			struct nf_hook_ops **elemp,
			int (*okfn)(struct sk_buff *),
			int hook_thresh)
	{
		unsigned int verdict;

		list_for_each_entry_continue_rcu((*elemp), head, list) {
			if (hook_thresh > (*elemp)->priority)
				continue;
	repeat:
			verdict = (*elemp)->hook(hook, skb, indev, outdev, okfn);
			if (verdict != NF_ACCEPT) {

				if (verdict != NF_REPEAT)
					return verdict;
				goto repeat;
			}
		}
		return NF_ACCEPT;
	}

只有优先级符合要求的hook函数才会被运行， 我们需要关注hook 函数的不同返回值对整个hook chain的影响

+ NF_ACCEPT : hook 函数返回该值后， 继续执行剩余的hook函数， 若无剩余的hook函数， 则返回NF_ACCEPT
+ NF_REPEAT : hook 函数返回该值后，该hook函数会被再次执行一次， 然后继续执行剩余的hook函数， 若无剩余的hook函数， 则返回NF_ACCEPT
+ NF_DROP	 NF_STOLEN NF_QUEUE NF_STOP	 : 终止hook chain的运行并且返回

nf_hook_slow()中会处理整个hook chain的返回值

	int nf_hook_slow(u_int8_t pf, unsigned int hook, struct sk_buff *skb,
		 struct net_device *indev,
		 struct net_device *outdev,
		 int (*okfn)(struct sk_buff *),
		 int hook_thresh)
	{
		struct nf_hook_ops *elem;
		unsigned int verdict;
		int ret = 0;

		/* We may already have this, but read-locks nest anyway */
		rcu_read_lock();

		elem = list_entry_rcu(&nf_hooks[pf][hook], struct nf_hook_ops, list);
	next_hook:
		verdict = nf_iterate(&nf_hooks[pf][hook], skb, hook, indev,
			     outdev, &elem, okfn, hook_thresh);
		if (verdict == NF_ACCEPT || verdict == NF_STOP) {
			ret = 1;
		} else if ((verdict & NF_VERDICT_MASK) == NF_DROP) {
			kfree_skb(skb);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
		} else if ((verdict & NF_VERDICT_MASK) == NF_QUEUE) {
			 err = nf_queue(skb, elem, pf, hook, indev, outdev, okfn, verdict >> NF_VERDICT_QBITS);
			if (err < 0) {
				if (err == -ECANCELED)
					goto next_hook;
				if (err == -ESRCH && (verdict & NF_VERDICT_FLAG_QUEUE_BYPASS))
					goto next_hook;
				kfree_skb(skb);
			}
		}
		rcu_read_unlock();
		return ret;
	}

+ 返回 NF_ACCEPT 和 NF_STOP 时， nf_hook_slow()返回1
+ 返回 NF_DROP 时， skb_free() 当前的skb， nf_hook_slow()返回NF_DROP
+ 返回 NF_QUEUE 时， 调用nf_queue()将数据包发往用户空间进行处理， 处理完后，数据包会通过nf_reinject()返回, 继续进行处理
+ 返回 NF_STOLEN 时， nf_hook_slow()不做处理， 直接返回NF_DROPNF_STOLEN

当nf_hook_slow()返回1时， hook点提供的okfn会被执行， 总结起来， 在如下的情况下， hook点的okfn会被执行

+ 若内核配置为不支持netfilter， 则所有的HOOK点会直接执行okfn
+ 若为条件hook点求且条件不成立， 则直接执行okfn
+ 当hook点的hook chain中， 每一个hook函数都返回NF_ACCEPT (也有可能是NF_REPEAT), hook chain返回1， okfn被执行

###7. netfilter 的 setsockopt() / getsockopot() 接口

netfilter框架提供了一套接口， 用于注册 setsockopt() 和 getsockopt() cmd， 例如， iptables， ebtables的内核部分代码都可以利用netfilter的框架来注册， 然后用户空间的iptables， ebtables命令可以利用setsockopt 和 getsockopt来和kernel中的iptables， ebtables代码通信

首先， 定义一个struct nf_sockopt_ops 实例

	struct nf_sockopt_ops {
		struct list_head list;

		u_int8_t pf;

		int set_optmin;
		int set_optmax;
		int (*set)(struct sock *sk, int optval, void __user *user, unsigned int len);
	#ifdef CONFIG_COMPAT
		int (*compat_set)(struct sock *sk, int optval, void __user *user, unsigned int len);
	#endif
		int get_optmin;
		int get_optmax;
		int (*get)(struct sock *sk, int optval, void __user *user, int *len);
	#ifdef CONFIG_COMPAT
		int (*compat_get)(struct sock *sk, int optval, void __user *user, int *len);
	#endif
		struct module *owner;
	};

其中的成员为

+ list : 用于形成链表
+ pf : 模块要针对的网络协议， NFPROTO_xxx
+ set_optmin : setsockopt() 的第3个参数为 optname，set_optmin指定支持的optname的起始值
+ set_opmax  : setsockopt() 的第3个参数为 optname，set_optmax指定支持的optname的结束值， 和 set_optmin构成支持的 optname的范围
+ set 和 compat_set : 处理setsockopt的函数  compat_set用于处理 64-bit kernel and 32-bit userspace的情况 
+ get_optmin ： getsockopt() 的第3个参数为 optname，get_optmin指定支持的optname的起始值
+ get_optmax :  getsockopt() 的第3个参数为 optname，get_optmax指定支持的optname的结束值， 和 get_optmin构成支持的 optname的范围
+ get 和 compat_get :  处理getsockopt的函数  compat_get用于处理 64-bit kernel and 32-bit userspace的情况 
+ owner : 模块的owner， 通常指定为 THIS_MODULE宏 

其它使用netfilter的模块使用如下的方式来注册/注销一个  nf_sockopt_ops 实例

	int nf_register_sockopt(struct nf_sockopt_ops *reg);
	void nf_unregister_sockopt(struct nf_sockopt_ops *reg);

示例

	ret = nf_register_sockopt(&arpt_sockopts);
	ret = nf_register_sockopt(&ebt_sockopts);
	ret = nf_register_sockopt(&ipt_sockopts);
	ret = nf_register_sockopt(&ip6t_sockopts);

所有注册的struct nf_sockopt_ops实例都会被添加到一个链表中去 

	static LIST_HEAD(nf_sockopts);

kernel 在处理系统调用时通过 nf_setsockopt() / nf_setsockopt() 来处理通过netfilter框架来注册的 sockopt

	nf_setsockopt()
		nf_sockopt
			nf_sockopt_find()
				nf_sockopt_ops->set()

	nf_getsockopt()
		nf_sockopt
			nf_sockopt_find()
				nf_sockopt_ops->get()
				