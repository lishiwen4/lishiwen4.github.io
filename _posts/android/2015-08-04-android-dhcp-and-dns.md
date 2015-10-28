---
layout: post
title: "android dhcp and dns"
description:
category: android
tags: [android, network, wifi]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1.dhcp 和 dns

DHCP（Dynamic Host Configuration Protocol，动态主机配置协议）通常被应用在大型的局域网络环境中，主要作用是集中的管理、分配IP地址，使网络环境中的主机动态的获得IP地址、Gateway地址、DNS服务器地址等信息，并能够提升地址的使用率

**对于一个TCP/IP网络来说， 最重要的设定就是 ip地址， 子网掩码， 网关， 以及dns， dhcp协议正是为了自动完成这些配置**

RFC 2131中有对DHCP详细的描述。DHCP有3个端口，其中UDP67和UDP68为正常的DHCP服务端口，分别作为DHCP Server和DHCP Client的服务端口；546号端口用于DHCPv6 Client，而不用于DHCPv4，是为DHCP failover服务，这是需要特别开启的服务，DHCP failover是用来做“双机热备”的 

DNS（Domain Name System，域名系统），因特网上作为域名和IP地址相互映射的一个分布式数据库，能够使用户更方便的访问互联网，而不用去记住能够被机器直接读取的IP数串。通过主机名，最终得到该主机名对应的IP地址的过程叫做域名解析（或主机名解析）

DNS协议运行在UDP协议之上，使用端口号53。在RFC文档中RFC 2181对DNS有规范说明，RFC 2136对DNS的动态更新进行说明，RFC 2308对DNS查询的反向缓存进行说明

linux和windows下， 可以使用“nslookup”命令来解析域名

linux下还可以使用 “dig 完全域名 +trace” 的形式来查看域名解析的过程

###2. dhcpcd

兼容 RFC2131的DHCP客户端程序, 支持DHCP的全部功能且体积非常小， android上使用dhcpcd作为dhcp的客户端， 通常在shell中可以使用

	$ dhcpcd wlan0

的cmd来在wlan0上发起dhcp请求

###3. android 的dhcpcd server

android中为dhcpcd定义了几个与wifi相关的native的service

	service dhcpcd_wlan0 /system/bin/dhcpcd -ABKLG
		class late_start
		disabled
		oneshot

	service dhcpcd_p2p /system/bin/dhcpcd -ABKLG
		class late_start
		disabled
		oneshot

	service iprenew_wlan0 /system/bin/dhcpcd -n
		class late_start
		disabled
		oneshot

	service iprenew_p2p /system/bin/dhcpcd -n
		class late_start
		disabled
		oneshot

+ dhcpcd_wlan0	: 用于为wlan0请求dhcp， 分配地址
+ dhcpcd_p2p	: 用于为p2p请求dhcp， 分配地址
+ iprenew_wlan0	: 用于为wlan0请求更新dhcp， 获得的ip地址不变， 即组约延期
+ iprenew_p2p	: 用于为p2p请求更新dhcp， 获得的ip地址不变， 即组约延期

另外， 在”/system/etc/dhcpcd/dhcpcd.conf“中还保存了dhcpcd的config信息

###4. libnetutils 与dhcp api

android 的 libnetutils中提供了一套接口， 用于触发dhcpcd， 发起dhcp请求

	extern int do_dhcp(char *iname);

	extern int dhcp_do_request(const char *ifname,
                          char *ipaddr,
                          char *gateway,
                          uint32_t *prefixLength,
                          char *dns[],
                          char *server,
                          uint32_t *lease,
                          char *vendorInfo,
                          char *domain,
                          char *mtu);

	extern int dhcp_do_request_renew(const char *ifname,
                                char *ipaddr,
                                char *gateway,
                                uint32_t *prefixLength,
                                char *dns[],
                                char *server,
                                uint32_t *lease,
                                char *vendorInfo,
                                char *domain,
                                char *mtu);

	extern int dhcp_stop(const char *ifname);

	extern int dhcp_release_lease(const char *ifname);

	extern char *dhcp_get_errmsg();

