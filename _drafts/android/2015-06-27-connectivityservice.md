---
layout: post
title: "Connectivity Sevice"
description:
category: android
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. connectivityservice


####1.1 NetworkCapabilities

NetworkCapabilities 用于描述一个网络的capability, transport type, 以及bandwidth

网络的capability存储在mNetworkCapabilities成员中， 类型如下：

0. NET_CAPABILITY_MMS
1. NET_CAPABILITY_SUPL
2. NET_CAPABILITY_DUN
3. NET_CAPABILITY_FOTA
4. NET_CAPABILITY_IMS
5. NET_CAPABILITY_CBS
6. NET_CAPABILITY_WIFI_P2P
7. NET_CAPABILITY_IA
8. NET_CAPABILITY_RCS
9. NET_CAPABILITY_XCAP
10. NET_CAPABILITY_EIMS
11. NET_CAPABILITY_NOT_METERED
12. NET_CAPABILITY_INTERNET
13. NET_CAPABILITY_NOT_RESTRICTED
14. NET_CAPABILITY_TRUSTED
15. NET_CAPABILITY_NOT_VPN

一个NetworkCapabilities对象初始化时会被赋予 NET_CAPABILITY_NOT_RESTRICTED， NET_CAPABILITY_TRUSTED， NET_CAPABILITY_NOT_VPN
并且提供如下的API来access其capability信息：

+ addCapability()
+ removeCapability()
+ getCapabilities()
+ hasCapability()
+ enumerateBits()
+ satisfiedByNetCapabilities()
+ equalsNetCapabilities()

网络的transport type, 存储在mTransportTypes成员中， 类型如下：

0. TRANSPORT_CELLULAR
1. TRANSPORT_WIFI
2. TRANSPORT_BLUETOOTH
3. TRANSPORT_ETHERNET
4. TRANSPORT_VPN

相关的api:

+ addTransportType()
+ removeTransportType()
+ getTransportTypes()
+ hasTransport()
+ combineTransportTypes()
+ satisfiedByTransportTypes()
+ equalsTransportTypes()

link up bandwidth 和 link down bandwidth， 存储在 mLinkUpBandwidthKbps 和 mLinkDownBandwidthKbps 成员中

+ setLinkUpstreamBandwidthKbps()
+ getLinkUpstreamBandwidthKbps()
+ setLinkDownstreamBandwidthKbps()
+ getLinkDownstreamBandwidthKbps()
+ combineLinkBandwidths()
+ satisfiedByLinkBandwidths()
+ equalsLinkBandwidths()

####1.2 NetworkRequest 和 NetworkRequestInfo

NetworkRequest用于发起一个network的请求， 其构造参数为：

	public NetworkRequest(NetworkCapabilities nc, int legacyType, int rId)

legacyType 为网络的类型，例如 TYPE_WIFI， TYPE_BLUETOOTH等， rId则为request id， 从1开始， 依次递增
其networkCapabilities成员保存了一个NetworkCapabilities对象

NetworkRequestInfo 继承自 IBinder.DeathRecipient 其主要的目的是通过binder来监听networkrequester(需要使用connectivitymanager的api通过binder来调用connectivityservice的requestNetwork()来请求网络)， 当requester死亡后， 完成后续的清理工作, 并未存储其它的信息

connectivityservice提供了requestNetwork接口， 用于请求一个network'

	public NetworkRequest requestNetwork(NetworkCapabilities networkCapabilities,
            Messenger messenger, int timeoutMs, IBinder binder, int legacyType) 

在其中会构造一个NetworkRequest对象和一个NetworkRequestInfo对象， 触发handleRegisterNetworkRequest()来处理 network request

####1.3 NetworkAgent 与 WifiNetworkAgent

ConnectivityService管理系统中的数据连接， android中有多种网络可以提供数据连接， 例如wifi， mobile， 蓝牙， 每一种网络连接有自己单独的service来管理， 例如wifi的WifiService， ConnectivityService与这些sevice之间的交互使用NetworkAgent来实现， 例如， 在WifiService中就使用了一个 WifiNetworkAgent

NetworkAgent与ConnectivityService之间使用AsyncChannel来通信， ConnectivityService通过ConnectivityManager提供一个api用于ConnectivityService端连接到一个NetworkAgent

	public void registerNetworkAgent(Messenger messenger, NetworkInfo ni, LinkProperties lp,
            NetworkCapabilities nc, int score, NetworkMisc misc) 

在通过AsyncChannel来连接时， ConnectivityService作为client端， NetworkAgent作为service端， NetworkAgent的构造方法中会调用该API，和ConnectivityService建立连接

通过AsyncChannel传递给ConnectivityService的message， 在ConnectivityService中有一个内部类NetworkStateTrackerHandler的对象mTrackerHandler， 由其来处理这些message， 
NetworkAgent提供了一些api用于向ConnectivityService发送消息：

	public void removeUidRanges(UidRange[] ranges)；
	public void sendLinkProperties(LinkProperties linkProperties)；
	public void sendNetworkCapabilities(NetworkCapabilities networkCapabilities)；
	public void sendNetworkInfo(NetworkInfo networkInfo)；
	public void sendNetworkScore(int score)；
	
