---
layout: post
title: "socket接口的网络协议无关性"
description:
category: network
tags: [network, linux]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. socket接口

BSD Socket 是UNIX系统中通用的网络接口， linux同样使用该接口， socket系统调用接口为

	#include <sys/types.h>          /* See NOTES */
	#include <sys/socket.h>

	int socket(int domain, int type, int protocol);

+ domain : 指定地址族
+ type : socket类型
+ protocl ： 通常为0，因为同一domain， 同一type下， 通常只有一种protocol(但是不是绝对的， 比如netlink就支持30多种protocol)

**从socket系统调用可以看出， socket是一个与网络协议无关的接口**

###2. linux上socket接口支持的网络协议 

linux上socket接口支持的domain(地址族)较多， 在linux源码的 ”kernel/include/linux/socket.h“ 中有

	/* Supported address families. */
	#define AF_UNSPEC	0
	#define AF_UNIX		1	/* Unix domain sockets 		*/
	#define AF_LOCAL	1	/* POSIX name for AF_UNIX	*/
	#define AF_INET		2	/* Internet IP Protocol 	*/
	#define AF_AX25		3	/* Amateur Radio AX.25 		*/
	#define AF_IPX		4	/* Novell IPX 			*/
	#define AF_APPLETALK	5	/* AppleTalk DDP 		*/
	#define AF_NETROM	6	/* Amateur Radio NET/ROM 	*/
	#define AF_BRIDGE	7	/* Multiprotocol bridge 	*/
	#define AF_ATMPVC	8	/* ATM PVCs			*/
	#define AF_X25		9	/* Reserved for X.25 project 	*/
	#define AF_INET6	10	/* IP version 6			*/
	#define AF_ROSE		11	/* Amateur Radio X.25 PLP	*/
	#define AF_DECnet	12	/* Reserved for DECnet project	*/
	#define AF_NETBEUI	13	/* Reserved for 802.2LLC project*/
	#define AF_SECURITY	14	/* Security callback pseudo AF */
	#define AF_KEY		15      /* PF_KEY key management API */
	#define AF_NETLINK	16
	#define AF_ROUTE	AF_NETLINK /* Alias to emulate 4.4BSD */
	#define AF_PACKET	17	/* Packet family		*/
	#define AF_ASH		18	/* Ash				*/
	#define AF_ECONET	19	/* Acorn Econet			*/
	#define AF_ATMSVC	20	/* ATM SVCs			*/
	#define AF_RDS		21	/* RDS sockets 			*/
	#define AF_SNA		22	/* Linux SNA Project (nutters!) */
	#define AF_IRDA		23	/* IRDA sockets			*/
	#define AF_PPPOX	24	/* PPPoX sockets		*/
	#define AF_WANPIPE	25	/* Wanpipe API Sockets */
	#define AF_LLC		26	/* Linux LLC			*/
	#define AF_CAN		29	/* Controller Area Network      */
	#define AF_TIPC		30	/* TIPC sockets			*/
	#define AF_BLUETOOTH	31	/* Bluetooth sockets 		*/
	#define AF_IUCV		32	/* IUCV sockets			*/
	#define AF_RXRPC	33	/* RxRPC sockets 		*/
	#define AF_ISDN		34	/* mISDN sockets 		*/
	#define AF_PHONET	35	/* Phonet sockets		*/
	#define AF_IEEE802154	36	/* IEEE802154 sockets		*/
	#define AF_CAIF		37	/* CAIF sockets			*/
	#define AF_ALG		38	/* Algorithm sockets		*/
	#define AF_NFC		39	/* NFC sockets			*/
	#define AF_VSOCK	40	/* vSockets			*/
	#define AF_MAX		41	/* For now.. */

虽然linux支持的domain非常多， 但是常用的还是只有 AF_UNIX/AF_LOCAL， AF_INET， AF_INET6， AF_NETLINK，AF_PACKET， AF_BLUETOOTH 这几种