+ do_dhcp()		: 
+ dhcp_do_request()	: 用于请求dhcp
+ dhcp_do_request_renew()	: 用于请求dhcp组约延期
+ dhcp_stop()		: 用于请求停止当前的dhcp过程
+ dhcp_release_lease()	: 用于请求dhcp服务器收回为该客户端分配的ip地址
+ dhcp_get_errmsg()	: 用于获取dhcp的错误信息

###5. dhcpcd 客户端与dhcpcd的通信

dhcpcd 通过属性与外界交换信息， 其执行过程如下：

####5.1 dhcp_do_request()

以wlan0接口的dhcp 处理的过程为例

1. 从 “ro.hostname” 属性中获取机器的hostname， hostname由ConnectivityService初始化时生成
2. 将属性 “dhcp.wlan0.result” 置为空字符串
3. 设置 “ctl.start” 属性， 例如， 内容为“dhcpcd_wlan0:-f /system/etc/dhcpcd/dhcpcd.conf -h android-dca7abfd017574a7 wlan0” (其中hostname在每台机器上都不相同)
4. 在10秒种内， 每隔200ms检查一次“init.svc.dhcpcd_wlan0”属性的值是否为running， 超时则认为dhcpc启动fail， 停止后续步骤
5. 在30秒内， 每隔200ms检查一次“dhcp.wlan0.result”属性是否被设置， 超时后则认为dhcp fail， 停止后续步骤
6. 检查“dhcp.wlan0.result”属性值(属性值可为 ok， failed， released中的一种)，若为ok则代表dhcp成功， 否则为失败， 停止后续步骤
7. 从系统属性中获取dhcp的结果,并且返回， 具体过程后续小节有详细说明

对于p2p接口， 将上述过程中的 wlan0 替换为 p2p

####5.2 dhcp_do_request_renew()

以wlan0接口的dhcp 处理的过程为例

1. 将属性 “dhcp.wlan0.result” 置为空字符串
2. 设置 “ctl.start” 属性， 例如， 内容为“iprenew_wlan0:wlan0”
3. 在30秒内， 每隔200ms检查一次“dhcp.wlan0.result”属性是否被设置， 超时后则认为dhcp fail， 停止后续步骤
4. 检查“dhcp.wlan0.result”属性值(属性值可为 ok， failed， released中的一种)，若为ok则代表dhcp成功， 否则为失败， 停止后续步骤
5. 从系统属性中获取dhcp的结果,并且返回， 具体过程后续小节有详细说明

对于p2p接口， 将上述过程中的 wlan0 替换为 p2p

####5.3 从系统属性获取dhcp的结果

根据dhcp的结果， dhcpcd会设置如下的属性(不是所有属性都有值有些为空)

+ dhcp.wlan0.dns1		: dns server 地址
+ dhcp.wlan0.dns2 		: dns server 地址, 通常为空
+ dhcp.wlan0.dns3		: dns server 地址, 通常为空
+ dhcp.wlan0.dns4 		: dns server 地址, 通常为空
+ dhcp.wlan0.domain	:
+ dhcp.wlan0.gateway	: 网关地址， 通常是无线路由自身的ip
+ dhcp.wlan0.ipaddress	: dhcp分配的ip地址
+ dhcp.wlan0.leasetime	: dhcp租期
+ dhcp.wlan0.mask		: 子网掩码
+ dhcp.wlan0.mtu		: mtu
+ dhcp.wlan0.server	: dhcp server地址
+ dhcp.wlan0.vendorinfo	:

另外还有一些与dhcpcd相关的属性：

+ dhcp.wlan0.pid		: 
+ dhcp.wlan0.result	: dhcp的结果"ok, failed, released"
+ dhcp.wlan0.reason	: 保存dhcpcd的状态， dhcpcd一共有7个状态“REBOOTING , INIT , SELECTING , REQUESTING , BOUND , REBINDING , RENEWING”

**需要注意的是， dhcpcd会与kernel建立 rtnetlink 连接， 当成功分配到ip之后， 会自动发送netlink消息， 通过RTM_NEWADDR属性来给对应的网络接口设置ip地址， 无需上层来设置ip地址**