NetworkAgent有一个抽象方法， unwanted(), 当ConnectivityService不需要使用该网络时， 会调用AsyncChannel.disconnect()来断开连接， 会触发NetworkAgent.unwanted()， NetworkAgent的子类需要实现该方法

ConnectivityService可以向NetworkAgent发送NetworkAgent.CMD_REPORT_NETWORK_STATUS消息，通知网络状态的变化(VALID_NETWORK或者INVALID_NETWORK) 触发NetworkAgent.networkStatus()， 该方法为一个空方法， 其子类可以根据需要来重写该方法

WifiNetworkAgent继承自NetworkAgent并且实现了其unwanted()方法， 重写了networkStatus()方法

####  wifi的网络评分


wifi的网络评分通过调用WifiNetworkAgent.sendNetworkScore()来通知ConnectivityService，WifiNetworkAgent.sendNetworkScore()的调用时机如下：

1. WifiStateMachine中接收到“prop_state_change”广播后， 调用handleStateChange()来处理， 其中若wifi网络状态差， 则评分为1， 否则，评分为wifi基础评分60分， 并将调用WifiNetworkAgent.sendNetworkScore()将此评分发送给ConnectivityService
2. 在WifiStateMachine中执行calculateWifiScore()计算wifi网络评分后，若评分有更新， 则调用WifiNetworkAgent.sendNetworkScore()来通知ConnectivityService新的网络评分，计算wifi评分的时机如下：
   1. 在WifiStateMachine进入L2ConnectedState状态后， 会发送一次CMD_RSSI_POLL消息， 触发rssi poll
   2. 在WifiStateMachine处理CMD_ENABLE_RSSI_POLL消息时， 会发送一次CMD_RSSI_POLL消息， 触发rssi poll
   3. 在WifiStateMachine处理CMD_RSSI_POLL消息时， 会获取wifi的 link layer state， 然后调用calculateWifiScore()计算wifi网络评分， 然后延时3秒再次发送CMD_RSSI_POLL， 循环进行

wifi的网络评分依据rssi， linkspeed， 

#### wifi的LinkProperties

LinkProperties 包含如下信息：

+ Interface name: 初始化时指定
+ IPv4 and IPv6 addresses: 通过NetlinkTracker从netlink中获取， WifiStateMachine中的mNetlinkTracker对象
+ IPv4 routes, DNS servers, and domains:通过DHCP信息来获取.
+ IPv6 routes and DNS servers: 通过NetlinkTracker从netlink中获取， WifiStateMachine中的mNetlinkTracker对象.
+ HTTP proxy: 通过WifiConfigureStore获取.

WifiStateMachine中在如下场景会调用updateLinkProperties()

1. L2ConnectedState中处理“DhcpStateMachine.CMD_POST_DHCP_ACTION”消息成功后，会调用handleIPv4Success()， 其中会调用updateLinkProperties()
2. L2ConnectedState中处理“DhcpStateMachine.CMD_POST_DHCP_ACTION”消息失败后，会调用handleIPv4Failure()， 其中会调用updateLinkProperties()
3. ObtainingIpState中处理"CMD_STATIC_IP_SUCCESS"消息时，会调用handleIPv4Success()，  其中会调用updateLinkProperties()
4. ObtainingIpState中处理"CMD_STATIC_IP_FAILURE"消息时，会调用handleIPv4Failure()，  其中会调用updateLinkProperties()
5. WifiStateMachine初始化时， 会发送一次“CMD_UPDATE_LINKPROPERTIES”消息， 在处理该消息时， 会调用updateLinkProperties()
6. ConnectModeState中处理“WifiStateMachine.CMD_AUTO_SAVE_NETWORK”消息时， 可能会调用会调用updateLinkProperties()
7. WifiStateMachine.handleNetworkDisconnect()中会调用clearLinkProperties()




#### NetworkFactory 与 WifiNetworkFactory


#### NetlinkTracker
NetworkManagementService.NetdCallbackReceiver.onEvent()
















###1. android L的multi-networking API、

android 5.0 提供了multi-networking API，  app 可以在指定网络特征( 指网络所提供的能力 )前提下，动态地扫描可用的网络，并且与它们建立连接。当你的 app 需要一个特殊的网络时，这个特性是相当有用的，比如 SUPL，MMS，计费网络，以及你想要通过特殊的传输协议发送数据

如果在app中需要动态地选择和链接一个网络， 步骤如下：

1. 构建一个 ConnectivityManager。
2. 使用 NetWorkRequest.Builder 类来构建一个 NetworkRequest 对象，并指定网络的特性以及 app 需要的传输类型
3. 如果你只想接收某个合适网络的扫描结果的通知而不是主动去切换， 调用registerNetworkCallback( )，并传入一个 NetworkRequest 对象以及实现一个 ConnectivityManager.NetworkCallback， 当扫描到合适的网络后， 注册的ConnectivityManager.NetworkCallback中的相应的方法会被回调
4. 如果你想要扫描某个合适网络并且希望主动切换过去，调用 requestNetwork( )， 并传入一个 NetworkRequest 对象以及实现一个 ConnectivityManager.NetworkCallback， 当扫描到合适的网络以后， 会主动切换过去， 并且注册的ConnectivityManager.NetworkCallback中的相应的方法会被回调