**注意， 对于上面的每一个AF_xxx宏， 都有一个对应的PF_xxx宏， 且它们的值完全相同， 但是AF_xxx表示地址族， PF_xxx表示协议族， 调用socket()系统调用时，最好使用AF_xxx来表示domain， PF_xxx来表示protocol, 但是混用AF_xxx和PF_xxx也不会有问题**

###3. linux支持的socket类型

在linux源码中的 “kernel/include/linux/net.h”中有

	enum sock_type {
		SOCK_STREAM	= 1,
		SOCK_DGRAM	= 2,
		SOCK_RAW	= 3,
		SOCK_RDM	= 4,
		SOCK_SEQPACKET	= 5,
		SOCK_DCCP	= 6,
		SOCK_PACKET	= 10,
	};

**需要注意的是， 对于一个socket domain， 并不是支持所有的socket type**

###4. socket接口网络协议无关性的实现

linux上， socket接口的网络协议无关性的实现类似于linux上的虚拟文件系统，即定义一套统一的操作接口， 而每一种具体网络协议则实现定义好的方法

####4.1 struct socket

在linux kernel中， 一个socket使用一个struct socket来描述

	struct socket {
		socket_state		state;

		kmemcheck_bitfield_begin(type);
		short			type;
		kmemcheck_bitfield_end(type);

		unsigned long		flags;

		struct socket_wq __rcu	*wq;

		struct file		*file;
		struct sock		*sk;
		const struct proto_ops	*ops;
	};

其中比较重要的成员有

+ state ： 套接字的状态
+ type  : 套接字的类型， 即第3节中定义的的SOCK_STREAM之类的值
+ file  : 套接字在用户空间使用fd来表示， 
+ sock	: socket相关的信息很多， 但是struct socket需要放置到 inode的一个union成员中， 因此struct socket不能太大， 只有将一部分信息分离出来， 放置到独立的struct sock中
+ ops	: socket接口网络协议无关性的基础， 不同的网络协议需要实现其中定义的方法

####4.2 struct proto_ops

proto_ops中定义了如下所示的方法， 由各网络协议来实现所需的方法

	struct proto_ops {
		int		family;
		struct module	*owner;
		int		(*release)   (struct socket *sock);
		int		(*bind)	     (struct socket *sock, struct sockaddr *myaddr, int sockaddr_len);
		int		(*connect)   (struct socket *sock, struct sockaddr *vaddr, int sockaddr_len, int flags);
		int		(*socketpair)(struct socket *sock1, struct socket *sock2);
		int		(*accept)    (struct socket *sock, struct socket *newsock, int flags);
		int		(*getname)   (struct socket *sock, struct sockaddr *addr, int *sockaddr_len, int peer);
		unsigned int	(*poll)	     (struct file *file, struct socket *sock, struct poll_table_struct *wait);
		int		(*ioctl)     (struct socket *sock, unsigned int cmd, unsigned long arg);
	#ifdef CONFIG_COMPAT
		int	 	(*compat_ioctl) (struct socket *sock, unsigned int cmd, unsigned long arg);
	#endif
		int		(*listen)    (struct socket *sock, int len);
		int		(*shutdown)  (struct socket *sock, int flags);
		int		(*setsockopt)(struct socket *sock, int level, int optname, char __user *optval, unsigned int optlen);
		int		(*getsockopt)(struct socket *sock, int level, int optname, char __user *optval, int __user *optlen);
	#ifdef CONFIG_COMPAT
		int		(*compat_setsockopt)(struct socket *sock, int level, int optname, char __user *optval, unsigned int optlen);
		int		(*compat_getsockopt)(struct socket *sock, int level, int optname, char __user *optval, int __user *optlen);
	#endif
		int		(*sendmsg)   (struct kiocb *iocb, struct socket *sock, struct msghdr *m, size_t total_len);
		int		(*recvmsg)   (struct kiocb *iocb, struct socket *sock, struct msghdr *m, size_t total_len, int flags);
		int		(*mmap)	     (struct file *file, struct socket *sock, struct vm_area_struct * vma);
		ssize_t		(*sendpage)  (struct socket *sock, struct page *page, int offset, size_t size, int flags);
		ssize_t 	(*splice_read)(struct socket *sock,  loff_t *ppos, struct pipe_inode_info *pipe, size_t len, unsigned int flags);
		int		(*set_peek_off)(struct sock *sk, int val);
	};