####5.4 dhcp_stop()

dhcp_stop()的处理过程很简单， 直接通过“ctl.stop”属性来停止相关的service， 然后检查相关service的状态是否为”stopped“，并且将“dhcp.wlan0.result”值设置为“failed”, 与dhcpcd相关的几个service有

+ dhcpcd_wlan0	: 可通过“init.svc.dhcpcd_wlan0” 属性来检查该service的状态("running" "stopped")
+ dhcpcd_p2p	: 可通过“init.svc.dhcpcd_p2p” 属性来检查该service的状态("running" "stopped")
+ iprenew_wlan0	: 可通过“init.svc.iprenew_wlan0” 属性来检查该service的状态("running" "stopped")
+ iprenew_p2p	: 可通过“init.svc.iprenew_p2p” 属性来检查该service的状态“("running" "stopped")

####5.5 dhcp_release_lease()

dhcp_release_lease()与dhcp_stop()的处理过程相同， 只是比后者少了将“dhcp.wlan0.result”值设置为“failed”的这一步

###6. dhcp信息中网关和dns信息的来源

TCP/IP网络做中要的配置就是 ip地址， 子网掩码， 网关， 以及dns

**网关是指连接不同网段的网络， 进行互通的设备的ip， 一般路由通过wan口连接外网时，路由自身就是一个网关， 而交换机则不是**  

以普通家庭路由器为例, 在默认设置的情况下：

+ ip 地址范围 ： 与路由器的lan口同一网段， 通常为 192.168.1.xxx
+ 子网掩码    ： 通常为 255.255.255
+ 网关       ： 路由器一般兼具网关的功能， 当通过wan口连接外网时，网关默认为路由器LAN口的ip， 例如192.168.1.1
+ dns	    ： 默认使用通关wan口获取到的dns(由isp提供)

当然， 大部分家用路由器都可以自己指定路由器上dhcp服务器的设定， 例如

+ ip 地址范围 ： 与路由器的lan口的网段一致即可
+ 子网掩码    ： 自行指定， 注意和ip 地址范围一致即可
+ 网管       ： 根据实际情况指定， 例如， 若只使用路由器的交换功能， 使用lan口上的某个设备作为网关， 则应该设置该设备的ip作为网关， 而不是使用路由的lan口ip
+ dns	   ： 根据需要指定

另外， 大部分家用路由器都可以为特定MAC静态分配ip地址

###7. dhcpcd 配置静态ip和dns

dhcpcd可以使用“-f” 选项来指定一个配置文件， android上， 路径一般为“/system/etc/dhcpcd/dhcpcd.conf”， 可以在配置文件中指定dhcpcd使用静态ip

	interface wlan0  
	static ip_address=x.x.x.x  
	static subnet_mask=x.x.x.x  
	static router=x.x.x.x  
	static domain_name_servers=x.x.x.x 

###8. DhcpStateMachine

wifi 的dhcp请求由DhcpStateMachine来完成， DhcpStateMachine定义了如下的5个状态

+ DefaultState		: 其它状态的直接父状态
+ StoppedState		: 停止状态
+ WaitBeforeStartState	: 在开始dhcp start请求之前， 用于执行一些动作， 例如， wifi在dhcp之前， 可能需要停止scan， 调整bt-cox等
+ RunningState		: 执行dhcp start请求或者dhcp renew请求
+ WaitBeforeRenewalState	: 在开始dhcp renew请求之前， 用于执行一些动作， 例如， wifi在dhcp之前， 可能需要停止scan， 调整bt-cox等

另外， DhcpStateMachine 中会设置定时器， 时间取为dhcp租期的48%， 到期后， 发起dhcp renew request， 进行dhcp续约

	private boolean runDhcp(DhcpAction dhcpAction) {
		......
		mAlarmManager.setExact(AlarmManager.ELAPSED_REALTIME_WAKEUP,
			SystemClock.elapsedRealtime() +
			leaseDuration * 480, //in milliseconds
			mDhcpRenewalIntent);
		......
	}