相关的class如下：

####1.1 android.net.NetworkCapabilities

NetworkCapabilities.mNetworkCapabilities 成员保存了capability, 可为以下值的位或

	public static final int NET_CAPABILITY_MMS            = 0;
	public static final int NET_CAPABILITY_SUPL           = 1;
	public static final int NET_CAPABILITY_DUN            = 2;
	public static final int NET_CAPABILITY_FOTA           = 3;
	public static final int NET_CAPABILITY_IMS            = 4;
	public static final int NET_CAPABILITY_CBS            = 5;
	public static final int NET_CAPABILITY_WIFI_P2P       = 6;
	public static final int NET_CAPABILITY_IA             = 7;
	public static final int NET_CAPABILITY_RCS            = 8;
	public static final int NET_CAPABILITY_XCAP           = 9;
	public static final int NET_CAPABILITY_EIMS           = 10;
	public static final int NET_CAPABILITY_NOT_METERED    = 11;
	public static final int NET_CAPABILITY_INTERNET       = 12;
	public static final int NET_CAPABILITY_NOT_RESTRICTED = 13;
	public static final int NET_CAPABILITY_TRUSTED        = 14;
	public static final int NET_CAPABILITY_NOT_VPN        = 15;

NetworkCapabilities.mNetworkCapabilities的初始值为：

	private long mNetworkCapabilities = (1 << NET_CAPABILITY_NOT_RESTRICTED) |
            (1 << NET_CAPABILITY_TRUSTED) | (1 << NET_CAPABILITY_NOT_VPN);

capability相关的API如下：

	public NetworkCapabilities addCapability(int capability)
	public NetworkCapabilities removeCapability(int capability)
	public int[] getCapabilities()
	public boolean hasCapability(int capability) 

NetworkCapabilities.mTransportTypes 成员保存了传输类型， 可为以下值的位或：

	public static final int TRANSPORT_CELLULAR = 0;
    	public static final int TRANSPORT_WIFI = 1;
    	public static final int TRANSPORT_BLUETOOTH = 2;
    	public static final int TRANSPORT_ETHERNET = 3;
    	public static final int TRANSPORT_VPN = 4;

transport type相关的API如下：

	public NetworkCapabilities addTransportType(int transportType)
	public NetworkCapabilities removeTransportType(int transportType)
	public int[] getTransportTypes()
	public boolean hasTransport(int transportType)

NetworkCapabilities.mLinkUpBandwidthKbps 成员保存了上行带宽
NetworkCapabilities.mLinkDownBandwidthKbps 成员保存了下行带宽

bandwidth相关的API如下：

	public void setLinkUpstreamBandwidthKbps(int upKbps)
	public int getLinkUpstreamBandwidthKbps()
	public void setLinkDownstreamBandwidthKbps(int downKbps)
	public int getLinkDownstreamBandwidthKbps() 
	
NetworkCapabilities.mNetworkSpecifier成员保存了一些特殊要求(string类型)
相关的API如下：

	public void setNetworkSpecifier(String networkSpecifier)
	public String getNetworkSpecifier()

比较一个 NetworkCapability 是否能够满足另外一个 NetworkCapability， 实质上就是比较 capability， transport type， bandwidth， Specifier是否都 满足

	public boolean satisfiedByNetworkCapabilities(NetworkCapabilities nc)

比较两个 NetworkCapability 的要求是否相同， capability， transport type， bandwidth， Specifier是否都相同

	public boolean equals(Object obj)

NetworkCapability保存了对一个网络的能力和特性的期望

####1.1 android.net.NetworkRequest

NetworkRequest代表一个网络请求， 包含如下成员

	public final NetworkCapabilities networkCapabilities;
	public final int requestId;
	public final int legacyType;

requestId作为tokens代表一个NetworkRequest
legacyType的类型由ConnectivityManager来定义，如 TYPE_MOBILE， TYPE_WIFI等

只有framework能够使用NetworkRequest的构造函数来实例化NetworkRequest的对象， app应该使用builder来构造

相关的api如下：

	public Builder addCapability(int capability)
	public Builder removeCapability(int capability)
	public Builder addTransportType(int transportType)
	public Builder removeTransportType(int transportType)
	public Builder setLinkUpstreamBandwidthKbps(int upKbps)
	public Builder setLinkDownstreamBandwidthKbps(int downKbps)
	public Builder setNetworkSpecifier(String networkSpecifier)

####1.1 ConnectivityService.NetworkRequestInfo

ConnectivityService的内部类NetworkRequestInfo用于监听发起NetworkRequest的退出， app使用ConnectivityManager的api来注册NetworkRequest：

	public void registerNetworkCallback (NetworkRequest request, ConnectivityManager.NetworkCallback networkCallback)
	public void requestNetwork (NetworkRequest request, ConnectivityManager.NetworkCallback networkCallback)