####4.3 struct net_proto_family

如4.1和4.2两小节所讲， socket在linux kernel中使用struct socket来表示， 而与网络协议相关的操作， 则保存在 socket.ops 成员(struct proto_ops)中， 那么这两者是如何联系起来的呢， 这就要涉及到 struct net_proto_family了

	struct net_proto_family {
		int		family;
		int		(*create)(struct net *net, struct socket *sock,
				  int protocol, int kern);
		struct module	*owner;
	};

每一种地址族，使用一个 net_proto_family 结构体来表示

+ family : 地址族的id， 即第2节中定义的AF_xxx宏
+ create : 该地址族的socket的create函数， 在系统调用socket()的执行期间被调用
+ owner ： 通常赋值为 THIS_MODULE

每一种网络协议，需要在其初始化的过程中， 使用 sock_register() 来注册自己的 net_proto_family 结构体， 例如

	err = sock_register(&bt_sock_family_ops);	/* kernel/net/bluetooth/af_bluetooth.c */

	(void)sock_register(&inet_family_ops);	/* kernel/net/ipv4/af_inet.c */

	sock_register(&netlink_family_ops);		/* kernel/net/netlink/af_netlink.c */

	sock_register(&unix_family_ops);		/* kernel/net/unix/af_unix.c */

在linux kernel中， 有一个静态数组来保存所有的 net_proto_family 结构体

	static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;

sock_register() 将每一个注册的地址族， 保存在该数组中， 各地址族的 net_proto_family 结构体在 net_families 数组中的顺序， 按照其net_proto_family.family的数值的顺序

####4.4 struct proto

即使是在同一个地址族里面， 也可能存在多种协议， 为此， linux中使用struct proto来表示某一个protocol的操作， 而该地址族里面的common操作， 仍放置在proto_ops， 当然， 如果不同的protocol， 其具体操作差别不大，可以共用， 则无需定义多个struct proto， 例如 netlink支持多个protocol， 但是只定义了一个struct proto


	struct proto {
		void			(*close)(struct sock *sk, long timeout);
		int			(*connect)(struct sock *sk, struct sockaddr *uaddr, int addr_len);
		int			(*disconnect)(struct sock *sk, int flags);

		struct sock *		(*accept)(struct sock *sk, int flags, int *err);

		int			(*ioctl)(struct sock *sk, int cmd, unsigned long arg);
		int			(*init)(struct sock *sk);
					(*destroy)(struct sock *sk);
		void			(*shutdown)(struct sock *sk, int how);
		int			(*setsockopt)(struct sock *sk, int level, int optname, char __user *optval, unsigned int optlen);
		int			(*getsockopt)(struct sock *sk, int level, int optname, char __user *optval, int __user *option);
	#ifdef CONFIG_COMPAT
		int			(*compat_setsockopt)(struct sock *sk, int level, int optname, char __user *optval, unsigned int optlen);
		int			(*compat_getsockopt)(struct sock *sk, int level, int optname, char __user *optval, int __user *option);
		int			(*compat_ioctl)(struct sock *sk, unsigned int cmd, unsigned long arg);
	#endif
		int			(*sendmsg)(struct kiocb *iocb, struct sock *sk, struct msghdr *msg, size_t len);
		int			(*recvmsg)(struct kiocb *iocb, struct sock *sk, struct msghdr *msg, size_t len, int noblock, int flags, int *addr_len);
		int			(*sendpage)(struct sock *sk, struct page *page, int offset, size_t size, int flags);
		int			(*bind)(struct sock *sk, struct sockaddr *uaddr, int addr_len);

		int			(*backlog_rcv) (struct sock *sk, struct sk_buff *skb);

		void		(*release_cb)(struct sock *sk);
		void		(*mtu_reduced)(struct sock *sk);

		/* Keeping track of sk's, looking them up, and port selection methods. */
		void			(*hash)(struct sock *sk);
		void			(*unhash)(struct sock *sk);
		void			(*rehash)(struct sock *sk);
		int			(*get_port)(struct sock *sk, unsigned short snum);
		void			(*clear_sk)(struct sock *sk, int size);

		......
	};