另外， 需要注意的是， 在dhcp renew request阶段，DhcpStateMachine还会请求一个wakelock 40s， 防止系统休眠

####8.1 DhcpStateMachine 的接口

DhcpStateMachine 通过msg与 WifiStateMachine 或者 WifiP2pService 来通信

DhcpStateMachine接收如下的消息：

+ CMD_START_DHCP			: 开始dhcp start请求
+ CMD_STOP_DHCP			: 停止dhcp过程
+ CMD_RENEW_DHCP			: 开始dhcp renew 请求
+ CMD_PRE_DHCP_ACTION_COMPLETE	: 通知DhcpStateMachine， pre Action 执行完成

DhcpStateMachine会向WifiStateMachine 或者 WifiP2pService发送如下的消息：

+ CMD_PRE_DHCP_ACTION	： 通知外部， 可以开始执行pre Action
+ CMD_POST_DHCP_ACTION	： 通知外部， dhcp的结果， 可以携带 DHCP_SUCCESS 或者 DHCP_FAILURE 来表示成功或者失败， 成功时， 还会携带dhcp result

####8.2 DhcpStateMachine 与 libnetutils

DhcpStateMachine 通过 jni 来调用 libnetutils 中提供的dhcp api，完成dhcp请求, "networkUtils.java"和“android_net_NetUtils.java“ 中定义了jni部分

**CMD_START_DHCP的处理：**

	DhcpStateMachine.StoppedState.processMessage( CMD_START_DHCP )
		DhcpStateMachine.runDhcp( DhcpAction.START ) 
			NetworkUtils.runDhcp()
				android_net_utils_runDhcp()
					android_net_utils_runDhcpCommon()
						dhcp_do_request()		/* libnetutils api */

**CMD_RENEW_DHCP的处理：**

	DhcpStateMachine.StoppedState.processMessage( CMD_START_DHCP )
		DhcpStateMachine.runDhcp( DhcpAction.START ) 
			NetworkUtils.runDhcpRenew()
				android_net_utils_runDhcpRenew()
					android_net_utils_runDhcpCommon()
						dhcp_do_request_renew()		/* libnetutils api */

**CMD_STOP_DHCP的处理：**
		
	DhcpStateMachine.RunningState.processMessage( CMD_STOP_DHCP )
	DhcpStateMachine.WaitBeforeRenewalState.processMessage( CMD_STOP_DHCP )
		NetworkUtils.stopDhcp()
			android_net_utils_stopDhcp()
				dhcp_stop()					/* libnetutils api */

DhcpStateMachine.runDhcp() 在处理完dhcp请求后， 会判断执行的结果， 然后向WifiStateMachine发送CMD_POST_DHCP_ACTION消息来通告结果

###9. WifiStateMachine 处理 dhcp

WifiStateMachine 中存在一个 ObtainingIpState， 在进入该状态后， 若当前网络设置为使用dhcp方式(添加wifi网络时在高级选项中设置)， 则会在进入 ObtainingIpState 状态后， 发起dhcp请求， 对于roming的情况， 发起dhcp renew请求， 否则， 发起dhcp start请求

	void startDhcp() {
		if (mDhcpStateMachine == null) {
			mDhcpStateMachine = DhcpStateMachine.makeDhcpStateMachine(
			mContext, WifiStateMachine.this, mInterfaceName);

		}
		mDhcpStateMachine.registerForPreDhcpNotification();
		mDhcpStateMachine.sendMessage(DhcpStateMachine.CMD_START_DHCP);
	}

	void renewDhcp() {
		if (mDhcpStateMachine == null) {
			mDhcpStateMachine = DhcpStateMachine.makeDhcpStateMachine(
			mContext, WifiStateMachine.this, mInterfaceName);

		}
		mDhcpStateMachine.registerForPreDhcpNotification();
		mDhcpStateMachine.sendMessage(DhcpStateMachine.CMD_RENEW_DHCP);
	}

事实上， 如同上一节所讲， 都是向DhcpStateMachine 发送 CMD_START_DHCP 和 CMD_RENEW_DHCP 消息来发起dhcp request