请求者实质上是通过binder调用ConnectivityService的方法， ConnectivityService会为每一个NetworkRequest实例化一个 NetworkRequestInfo，保存对应的request和binder， 并且其内部会注册Binder的死亡监听， 当app退出时， 会将该app发起的 NetworkRequest 失效

另外 NetworkRequestInfo.isRequest 会保存对应的NetworkRequest是否是 REQUEST 类型(还可以是LISTEN类型)

app端也可以使用ConnectivityManager的api来使NetworkRequest失效

	void 	unregisterNetworkCallback(ConnectivityManager.NetworkCallback networkCallback)

####1.1  ConnectivityManager.NetworkCallback 

上面讲到过， 注册NetworkRequest的时候， 需要传递一个  ConnectivityManager.NetworkCallback 的子类， 在合适的network被发现后， 将会回调该类的相应方法，其回调方法如下：

	public void onPreCheck(Network network)
	public void onAvailable(Network network)
	public void onLosing(Network network, int maxMsToLive)
	public void onLost(Network network)
	public void onUnavailable()
	public void onCapabilitiesChanged(Network network, NetworkCapabilities networkCapabilities)
	public void onLinkPropertiesChanged(Network network, LinkProperties linkProperties)

networkCallback对象中保存了对应的networkRequest(由ConnectivityService分配， 并非app发起请求时提供的那一个networkRequest对象)

####1.1 request network的注册过程

ConnectivityManager提供的发起network请求的api：

	public void requestNetwork(NetworkRequest request, NetworkCallback networkCallback)
	public void requestNetwork(NetworkRequest request, NetworkCallback networkCallback, int timeoutMs)
	public void registerNetworkCallback(NetworkRequest request, NetworkCallback networkCallback)

注意requestNetwork的timeout版本， 当给定的timeout值到期后， 若仍没有发现合适的network， 则NetworkCallback.unavailable()会被回调， 并无其它的差别

这3个api都是直接调用ConnectivityManager.sendRequestForNetwork()来实现的

	private NetworkRequest sendRequestForNetwork(NetworkCapabilities need,
            NetworkCallback networkCallback, int timeoutSec, int action,
            int legacyType)

其参数分别为：

+ need ： 请求的network 的 capability
+ networkCallback ： 注册的回调
+ timeoutSec ： 超时值
+ action ： 对于requestNetwork() 为2(REQUEST)， 对于registerNetworkCallback()为1(LISTEN)
+ legacyType : 请求的网络类型， 从ConnectivityManager.TYPE_NONE到CnnectivityManager.TYPE_VPN可选，但是registerNetworkCallback()会固定使用TYPE_NONE


ConnectivityManager.sendRequestForNetwork() 中对于 LISTEN 和 REQUEST 这两种类型的 NetworkRequest， 分别调用 ConnectivityService.listenForNetwork() 和 ConnectivityService.requestNetwork() 来处理， 最后， 将NetworkRequest对象和其对应的NetworkCallback对象作为键值对， 存储在一个Hashap  ConnectivityManager.sNetworkCallback 中

先看 ConnectivityService.listenForNetwork()， 其过程比较简单：

1. 实例化一个NetworkRequest对象， network类型为TYPE_NONE
2. 实例化一个NetworkRequestInfo对象， 监听发起NetworkRequest的app的退出
3. 触发 ConnectivityService.handleRegisterNetworkRequest()进行处理

ConnectivityService.requestNetwork()的处理过程与ConnectivityService.listenForNetwork()基本相同,但是对于指定了timeout的NetworkRequest， 会延时发送EVENT_TIMEOUT_NETWORK_REQUEST消息， 但是当前版本中该cmd并不会被处理

再来看ConnectivityService.handleRegisterNetworkRequest()的处理过程：

1. 通过 ConnectivityService.mNetworkAgentInfos 来遍历所有注册的NetworkAgentInfo信息， 对于能够满足NetworkRequest的要求的NetworkAgentInfo， 选取score最高的Network作为bestNetwork， 若networkRequest类型为REQUEST， 则对bestnetwork调用NetworkAgentInfo.addRequest(), 并且发起NetworkCallback.onAvailable()回调， 在遍历的过程中可能会被回调好几次
2. 若该NetWorkRequest已经被处理过， 则忽略(ConnectivityService每实例化一个NetworkRequest时，都会分配一个ID作为token， 在处理该NetworkRequest后， 会记录其ID到ConnectivityService.mNetworkForRequestId中)
3. 若找到了best network， 并且request的类型为REQUEST，best network已经连接上， 则清除best network的 lingering， 通知其NetworkMonitor该network已经连接上
4. 若找到了best network， 则对bestnetwork调用NwteorkAgentInfo.addRequest()， 记录NetworkRequest的ID到ConnectivityService.mNetworkForRequestId中， 然后还会发起NetworkCallback.onAvailable回调, 对于网络类型不为TYPE_NONE， 请求类型为REQUEST的NetworkRequest， 还会添加到LegacyTypeTracker中去
5. 记录NetworkRequest到ConnectivityService.mNetworkRequests中去
6. 对于请求类型为REQUEST的NetworkRequest， 遍历所有注册的NetworkFactoryInfo， 向对应的NetworkFactory发送 CMD_REQUEST_NETWORK 请求