**一个地址族可以根据需要， 定义一个或者多个struct proto结构体， 例如， 大部分的地址族只定义了一个proto结构体， 且未实现其中的方法(所有处理方法都定义在struct proto_ops中)， 但是AF_INET 和 AF_INET6 均定义了多个oproto结构体**

在socket() 系统调用的执行过程中， 对应的 net_proto_family.create()在执行过程中， 需要调用sk_alloc() 来分配 socket->sk 成员， 这一步需要根据使用的协议传递合适的proto结构体作为参数

	struct sock *sk_alloc(struct net *net, int family, gfp_t priority,
		      struct proto *prot)
	{
		......
		sk->sk_prot = sk->sk_prot_creator = prot;
		......
	}

在proto_ops中的方法中， 可以根据需要， 调用对应的socket->sk->sk_prot中的方法(如果实现了的话)来完成具体的操作

另外， 每一个proto结构体， 还需要调用 proto_register() 将其注册到链表 proto_list 中， 读取“/proc/net/protocols” 即可以从该链表中获取所有注册的网络协议的信息

####4.5 socket的create流程

	/* userspace */
	socket()

	/* kernel */
	SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
		sock_create();
			__sock_create();
		sock_map_fd();

	
	int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
	{
		......
		sock = sock_alloc();
		......
		sock->type = type;
		......
		pf = rcu_dereference(net_families[family]);
		......
		err = pf->create(net, sock, protocol, kern);
		......
	}

在__sock_create()中

1. 分配 struct socket结构体
2. 设置 socket.type为对应的type
3. 根据socket()系统调用传递的family参数， 从net_familie数组中获取对应的net_proto_family
4. 调用 对应的 net_proto_family.create()方法

**无论是那一种协议族的 net_proto_family.create()， 在其执行过程中都会将该地址族对应的 proto_ops 赋值给 分配的socket.ops成员, 因此， 根据socket()调用时传递的family参数， socket结构能够绑定对应的地址族的操作方法**

再来看sock_map_fd(), 由于socket在用户空间使用fd来描述， 因此， 在kernel中， 需要让socket和file之间建立联系

	sock_map_fd()
		sock_alloc_file()
			alloc_file(&path, FMODE_READ | FMODE_WRITE, &socket_file_ops);
				

	struct file *alloc_file(struct path *path, fmode_t mode,
		const struct file_operations *fop)
	{
		......
		file->f_op = fop;
		......
	}

**最终， 用户空间对于socket fd的文件操作， 由 socket_file_ops 来处理**

####4.6 socket操作的处理流程

	read()
		SYSCALL_DEFINE3(read)
			vfs_read()
				do_sync_read()
					file->f_op->aio_read()		/* 即 socket_file_ops.aio_read() */
						sock_aio_read()
							do_sock_read()
								__sock_recvmsg()
									__sock_recvmsg_nosec()
										socket->ops->recvmsg()

	write()
		SYSCALL_DEFINE3(write)
			vfs_write()
				do_sync_write()
					file->f_op->aio_write()		/* 即 socket_file_ops.aio_write() */
						sock_aio_write()
							do_sock_write()
								__sock_sendmsg()
									__sock_sendmsg_nosec()
										socket->ops->sendmsg()

	ioctl()
		SYSCALL_DEFINE3(ioctl)
			do_vfs_ioctl()
				vfs_ioctl()
					file->f_op->unlocked_ioctl()	/* 即 socket_file_ops.unlocked_ioctl() */
						sock_ioctl()
							dev_ioctl()
							sock_do_ioctl()
								socket->ops->ioctl()