DhcpStateMachine在处理完dhcp请求之后， 会向WifiStateMachine 发送 DhcpStateMachine.CMD_POST_DHCP_ACTION 消息, 通知dhcp的结果， 若成功， WifiStateMachine会调用handleIPv4Success()来处理

dhcp的结果包含 ip地址， dns， 网关， mtu， dhcp租期等信息， 我们重点关注前3个信息

	WifiStateMachine.handleIPv4Success()
		WifiStateMachine.updateLinkProperties()
			ConnectivityService.updateLinkProperties()
				ConnectivityService.updateRoutes()				/*更新路由表*/
					NetworkManagementService.addRoute()
				ConnectivityService.updateDnses()
					NetworkManagementService.setDnsServersForNetwork()	/* 设置netd的dns信息 */
					ConnectivityService.setDefaultDnsSystemProperties()	/* 设置系统属性 net.dns1, net.dns2 */
					NetworkManagementService.flushNetworkDnsCache		/* 刷新dns缓存 */

**dhcp获取到的ip地址， 会由dhcpcd通过与kernel的rtnetlink连接，设置对应网络接口的ip地址， 因此， 无需上层来设置ip地址**

属性“net.dns1” 和 “net.dns2”可用于查看系统中的dns， 但是netd在解析域名时， 不会去读取这两个属性, dns server的信息会保存在netd中
		
###10. WifiStateMachine 处理静态ip和dns

与 dhcpcd 配置静态ip和dns的方式不同， android wifi settings 配置静态ip和dns后， 使用指定的ip和dns， 不再通过dhcpcd来获取ip和dns

在添加android settings中添加wifi 网络时， 勾选“高级选项”后， 在“IP设置” 一栏下拉， 手动选择“静态”ip的方式， 然后手动填写如下信息：

+ IP地址
+ 网关地址
+ 子网掩码长度
+ DNS server 1
+ DNS server 2

如上的信息会被保存到“/data/misc/wifi/ipconfig.txt”中

在WifiConfigureStore.loadConfiguredNetworks() 中每次读取“/data/misc/wifi/wpa_supplicant.conf”中保存的network项，生成对应的WifiConfiguration对象时， 还会读取“/data/misc/wifi/ipconfig.txt”中的内容， 解析后，dns server的信息被保存在IpConfiguration.staticIpConfiguration.dnsServers 列表中， 最后被设置到对应的WifiConfiguration.mIpConfiguration 中去

在WifiStatemachine中， 对静态ip的处理过程如下

	WifiStateMachine. ObtainingIpState.enter()

		StaticIpConfiguration config = mWifiConfigStore.getStaticIpConfiguration(mLastNetworkId);
		......
		mNwService.setInterfaceConfig(mInterfaceName, ifcg);
		DhcpResults dhcpResults = new DhcpResults(config);
		sendMessage(CMD_STATIC_IP_SUCCESS, dhcpResults);
		......
	}

+ 首先根据静态配置的内容， 设置对应网络接口的ip地址
+ 然后根据静态配置的内容， 构造一个dhcp result， 发送CMD_STATIC_IP_SUCCESS消息， 触发handleIPv4Success()运行，设置route和dns

先看设置ip地址的过程

	NetworkManagementService.setInterfaceConfig()
			
	public void setInterfaceConfig(String iface, InterfaceConfiguration cfg) {
		......
		LinkAddress linkAddr = cfg.getLinkAddress();
		......
		final Command cmd = new Command("interface", "setcfg", iface,
			linkAddr.getAddress().getHostAddress(),
			linkAddr.getPrefixLength());

			......
			mConnector.execute(cmd);
			......
    	}

而设置route和dns的过程， 与使用dhcp获取ip时的过程相同

	WifiStateMachine.handleIPv4Success()
		WifiStateMachine.updateLinkProperties()
			ConnectivityService.updateLinkProperties()
				ConnectivityService.updateRoutes()				/*更新路由表*/
					NetworkManagementService.addRoute()
				ConnectivityService.updateDnses()
					NetworkManagementService.setDnsServersForNetwork()	/* 设置netd的dns信息 */
					ConnectivityService.setDefaultDnsSystemProperties()	/* 设置系统属性 net.dns1, net.dns2 */
					NetworkManagementService.flushNetworkDnsCache
	