####1.1 NetworkCallback的调用的时机

1. 某个network对应的NetworkAgent发送EVENT_NETWORK_PROPERTIES_CHANGED后， 触发 onLinkPropertiesChanged()回调
2. 某个network对应的NetworkAgent和ConectivityService断开连接后， 触发onLost()回调
3. 当ConnectivityService调用updateLinkProperties()时， 会触发onLinkPropertiesChanged()回调
4. 当ConnectivityService调用updateCapabilities()时， 会触发onCapabilitiesChanged()回调
5. 当ConnectivityService调用rematchNetworkAndRequests()时触发onLosing()回调
6. 当ConnectivityService调用updateNetworkInfo()时会触发onPreCheck()回调

####1.1 NetworkRequest 的释放过程

NetworkRequest的释放由 Connectivity.unregisterNetworkCallback() 来完成， 实际上是调用 ConnectivityService.releaseNetworkRequest()来实现，执行的动作如下：

1. 取消对应的NetworkRequestInfo的死亡监听
2. 从ConnectivityService.mNetworkRequests中移除对应的NetworkRequest
3. 如果该NetworkRequest的类型为REQUEST(即非LISTEN), 则遍历所有的NetworkAgentInfo， 从所有满足该请求的Network的NetworkAgent的request列表中移除该request， 移除之后， 若该Network的request列表中不再存在REQUEST类型的NetworkRequest， 且该network也非VPN， 则断开对应的NetworkAgent的连接， 并触发其unwanted()回调， 对于wifi来说， 将导致wifi断连
4. 如果该NetworkRequest的类型为REQUEST(即非LISTEN), 则遍历所有的NetworkFactoryInfo， 发送CMD_CANCEL_REQUEST消息
5. 如果该NetworkRequest的类型为LISTEN(即非REQUEST), 则仅仅遍历所有的NetworkAgentInfo， 从所有满足该请求的Network的NetworkAgent的request列表中移除该request






####1.1  NetworkFactory

ConnectivityService管理系统中的数据连接， android中有多种网络可以提供数据连接， 例如wifi， mobile， 蓝牙， 每一种网络连接有自己单独的service来管理， 例如wifi的WifiService， ConnectivityService与这些service之间使用NetworkFactory进行交互(主要是ConnectivityService将NetworkRequest通知给对应的NetworkFactory并由其处理)

NetworkFactory存在于对应的network的service中， 与ConnectivityService之间通过AsyncChannel来通信

ConnectivityService可以通过发送“CMD_REQUEST_NETWORK”消息来通知NetworkFactory， 有新的NetworkRequest， 也可以使用NetworkFactory的API来通知有新的NetworkRequest：

	public void addNetworkRequest(NetworkRequest networkRequest, int score)

两个参数，一个是NetworkRequest，另一个当前任一一个能够满足该NetworkRequest的network的得分(如果某个NetworkFactory对应的network认为自己的得分不会超过该得分， 则该网络将不会去尝试连接)

NetworkFactory内部使用一个私有类NetworkRequestInfo来表示一个NetworkRequest

	public NetworkRequestInfo(NetworkRequest request, int score) {
        		this.request = request;
        		this.score = score;
        		this.requested = false;
        }

NetworkRequestInfo保存了对应的NetworkRequest以及添加该NetworkRequest时的得分， 以及该request是否被NetworkFactory满足， 当然， 无论是否满足，对于未被添加过的NetworkRequest， 所添加的NetworkRequest都会被保存在数组NetworkFactory.mNetworkRequests中, 对于已经添加过的NetworkRequest， 也会更新其score， 重新检查对应的network是否能满足该network request

在添加新的新的NetworkRequest时， NetworkFactory需要自己来检查是否能够满足其要求，NetworkFactory通过score和capability来过滤NetworkRequest， 并提供了两个API来进行设置

	public void setScoreFilter(int score)
	public void setCapabilityFilter(NetworkCapabilities netCap)

每次重新设置过滤条件后， 都会重新检查所有的NetworkRequest， 是否符合条件， 另外除了通过score和capability来过滤外， 还可以自定义过滤条件， 通过重写NetworkFactory的API来实现

	public boolean (NetworkRequest request, int score)
		return true;
    	}

检查是否能满足NetworkRequest的工作由NetworkFactory.evalRequest()完成，若设定的score过滤值大于NetworkRequest时传递的score，也能够满足capability的要求， 并且acceptRequest()也通过， 则**调用NetworkFactory.needNetworkFor()， 并设置对应的NetworkRequestInfo.requested为true**，否则**调用NetworkFactory.releaseNetworkFor()， 并设置对应的NetworkRequestInfo.requested为false** 

NetworkFactory.needNetworkFor()和NetworkFactory.releaseNetworkFor()的定义如下：

	protected void needNetworkFor(NetworkRequest networkRequest, int score) {
        		if (++mRefCount == 1) startNetwork();
    	}

    	protected void releaseNetworkFor(NetworkRequest networkRequest) {
        		if (--mRefCount == 0) stopNetwork();
    	}