**需要注意的是， socket fd的ioctl， 有些是与目标network interface相关的， 有些是与网络协议相关的， 因此，从 vfs_ioctl() 开始， 各层函数中都有对ioctl cmd的处理**

	listen()
		SYSCALL_DEFINE2(listen)
			socket->ops->listen()	
		
	shutdown()
		SYSCALL_DEFINE2(shutdown)
			socket->ops->shutdown()	


	bind()
		SYSCALL_DEFINE3(bind)
			socket->ops->bind()
		
	connect()
		SYSCALL_DEFINE3(connect)
			socket->ops->connect()

	accept()
		SYSCALL_DEFINE4(accept)
			sys_accept4()
				socket->ops->accept()
		
	socketpair()
		SYSCALL_DEFINE4(socketpair)
			sock_create()
			sock_create()

			socket->ops->socketpair()

	sendmsg()
		SYSCALL_DEFINE3(sendmsg)
			__sys_sendmsg()
				___sys_sendmsg()
					sock_sendmsg_nosec()
						__sock_sendmsg_nosec()
							socket->ops->sendmsg()
	sendmmsg()
		SYSCALL_DEFINE4(sendmmsg)
			__sys_sendmmsg()
				___sys_sendmsg()
					sock_sendmsg_nosec()
						__sock_sendmsg_nosec()
							sock->ops->sendmsg()

	send()
		sys_sendto(fd, buff, len, flags, NULL, 0)
			见 sendto()
	

	sendto()
		SYSCALL_DEFINE6(sendto)
			sock_sendmsg()
				__sock_sendmsg()
					__sock_sendmsg_nosec()
						sock->ops->sendmsg()	

	recvmsg()
		SYSCALL_DEFINE3(recvmsg)
			__sys_recvmsg()
				___sys_recvmsg()
					sock_recvmsg_nosec()
						__sock_recvmsg_nosec()
							socket->ops->recvmsg()

	recvmmsg()
		SYSCALL_DEFINE5(recvmmsg)
			__sys_recvmmsg()
				___sys_recvmsg()
					sock_recvmsg_nosec()
						__sock_recvmsg_nosec()
							socket->ops->recvmsg()

	recvfrom()
		SYSCALL_DEFINE6(recvfrom)
			sock_recvmsg()
				__sock_recvmsg()
					__sock_recvmsg_nosec()
						socket->ops->recvmsg()

	setsockopt()
		SYSCALL_DEFINE5(setsockopt)
			socket->ops->setsockopt()

	getsockopt()
		SYSCALL_DEFINE5(getsockopt)
			socket->ops->getsockopt()

###5. AF_INET

AF_INET 是internet网络地址族(即ipv4地址族), AF_INET6 是ipv6的版本

前面的小节提到过， 对于每一个地址族， 都应该定义一个 struct net_proto_family 结构体， 对于AF_INET来说，其为

	static const struct net_proto_family inet_family_ops = {
		.family = PF_INET,
		.create = inet_create,
		.owner	= THIS_MODULE,
	};

####5.1 AF_INET 中的网络协议

针对不同的socket类型， AF_INET的网络协议栈代码中定义了不同的几个proto_ops结构体

	const struct proto_ops inet_stream_ops = {
		.family		   = PF_INET,
		.owner		   = THIS_MODULE,
		.release	   = inet_release,
		.bind		   = inet_bind,
		.connect	   = inet_stream_connect,
		.socketpair	   = sock_no_socketpair,
		.accept		   = inet_accept,
		......
	};

	const struct proto_ops inet_dgram_ops = {
		.family		   = PF_INET,
		.owner		   = THIS_MODULE,
		.release	   = inet_release,
		.bind		   = inet_bind,
		.connect	   = inet_dgram_connect,
		.socketpair	   = sock_no_socketpair,
		.accept		   = sock_no_accept,
		......
	};

	static const struct proto_ops inet_sockraw_ops = {
		.family		   = PF_INET,
		.owner		   = THIS_MODULE,
		.release	   = inet_release,
		.bind		   = inet_bind,
		.connect	   = inet_dgram_connect,
		.socketpair	   = sock_no_socketpair,
		.accept		   = sock_no_accept,
		......
	};

可以看到， 分别针对 SOCK_STREAM， SOCK_DGRAM 和 SOCK_RAW