属性“net.dns1” 和 “net.dns2”可用于查看系统中的dns， 但是netd在解析域名时， 不会去读取这两个属性, dns server的信息会保存在netd中

###11. dns查询的api

####11.1 java api

java.net.InetAddress类是Java对IP地址的封装， 提供了静态方法做dns查询

	static InetAddress[] getAllByName(String host)
	static InetAddress getByAddress(byte[] addr)
	static InetAddress getByAddress(String host,byte[] addr)
	static InetAddress getByName(String host)
	static InetAddress getLocalHost()

在这些静态方法中，最为常用的应该是getByName(String host)方法，只需要传入目标主机的名字，InetAddress会尝试做连接DNS服务器，并且获取IP地址的操作

java.net.InetAddress的实现， 都调用了下面所介绍的 bionic libc 中的 api


####11.2 bionic libc api

	int getaddrinfo(const char *hostname, const char *servname, const struct addrinfo *hints, struct addrinfo **res)
	int getnameinfo(const struct sockaddr* sa, socklen_t salen, char* host, size_t hostlen, char* serv, size_t servlen, int flags)
	struct hostent *gethostbyname(const char *name)

####11.3 getaddrinfo()的执行流程

	getaddrinfo()
		android_getaddrinfofornet()	
			android_getaddrinfo_proxy()

	static int android_getaddrinfo_proxy(
    		const char *hostname, const char *servname,
    		const struct addrinfo *hints, struct addrinfo **res, unsigned netid)
	{
		......
		sock = socket(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0);
		......
    
    		proxy_addr.sun_family = AF_UNIX;
    		strlcpy(proxy_addr.sun_path, "/dev/socket/dnsproxyd",
		......

    		// Send the request.
    		proxy = fdopen(sock, "r+");
    		if (fprintf(proxy, "getaddrinfo %s %s %d %d %d %d %u",
           		hostname == NULL ? "^" : hostname,
            		servname == NULL ? "^" : servname,
            		hints == NULL ? -1 : hints->ai_flags,
            		hints == NULL ? -1 : hints->ai_family,
            		hints == NULL ? -1 : hints->ai_socktype,
            		hints == NULL ? -1 : hints->ai_protocol,
            		netid) < 0) {
        			goto exit;
    		}

		......
	}

再来看netd对于“getaddr”消息的处理

	DnsProxyListener::GetAddrInfoHandler::run()
		android_getaddrinfofornet()
			explore_fqdn()

事实上， 在应用程序中和netd中都会调用到 android_getaddrinfofornet()， 但是netd中会设置“ANDROID_DNS_MODE”环境变量值为“local”， 而普通的应用程序则不会这么做， 因此， 两者在android_getaddrinfofornet()中的执行路径不同
				
####11.4 getnameinfo()的执行流程

	getnameinfo()
		android_getnameinfofornet()
			/* getnameinfo_local()	*/
			getnameinfo_inet()	
				android_gethostbyaddrfornet_proxy()
					/* 打开“/dev/socket/dnsproxyd” 向netd发送 “gethostbyaddr” 消息*/

	DnsProxyListener::GetHostByAddrHandler::run()
		android_gethostbyaddrfornet()
			android_gethostbyaddrfornet_real()

同样的， 在 android_gethostbyaddrfornet() 中， 由于netd设置了“ANDROID_DNS_MODE”环境变量， 因此， 调用的是 android_gethostbyaddrfornet_real()

		
####11.5 gethostbyname()的执行流程

	gethostbyname()
		gethostbyname_internal()
			gethostbyname_internal()
				/* 普通进程， 通过“/dev/socket/dnsproxyd” 向netd发送 “gethostbyname” 消息 */

	DnsProxyListener::GetHostByNameHandler::run()
		android_gethostbynamefornet()
			gethostbyname_internal()
				gethostbyname_internal_real()

###12. dhcp的过程

dhcp 协议中的7种报文

1. dhcp discover	: 此为client开始DHCP过程中的第一个请求报文
2. dhcp offer	: 此为server 对dhcp discover 报文的响应
3. dhcp requst	: 此为client 对dihcp offer 报文的响应
4. dhcp declient	: 当client发现server分配给它的IP地址无法使用，如 IP地址发生冲突时，将发出此报文让server禁止使用这次分配的IP地址。
5. dhcp ack	: server对 dhcp requst 报文的同意响应，client收到此报文后才真正获得了IP地址和相关配置信息。
6. dhcp nak	: 此报文是server对client的dhcp requst报文的拒绝响应，client 收到此报文后，一般会重新开始DHCP过程。
7. dhcp release	: 此报文是 client主动释放IP地址，当server 收到此报文后就可以收回IP地址分配给其他的client.

dhcp的过程：

1. 发现阶段 ： 即DHCP客户端寻找DHCP服务器的阶段。因为DHCP服务器的IP地址对于客户端来说是未知的，所以DHCP客户端以广播方式发送DHCP Discover报文来寻找DHCP服务器，只有DHCP Server才会进行响应
2. 提供阶段 ： 即DHCP服务器提供IP地址的阶段。DHCP Server接收到Client的DHCP Discover报文后，从IP地址池中挑选一个尚未分配的IP地址分配给客户端，向该客户端发送包含出租IP地址和其它设置的DHCP Offer报文
3. 选择阶段 ： 即DHCP Client选择IP地址的阶段。如果有多台DHCP Server向该客户端发来DHCP Offer报文，客户端只接受第一个收到的DHCP Offer报文，然后以广播方式向各DHCP服务器回应DHCP Request报文，该信息中包含向所选定的DHCP服务器请求IP地址的内容
4. 确认阶段 ： 即DHCP服务器确认所提供IP地址的阶段。当DHCP服务器收到DHCP客户端回答的DHCP Request报文后，判断Option字段中的DHCP Server的IP地址是否与自己的相同
   + 如果不相同，则不作任何处理
   + 否则，DHCP Server会向客户端发送包含它所提供的IP地址和其它设置的DHCP ACK确认报文。DHCP Client收到DHCP ACK报文后，检查DHCP Server分配给自己的IP地址是否能够使用，比如在以太网络中，DHCP Client会发免费的ARP请求来确定IP地址是否已经被其他客户端使用。
      + 如果IP地址已经被其他客户端使用，则该DHCP Client会发DHCP Decline报文通知DHCP Server禁用这个IP地址以免引起冲突；
      + 否则，该DHCP Client成功获取IP地址
5. 更新租约，DHCP服务器向DHCP客户端出租的IP地址都有一个租界期限，期满后DHCP服务器便会回收出租的IP地址。如果DHCP客户端要延长其IP租约，须更新其IP租约。DHCP客户端在IP租约期限过一半时，会自动向DHCP服务器发送单播的DHCP Request报文续延租期
   + DHCP服务器收到DHCP Request续租报文后，如果此IP地址无法再分配给该DHCP客户端时，DHCP服务器给DHCP客户端回应DHCP NAK报文
   + DHCP服务器收到DHCP Request续租报文后，如果此IP地址可以继续分配给该DHCP客户端，DHCP服务器给DHCP客户端回应DHCP ACK报文
   + DHCP客户端收到DHCP ACK报文后，租期相应延长
   + DHCP客户端如果收到DHCP NAK报文，则客户端不能继续使用这个IP地址， 可以重新广播发送DHCP Discover发现信息来请求新的IP地址
   + DHCP客户端如果没有收到DHCP ACK，也没有收到DHCP NAK报文，则客户端可以继续使用这个IP地址，直到在使用租期过去7/8时，再次向DHCP服务器发送广播的DHCP Request报文

DHCP客户端在成功获取IP地址后，随时可以通过发送DHCP Release报文释放自己的IP地址，DHCP服务器收到DHCP Release报文后，会回收相应的IP地址重新分配