startNetwork()和stopNetwork()需由子类去实现

最后ConnectivityService可以通过发送”CMD_CANCEL_REQUEST“消息来通知NetworkFactory， remove 一个NetworkRequest， 也可以使用NetworkFactory的API来remove一个NetworkRequest

	public void removeNetworkRequest(NetworkRequest networkRequest)

NetworkFactory在相应的network的service中实例化， 需要通过ConnectivityManager的api来连接ConnectivityService或者断开连接

	public void registerNetworkFactory(Messenger messenger, String name)
	public void unregisterNetworkFactory(Messenger messenger)

WifiNetworkFactory继承自NetworkFactory，并重写了startNetwork()和stopNetwork()， 但是目前为两个空方法

WifiStateMachine在接收到”android.intent.action.BOOT_COMPLETED“广播后， 会实例化WifiNetworkFactory对象并连接到ConnectivityService， 其capability的filter为

	mNetworkCapabilitiesFilter.addTransportType(NetworkCapabilities.TRANSPORT_WIFI);
	mNetworkCapabilitiesFilter.addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET);
	mNetworkCapabilitiesFilter.addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_RESTRICTED);
	mNetworkCapabilitiesFilter.setLinkUpstreamBandwidthKbps(1024 * 1024);
	mNetworkCapabilitiesFilter.setLinkDownstreamBandwidthKbps(1024 * 1024);

score filter为 60 分

####1.1 ConnectivityService.NetworkFactoryInfo

ConnectivityService.NetworkFactoryInfo用于保存一个和ConnectivityService连接的NetworkFactory的信息

	private static class NetworkFactoryInfo {
		public final String name;
		public final Messenger messenger;
		public final AsyncChannel asyncChannel;

		public NetworkFactoryInfo(String name, Messenger messenger, AsyncChannel asyncChannel) {
			this.name = name;
			this.messenger = messenger;
			this.asyncChannel = asyncChannel;
		}
	}
	
当调用ConnectivityManager.registerNetworkFactory()连接一个NetworkFactory到ConnectivityService时， 在ConnectivityService中会实例化一个NetworkFactoryInfo， 所有的NetworkFactoryInfo保存在一个HashMap ConnectivityService.mNetworkFactoryInfos 中










####1.1 NetworkAgent

ConnectivityService管理系统中的数据连接， android中有多种网络可以提供数据连接， 例如wifi， mobile， 蓝牙， 每一种网络连接有自己单独的service来管理， 例如wifi的WifiService， 这些sevice通过NetworkAgent来向ConnectivityService通知它们的状态变化， 例如， 在WifiService中就使用了一个 WifiNetworkAgent

NetworkAgent与ConnectivityService之间使用AsyncChannel来通信， ConnectivityService作为client端， NetworkAgent作为service端，通过ConnectivityManager提供的一个api将ConnectivityService端连接到一个NetworkAgent端

	public void registerNetworkAgent(Messenger messenger, NetworkInfo ni, LinkProperties lp,
            NetworkCapabilities nc, int score, NetworkMisc misc) 

NetworkAgent的构造方法中会调用该API，和ConnectivityService建立连接

通过AsyncChannel传递给ConnectivityService的message， 在ConnectivityService中有一个内部类NetworkStateTrackerHandler的对象mTrackerHandler， 由其来处理这些message， 
NetworkAgent提供了一些api用于向ConnectivityService发送消息：

	public void sendLinkProperties(LinkProperties linkProperties)；
	public void sendNetworkCapabilities(NetworkCapabilities networkCapabilities)；
	public void sendNetworkInfo(NetworkInfo networkInfo)；
	public void sendNetworkScore(int score)；
	public void addUidRanges(UidRange[] ranges)
	public void removeUidRanges(UidRange[] ranges)
	public void explicitlySelected()

+ sendLinkProperties ： 通过AsyncChannel发送“EVENT_NETWORK_PROPERTIES_CHANGED”通知ConnectivityService， 连接属性发生变化
+ sendNetworkCapabilities ： 通过AsyncChannel发送“EVENT_NETWORK_CAPABILITIES_CHANGED”通知ConnectivityService， 网络能力发生变化
+ sendNetworkInfo ： 通过AsyncChannel发送“EVENT_NETWORK_INFO_CHANGED”通知ConnectivityService， 网络信息发生变化
+ sendNetworkScore ： 通过AsyncChannel发送“EVENT_NETWORK_SCORE_CHANGED”通知ConnectivityService， 网络得分发生变化
+ addUidRanges : 通过AsyncChannel发送“EVENT_UID_RANGES_ADDED”通知ConnectivityService， 添加允许使用VPN的UID， 通常被framework中vpn相关代码使用
+ removeUidRanges ： 通过AsyncChannel发送“EVENT_UID_RANGES_REMOVED”通知ConnectivityService， 添加允许使用VPN的UID， 通常被framework中vpn相关代码使用
+ explicitlySelected ： 通过AsyncChannel发送“EVENT_SET_EXPLICITLY_SELECTED”通知ConnectivityService， 该网络由用户手动选择,以获得特殊对待