对于一个地址族来说， 可以支持多种网络协议， 可以将common方法保存在 struct proto_ops中， 网络协议自身的方法保存在 proto 中

+ soket.ops : 保存 struct proto_ops
+ socket.sk.sk_prot : 保存 struct proto

AF_INET 的网络协议代码中， 定义了多个 proto 结构体

	struct proto tcp_prot = {
		.name			= "TCP",
		.owner			= THIS_MODULE,
		.close			= tcp_close,
		.connect		= tcp_v4_connect,
		.disconnect		= tcp_disconnect,
		.accept			= inet_csk_accept,
		.ioctl			= tcp_ioctl,
		......
	};

	struct proto udp_prot = {
		.name		   = "UDP",
		.owner		   = THIS_MODULE,
		.close		   = udp_lib_close,
		.connect	   = ip4_datagram_connect,
		.disconnect	   = udp_disconnect,
		.ioctl		   = udp_ioctl,
		......
	};

	struct proto ping_prot = {
		.name =		"PING",
		.owner =	THIS_MODULE,
		.init =		ping_init_sock,
		.close =	ping_close,
		.connect =	ip4_datagram_connect,
		.disconnect =	udp_disconnect,
		......
	};

	struct proto raw_prot = {
		.name		   = "RAW",
		.owner		   = THIS_MODULE,
		.close		   = raw_close,
		.destroy	   = raw_destroy,
		.connect	   = ip4_datagram_connect,
		.disconnect	   = udp_disconnect,
		.ioctl		   = raw_ioctl,
		......
	};

即定义了 tcp， udp， raw socket 和 ping 这些网络协议

####5.2 struct inet_protosw

AF_INET网络协议的代码中使用了一个 struct inet_protosw 来表示各个网络协议的相关信息

	struct inet_protosw {
		struct list_head list;			/* 用于形成链表*/

		unsigned short	 type;	   		/* socket类型， 即socket()系统调用的第2个参数 */
		unsigned short	 protocol;		/* L4 protocol number，socket()系统调用的第3个参数*/

		struct proto	 *prot;			/* */
		const struct proto_ops *ops;
  
		char             no_check;   		/* checksum on rcv/xmit/none? */
		unsigned char	 flags;      		/* See INET_PROTOSW_* below.  */
	}; 

而在 AF_INET的网络协议代码里面， 定义了

	static struct inet_protosw inetsw_array[] =
	{
		{
			.type =       SOCK_STREAM,
			.protocol =   IPPROTO_TCP,
			.prot =       &tcp_prot,
			.ops =        &inet_stream_ops,
			.no_check =   0,
			.flags =      INET_PROTOSW_PERMANENT |
			      INET_PROTOSW_ICSK,
		},

		{
			.type =       SOCK_DGRAM,
			.protocol =   IPPROTO_UDP,
			.prot =       &udp_prot,
			.ops =        &inet_dgram_ops,
			.no_check =   UDP_CSUM_DEFAULT,
			.flags =      INET_PROTOSW_PERMANENT,
		},

		{
			.type =       SOCK_DGRAM,
			.protocol =   IPPROTO_ICMP,
			.prot =       &ping_prot,
			.ops =        &inet_dgram_ops,
			.no_check =   UDP_CSUM_DEFAULT,
			.flags =      INET_PROTOSW_REUSE,
       		},

      		{
	       		.type =       SOCK_RAW,
	       		.protocol =   IPPROTO_IP,	/* wild card */
	       		.prot =       &raw_prot,
	       		.ops =        &inet_sockraw_ops,
	       		.no_check =   UDP_CSUM_DEFAULT,
	       		.flags =      INET_PROTOSW_REUSE,
       		}
	};

注意里面的protocol， AF_INET中定义了多个protocol，例如 IPPROTO_IP， IPPROTO_ICMP， IPPROTO_TCP， IPPROTO_UDP， IPPROTO_RAW 等

通过struct inet_protosw， 将 socket type， socket protocol 以及对应的 struct proto 和 struct proto关联到了一起， 对于这些 inet_protosw 结构体， 需要调用 inet_register_protosw()  将他们注册到静态链表 inetsw 中

	static struct list_head inetsw[SOCK_MAX]