NetworkAgent有一个抽象方法， unwanted(), 当ConnectivityService不需要使用该网络时， 会调用AsyncChannel.disconnect()来断开连接， 会触发NetworkAgent.unwanted()， NetworkAgent的子类需要实现该方法

ConnectivityService可以向NetworkAgent发送NetworkAgent.CMD_REPORT_NETWORK_STATUS消息，通知网络状态的变化(VALID_NETWORK或者INVALID_NETWORK) 触发NetworkAgent.networkStatus()， 该方法为一个空方法， 其子类可以根据需要来重写该方法

	abstract protected void unwanted()
	protected void networkStatus(int status)

WifiNetworkAgent继承自NetworkAgent并且实现了其unwanted()方法， 重写了networkStatus()方法

WifistateMachine 在进入 L2Connectedtate 状态时实例化WifiNetworkAgent

####1.1 ConnectivityService处理网络的改变

+ EVENT_NETWORK_CAPABILITIES_CHANGED ： updateCapabilities()
当某个network的Service调用ConnectivityManager.sendLinkProperties()时， ConnectivityService收到 EVENT_NETWORK_CAPABILITIES_CHANGED 消息，并且由updateCapabilities()来处理

当capability的确发生了变化时， 其中会更新对应的NetworkAgentInfo中保存的capabilities， 然后回调 NetworkCallback.onCapabilitiesChanged()来通知所有添加到NetworkAgentInfo中的NetworkRequest的发起者

+ EVENT_NETWORK_PROPERTIES_CHANGED ： updateLinkProperties()
当某个network的Service调用ConnectivityManager.sendLinkProperties()时， ConnectivityService收到 EVENT_NETWORK_PROPERTIES_CHANGED 消息(代表IP， DNS等发生变化)，并且由updateLinkProperties()来处理

调用updateInterfaces()，调用 updateMtu()更新mtu， 调用updateTcpBufferSizes()更新tcpbuffer， 调用updateDnses()设置dns服务器， 调用updateClat()设置clat(clat用于为那些不支持dns64的应用实现ipv4到ipv6的转换), 然后回调NetworkCallback.onLinkPropertiesChanged()回调来通知所有添加到NetworkAgentInfo中的NetworkRequest的发起者，  

+ EVENT_NETWORK_INFO_CHANGED ： updateNetworkInfo()
当某个network的Service调用ConnectivityManager.sendNetworkInfo()时， ConnectivityService收到 EVENT_NETWORK_INFO_CHANGED 消息，并且由updateNetworkInfo()来处理

若network为连接状态且还未被处理， 则对于VPN， 调用netd.createVirtualNetwork，普通网络则调用netd.createPhysicalNetwork(), 调用updateLinkProperties()更新连接属性, 然后回调 NetworkCallback.onPreCheck()来通知所有添加到NetworkAgentInfo中的NetworkRequest的发起者, 再给对应的network的NetworkAgentInfo的NetworkMonitor发送“CMD_NETWORK_CONNECTED”消息， 对于vpn网络， 还会disable deafult proxy， 调用rematchNetworkAndRequests()重新选择网络（因为有新的网络连接上）

若网络处于断开连接的状态或者suspend的状态， 则会断开对应的NetworkAgent的连接, 将导致执行NetworkAgent.unwanted()， 若该网络为vpn， 则还会恢复default proxy (vpn连接上时， 会disable default proxy)

+ EVENT_NETWORK_SCORE_CHANGED ： updateNetworkScore()
当某个network的Service调用ConnectivityManager.sendNetworkScore()时， ConnectivityService收到 EVENT_NETWORK_SCORE_CHANGED 消息，并且由updateNetworkScore()来处理

先调用rematchAllNetworksAndRequests() 重新选择网络(可能或导致网络的切换), 然后调用sendUpdatedScoreToFactories()， 通知NetworkFactory， 每一个NetworkRequest当前的得分

+ EXPLICITLY_SELECTED_NETWORK_SCORE
当某个network的Service调用ConnectivityManager.explicitlySelected()时， ConnectivityService收到 EXPLICITLY_SELECTED_NETWORK_SCORE 消息, 会标记对应的NetworkAgentInfo， 该network是用户手动选择， 对于用户选择的网络， 会给予优待， 在获取其score时， 会返回100分


####1.1 ConnectivityService.NetworkAgentInfo

NetworkAgent在对应的service中实例化， 在连接到ConnectivityService中后， ConnectivityService中会实例化一个NetworkAgentInfo保存NetworkAgent的信息以及通过获取的信息：

+ asyncChannel ： 与NetworkAgent通信的AsyncChannel
+ networkInfo ： 对应的NetworkAgent对应network的networkinfo
+ linkProperties : 对应的NetworkAgent对应的network的linkProperties
+ networkCapabilities : 对应的NetworkAgent对应的network的networkCapabilities
+ currentScore ： 对应的NetworkAgent对应的network的当前得分
+ networkMonitor ： 在NetworkAgentInfo的构造函数中实例化
+ networkRequests ： 对应的NetworkAgent对应的network所能满足的所有的NetworkRequest
+ networkLingered ： 

NetworkAgentInfo还提供了如下的API供ConnectivityService调用：

	public void addRequest(NetworkRequest networkRequest)
	public boolean isVPN()
、	public int getCurrentScore()
	public void setCurrentScore(int newScore)


####1.1 NetworkMonitor

NetworkMonotor用于监测网络是否可用， 区分的方式如下：

+ valid network:
   1. 能够访问internet
   2. 或者使用网页认证(captive portal), 并且用户决定使用这一网络

+ invalid network：
   1. 使用网页认证(captive portal)， 但是用户不想使用
   2. 或者是有问题的网络， 例如dns fail， HTTP请求失败

ConnectivityService可以向一个NetworkMonitor对象发送如下的消息：

+ CMD_NETWORK_CONNECTED ： 通知NetworkMonitor， 对应的网络已经连接上
+ CMD_NETWORK_LINGER ： 通知NetworkMonitor，
+ CMD_NETWORK_DISCONNECTED ： 通知NetworkMonitor， 对应的网络断开连接
+ CMD_FORCE_REEVALUATION ： 通知NetworkMonitor， 重新检查网络是否可用

NetworkMonitor对象可以向ConnectivityService发送如下消息：

+ EVENT_NETWORK_TESTED ： 通知ConnectivityService已经检查完network是否可用， 消息中会附带NETWORK_TEST_RESULT_VALID或者NETWORK_TEST_RESULT_INVALID
+ EVENT_NETWORK_LINGER_COMPLETE ： 通知ConnectivityService已经完成 linger

NetworkMonitor的几个状态如下(其它的状态都是DefaultState的子状态)：

+ DefaultState : 状态的父状态
+ OfflineState : 在EvaluatingState阶段HTTP请求失败或者在CaptivePortalState网页认证失败都会进入OfflineState
+ ValidatedState ： 该状态代表网络可用，如下状态下， 会转移到该状态
   1. 在EvaluatingState状态， 若检查的网络为VPN， 则转移到ValidatedState
   2. 在EvaluatingState状态， 若检查的网络自身声明不具备internet的访问能力，则不进行http请求检查， 直接转移到ValidatedState
   3. 在EvaluatingState状态， 若http的204请求成功， 则转移到ValidatedState
   4. 在CaptivePortalState状态， 若网页认证成功， 则转移到ValidatedState
   5. 在LingeringState状态， 若收到CMD_NETWORK_CONNECTED消息， 则转移到ValidatedState
+ EvaluatingState ： 该状态下检查网络是否可用， 对于vpn网络或者是声明自己没有internet能力的网络， 直接判定为可用， 否则发起http请求204，若返回204说明internet可用， 若返回200～399， 则说明需要进行网页认证，否则， 说明网络不可用， 如下条件下会转移到EvaluatingState
   1. 在DefaultState或其子状态下， 处理CMD_NETWORK_CONNECTED消息时， 会转移到EvaluatingState  
   2. 在DefaultState或其子状态下， 处理CMD_FORCE_REEVALUATION消息时， 会转移到EvaluatingState  
+ UserPromptedState : 该状态下等待用户选择是否进行网页认证
   1. 在EvaluatingState状态，下， 若HTTP请求的返回值代表需要进行网页认证， 转移到UserPromptedState
+ CaptivePortalState ： 在Android L之前， WifistateMachine 中使用一个 CaptivePortalcheckState 来完成网页认证， 但是在Android L中， 移除了该状态， 由NetworkMonitor中的CaptivePortalState状态来完成该任务， 若认证成功则转入ValidatedState， 否则转入OfflineState
   1. UserPromptedState， 若用户确认进行网页认证，则转入CaptivePortalState
+ LingeringState ： 因为score或者其它的原因， ConnectivityService可能会切换网络， 将之前的网络置为unwanted，根据各个service的实现的不同， 可能导致网络断开， 为此， ConnectivityService让对应的NetworkMonitor进入lingering的状态，大约30秒， 当时间到期后， 才将对应的network置为unwanted状态，并向ConnectivityService发送EVENT_NETWORK_LINGER_COMPLETE消息，在这期间， 若网络会恢复正常，则取消lingering状态
   1. Default状态或者其子状态下， 若接收到来自ConnectivityService的“CMD_NETWORK_LINGER”消息， 则进入LingeringState状态

当NetworkAgent与ConenctivityService之间的AsyncChannel连接断开时， ConnectivityService会给对应的NetworkAgentInfo的NetworkMonitor发送CMD_NETWORK_DISCONNECTED"消息时， 接收到此消息后， NetworkMonitor内部的状态机会直接退出，因为NetworkAgent断开连接， 说明对应的network自身的连接断开了， 当该network重新连接上之后， 会重新实例化NetworkAgent并连接ConnectivityService， ConnectivityService端则会重新实例化NetworkAgentInfo， 其内部则会实例化一个NetworkMonitor并start其状态机