**inetsw是一个数组， 为每一种socket type保存一个链表头， 即 inet_protosw 结构体按照socket type分类， 放在不同的链表里面**

####5.3 AF_INET socket的create过程

从 socket 的create 过程来看上述的数据结构是如何联系到一起的

	/* userspace */
	socket()

	/* kernel */
	SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
		sock_create();
			__sock_create();
				net_proto_family->create()			/* 即inet_create() */

	
	static int inet_create(struct net *net, struct socket *sock, int protocol,
		       int kern)
	{
		......
		list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {
			......
			if (protocol == answer->protocol) {
				if (protocol != IPPROTO_IP)
					break;
			} else {
				/* Check for the two wild cases. */
				if (IPPROTO_IP == protocol) {
					protocol = answer->protocol;
					break;
				}
				if (IPPROTO_IP == answer->protocol)
					break;
			}
			......
		}
		......
		sock->ops = answer->ops;
		answer_prot = answer->prot;
		......
		sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot);
		......
		if (sk->sk_prot->init) {
			err = sk->sk_prot->init(sk);
		......
	}

处理过程如下：

1. 首先， 遍历inetsw数组中对应的socket type的链表，根据protocol匹配对应的 inet_protosw 结构体
2. 将对应的 inet_protosw 结构体的 inet_protosw.ops 和 inet_protosw.prot 赋值给 socket.ops 和 socket->sk.sk_prot
3. 最后， 调用 socket->sk->sk_prot->init()

####5.4 AF_INET socket的操作流程

前面的4.6节有分析过， socket 的xxx 操作， 最终会对应 socket->ops->xxx()， 那么， 在AF_INET中, socket->ops->xxx() 最终还是调用 socket->sk->sk_prot->xxx()来完成操作, 以tcp socket为例

	/* userspace */
	connect()

	/* */
	socket->ops->connect()	/* 即 inet_stream_connect() */
		__inet_stream_connect()
			sk->sk_prot->connect()	/* 即 tcp_v4_connect() */

####5.5 AF_INET 中的网络协议

linux中， AF_INET上支持多种网络协议， 除了前面提到过的 tcp， udp， icmp， raw socket之外， 还有如下的网络协议

	static struct inet_protosw dccp_v4_protosw = {
		.type		= SOCK_DCCP,
		.protocol	= IPPROTO_DCCP,	
		.prot		= &dccp_v4_prot,
		.ops		= &inet_dccp_ops,
		.no_check	= 0,
		.flags		= INET_PROTOSW_ICSK,
	};

	static struct inet_protosw l2tp_ip_protosw = {
		.type		= SOCK_DGRAM,
		.protocol	= IPPROTO_L2TP,
		.prot		= &l2tp_ip_prot,
		.ops		= &l2tp_ip_ops,
		.no_check	= 0,
	};

	static struct inet_protosw sctp_seqpacket_protosw = {
		.type       = SOCK_SEQPACKET,
		.protocol   = IPPROTO_SCTP,
		.prot       = &sctp_prot,
		.ops        = &inet_seqpacket_ops,
		.no_check   = 0,
		.flags      = SCTP_PROTOSW_FLAG
	};

	static struct inet_protosw sctp_stream_protosw = {
		.type       = SOCK_STREAM,
		.protocol   = IPPROTO_SCTP,
		.prot       = &sctp_prot,
		.ops        = &inet_seqpacket_ops,
		.no_check   = 0,
		.flags      = SCTP_PROTOSW_FLAG
	};

	static struct inet_protosw udplite4_protosw = {
		.type		=  SOCK_DGRAM,
		.protocol	=  IPPROTO_UDPLITE,
		.prot		=  &udplite_prot,
		.ops		=  &inet_dgram_ops,
		.no_check	=  0,		/* must checksum (RFC 3828) */
		.flags		=  INET_PROTOSW_PERMANENT,
	};
    
在“/proc/net/protocols”中可以读取到linux系统当前支持的网络协议的类型， 当然，包括所有的地址族， 而不只是 AF_INET
