---
layout: post
title: "wifi漫游与802.11r"
description:
category: wifi
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. wifi 漫游

wifi漫游指的是一个wifi的STA在一个ESS内从一个BSS接入到另外一个BSS

大部分的终端设备在漫游的时候不会和旧AP断开，即不会主动向旧AP发送解关联或解认证报文，而直接选择一个新AP进行链接建立

####1.1 漫游由STA来决定

802.11 的漫游机制,完全取决于用户端的决定。何时何地送出 Association request 信帧, 完全操之于用户端系统的驱动程序与软件, 对于不同的系统， 可能是无线芯片的fw或者是driver来发起漫游， 也可能是由上层的应用程序来发起漫游

####1.2 何时发起漫游

802.11 并未限制STA如何决定是否切换基站,而且不允许基站以任何直接的方式影响STA的决定，就算STA决定与信号最差的基站连接,虽然是相当糟糕的做法,却仍旧符合 802.11 的规范，事实上大多数STA以信号强度或品质做为主要依据,并试图与信号最强的基站进行连接

大部分STA会在刚开始连接时， 选择信号较好的基站进行连接，并且会随时监测所收到帧的信噪比,并以目前所使用的数据传输率,判定何时应该漫游到新的基站。如果数据传输率已经很慢且信噪比又低,用户端系统就会开始寻找其他基站， 但是也有一些STA会在连接一个基站之后， 就与之终生厮守,就算STA距离基站愈来愈远,而且信号强度持续滑落,也不会开始进行漫游过程, 直到与其断开连接才考虑连接到信号较好的基站

**因为漫游完全是一个STA端的行为， 因此， 不同的STA的漫游表现有可能差异较大**

####1.3 基于信号强度的漫游

STA的漫游通常取决于驱动算法， RSSI， RSN， 上一次连接的终端等因素的影响，大部分的STA使用RSSI强度作为参考， 其过程如下：

+ 扫描阈值 ： 当当前连接的基站的RSSI低于该值时， 启动漫游扫描过程， 开始查找是否有合适的基站用于漫游
+ 切换阈值 ： 当漫游扫描过程检测到有另一个基站的RSSI好于当前连接的基站， 并且差值要大于该阈值， 则进行漫游

####1.4 AP诱导STA漫游

上面说过**802.11协议不允许基站以任何直接的方式影响STA的决定**， 事实上，作为优化手段，一些AP可能会使用一些间接的手段来诱导STA漫游， 主要有：

1. **AP触发重新认证:** AP将感知无线客户端的信号强度，并根据指定的条件阈值进行判断，如果信号强度低于指定值，那么AP会主动向客户端发送解除认证帧报文，给无线客户端一次重新连接或者漫游的机会。这个手段对于希望终端始终连接到信号最好的AP这种需求有促进作用，其是通过主动将终端踢下线来改变终端漫游不灵敏的现状，从而实现无线链路良好状态的持续，这一过程会造成业务中断， 因此适用于那些不注重漫游过程， 只注重漫游结果， 希望终端能够很及时的切换到最佳的AP上的场景
2. **降低AP的发送功率:** AP也将感知无线客户端的信号强度，并根据指定的条件阈值进行判定，如果信号强度低于指定值，那么AP会主动降低其Beacon管理帧的发送功率至设定值，同时对于响应终端的Probe Response报文也以设定功率值进行发送。这样终端无论是从Beacon帧采集信号信息，还是从Probe Response采集，都可感知到原有AP信号的强度降低，从而选择其它信号良好的AP，实现主动漫游的效果, 如果终端漫游离开AP，则AP的Beacon帧发送功率会恢复. 即适用于关注漫游的过程而希望终端漫游尽可能无缝不间断

####1.5 不同加密方式下的漫游时间

+ OPEN WEP共享密钥和PSK的方式， 因为认证的流程很短， 漫游时间也较短， 基本能够满足业务不中断的需求
+ 802.1x认证由于认证过程较长， 导致漫游时间大于200ms， 对于实时性的业务影响较大， 为此引入了快速漫游技术

####1.6 802.11k

802.11k是一个新提出的标准，它为无线局域网应该如何进行信道选择、漫游服务和传输功率控制提供了标准。它也是802.11规范家族的一部分。　　在一个无线局域网内，每个设备通常连接到提供最强信号的接入点。这种管理有时可能导致对一个接入点过度需求并且会使其他接入点利用率降低，从而导致整个网络的性能降低，这主要是由接入用户的数目及地理位置决定的。在一个遵守802.11k规范的网络中，如果具有最强信号的接入点以其最大容量加载，而一个无线设备连接到一个利用率较低的接入点，在这种情况下，即使其信号可能比较弱，但是总体吞吐量还是比较大的，这是因为这时网络资源得到了更加有效的利用

###2. fast roaming

Fast Roaming就是为了提高roaming的效率。在802.11r前，通常是指在RSN(WPA2)的框架下，略过EAP/802.1X，直接进入4way-handshake阶段

在802.11r发布之前，Fast Roaming大致分为3种情况：

+ PMK Caching
+ Preauthentication
+ OKC（Opportunistic Key Caching）

无论哪种情况，在Roaming时的流程都是差不多的：STA在(re)associate request的RSN IE里带上一个PMKID List，然后AP在PMKSA Cache里找到一个PMKID匹配的Entry, 差别在于Roaming前的行为

####2.1 PMKSA

PMKSA(Pairwise masker key security association) 是由supplicant与AS服务器成功完成802.1X/EAP authentication认证或者PSK认证之后生成(当完成4way-handshake， 并且派生PTK之后，即生成PMKSA)， 具体可由如下步骤生成:

+ 完整的802.1X/EAP认证
+ PSK authentication
+ PMK caching
+ Preauthentication

PMKSA存在于STA和AP端， 包含如下信息：

+ PMK
+ PMKID
+ AP的MAC
+ lifetime
+ AKMP 
+ Authorization parameters

一个PMKID用于标识一个PMKSA， 生成方式如下：

	PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)

**PMKID只会在由STA发往AP的association帧和reassociation帧中的RSN IE中出现, STA端可以保存多份PMKSA， 因此这些帧中可以出现多个PMKID**

**PMKID用于快速漫游， 下文会详细叙述**

####2.2 PMK(Pairwise Master Key) Caching

PMK Caching是最早的fast-secure方案， 作为IEEE 802.11i的补充

1. 当STA roam到新的AP后， 旧的AP和STA都会缓存它们的PMKSA一段时间
2. 当STA roam back到旧的AP时， STA会发送reassociation帧并在RSN IE中列出PMKID(允许不止一个， 但是通常只有一个)
3. 若AP为其缓存的PMKSA仍存在， 则能够找到匹配的PMKID， 则直接跳过802.1x/EAP认证， 进入4way-handshake

这一技术通常被称为"fast secure roam-back", 能够将roaming的延迟降低到 100ms以内

PMK Caching并未得到大规模的应用， 和如下的原因相关:

1. 这一方案是可选的， 但是很多WPA2设备并不支持，因为802.11i的着重点不在 fast-secure roaming 上，并且， IEEE成立了802.11r工作组来制定fast-secure roaming标准
2. 这一方案有很大的局限性 : 只能在回到已经认证过的AP时才能执行 fast-secure roaming， 当roam到未认证过的AP时， 仍需要执行完整的认证过程

设备能够缓存的PMK数目通常都存在限制， 当达到上限后， 会移除之前缓存的PMK以获取空间

**这一技术也被称为SKC(Sticky Key Caching)， 区别于下文的OKC**

####2.3 pre-authentication

pre-authentication在IEEE 802.11-2007中定义, 是指在roam到一个新的AP之前就和其建立好PMKSA， **预认证需要使用RSN IE， 因WPA不包含RSN IE， 因此不支持预认证**

1. 当STA连接上一个AP之后， 通过该AP与其它的AP分别进行认证， 生成对应的PMK和PMKSA， 
2. AP 会在其 RSN IE中通告其是否支持pre-authentication
3. 当roam 到其它的AP时， STA在reassociation帧的RSN IE中列出PMKID， 若AP匹配到该PMKID， 即可绕过认证过程，进入4way-handshake

pre-authentication的伸缩性并不好， STA需要同每一个AP进行pre-authentication， 会加重radius的负担

IEEE成立了802.11r工作组， 试图推出更好的fast roaming方案， 但是很多厂商等不及， 推出了“Opportunistic key caching”技术

####2.4 Opportunistic key caching
  
**OKC也叫OPC (Opportunistic PMK Caching) , 是PMK caching的增强版, 并非IEEE 802.11标准， 而是由微软定义，不过多数厂商都支持这种方式，也成为了一种事实标准**

OKC技术在STA和所有的AP之间共享同一份PMK (这一份PMK如何传递其它的AP到每一个AP由厂商自行实现)， 而无需同每一个AP都进行认证， 生成PMK

1. STA同第一个AP认证， 生成PMK和PMKID， 这一PMK缓存在WLAN controller上， 并由其发送到所有的其它AP上
2. 当STA漫游到其它的AP时， 利用PMKID的计算公式， 计算新的PMKID， 然后附带在reassociation帧的RSN IE中， 发送给新的AP
3. AP也利用PMKID的计算公式， 计算STA的PMKID， 因为他们使用的PMK相同， 因此， 计算出的PMKID也相同， 即可绕过认证过程，进入4way-handshake


###3. IEEE 802.11r (Fast BSS Transition)

简称FT， 是IEEE802.11 提出的第一份关于AP间快速转移的官方方案， 鉴于上叙的几种快速漫游方案已经被应用了一段时间， FT 的推广进展缓慢

+ Preauthentication虽然在Roaming期间略过了802.1X，但还是每次都要做，只不过提前罢了
+ OKC的出现，就是为了解决每次都要做802.1X的问题，可以提高效率，降低网络负荷。可是，OKC的方便是建立在牺牲安全性的基础上的。这会导致每个AP上都拿到相同的PMK。

为了同时解决Preauthentication和OKC的缺陷，IEEE推出了802.11r，对Fast Roaming进行了补充

802.11r 将一个ESS定义为一个MD(移动域) ， 并且增加了 MDIE， FTIE， TIE等元素和几种Action帧

在RSN（802.11i）的基础上,802.11r提出了三层密钥结构和计算方法(而RSN则是一层)分别为:

1. PMK_R0 ： 第一层， 由MSK或者PSK推演得到， 由R0KH和S0KH角色持有
2. PMK_R1 ： 第二层， 由S0KH和R0KH共同推演得到， 由R1KH和S1KH角色持有
3. PTK ： 第3层， 由S1KH和R1KH共同推演得到

PMK_R0和PMK_R1的计算则是80211r特有的,PTK的计算方式与80211i的计算方式也不同，80211i是通过伪随机函数（PRF）展开PMK来获得PTK，而80211i则是通过密钥派生函数（KDF）来展开PMK，KDF函数实际上是PRF的变种。 
 
不同层级的密钥由不同的角色持有：

+ R0KH ： (PMK R0 Key holder)为认证者(AS)一端的密钥管理实体，PMK_R0和PMK_R1的计算由R0KH控制，R0KH同时还要负责提供PMK_R1给R1KH
+ R1KH ： (PMK R1 Key holder)为认证者一端的密钥管理实体，PTK的计算由R1KH控制
+ S0KH ： 为申请者(STA)一端的密钥管理实体， 功能与R0KH相对应
+ S1KH ： 为申请者一端的密钥管理实体， 功能与R1KH相对应

这些角色使用不同的ID来标识：

+ R0KH-ID ： R0KH的标识（NAS—ID）,按IEEE标准为1-48字节长,可由厂商自定义
+ R1KH-ID ： R1KH的标识，认证者的MAC地址
+ S0KH-ID ： S0KH的标识，无线工作站的MAC地址
+ S1KH-ID ： S1KH的标识，无线工作站的MAC地址

####3.1 KDF 函数

KDF函数用于PMK_R0和PMK_R1的生成过程， 其原型描述为

	Output <- KDF-Length (K, label, Context) 
 
+ Input:
   1. K : 256bit的初始key
   2. label ： 一个字符串描述生成key的目的
   3. context ： 一个bit串用于标识生成的key
   4. Length ： 生成的key的长度
+ Output：生成的key

KDF函数的伪码为

	result = ""
	iterations = (Length+255)/256
	do i = 1 to iterations
		result = result || HMAC-SHA256(K, i || label || Context || Length)
	od
	return first Length bits of result, and securely delete all unused bits

####3.2 PMK_R0的计算

1. 如果采用的是802.1X认证方式，R0KH就根据认证交互过程中从radius认证服务器 那得到的主绘话密钥(MSK)计算出PMK_R0
2. 如果是采用PSK方式则直接根据 PSK计算出PMK_R0

PMK_R0的计算

	R0-Key-Data = KDF-384(XXKey, "FT-R0", SSIDlength || SSID || MDID || R0KHlength || R0KH-ID || S0KH-ID)
	PMK-R0 = L(R0-Key-Data, 0, 256)
	PMK-R0Name-Salt = L(R0-Key-Data, 256, 128)

+ KDF-384() ： 代表使用KDF函数生成384bit的key
+ L() ： 代表从某个bit串的指定位置开始， 提取指定长度的bit串
+ XXKey ： 对于802.1x认证， XXKey=L(MSK, 256, 256), 对于PSK认证， XXKey=PSK
+ SSIDlength ： 一个字节， 表示SSID的字节个数
+ MDID ： 移动域的ID， 从帧中的MD IE中获得
+ R0KHlength ： 一个字节，表示R0KH-ID的字节个数
+ R0KH-ID : 1~48字节，厂商自定义
+ S0KH-ID : STA的MAC

PMK_R0 可使用 PMKR0NAME来表示

	PMKR0Name = Truncate-128(SHA-256("FT-R0N" || PMK-R0Name-Salt))

**一个MD中PMK_R0只有一份，PMK_R0的生命周期取决于认证时得到的MSK或者PSK，长度不能超过MSK或者本身的生命周期**

当认证成功后，R0KH删除MD中以前存有的与S0KH有关的 PMK_R0的安全关联，以及由它计算出来的PMK_R1的安全关联

####3.3 PMK_R1的计算

R0KH 遍历MD内的不同R1KH， 根据他们的R1KH_ID计算出PMK_R1， 并将它传给同一MD中的所有R1KHs。PMKRls是用来计算成对临时密钥(PTK)的。R1KH接收到PMK—R1时就会删除以前的 PMK_R1的安全关联，以及它所计算出的PTKSAs

R0KH为R1KHs计算PMK_R1的时机，802.11r文档没有明确说明，从状态机上看应该是计算出PTK之后，再为其他R1KH计算分发PMK_R1

PMK_R1的计算

	PMK-R1 = KDF-256(PMK-R0, "FT-R1", R1KH-ID || S1KH-ID)

PMK_R1 可使用PMKR1NAME来索引

	PMKR1Name = Truncate-128(SHA-256(“FT-R1N” || PMKR0Name || R1KH-ID || S1KH-ID))

**PMK_R1认证时得到的MSK或者PSK，长度不能超过MSK或者PSK本身的生命周期**

####3.4 PTK的生成

PTK的计算

	PTK = KDF-PTKLen(PMK-R1, "FT-PTK", SNonce || ANonce || BSSID || STA-ADDR)

+ SNonce ： S1KH 生成的256bit随机数
+ ANonce ： R1KH 生成的256bit随机数
+ BSSID ： 目标AP的BSSID
+ STA-ADDR ： STA的MAC

PTK 可使用PTKNAME来索引

	PTKName = Truncate-128(SHA-256(PMKR1Name || “FT-PTKN” || SNonce || ANonce || BSSID || STA-ADDR))

每一个PTK包含3个key：

	KCK = L(PTK, 0, 128)
	KEK = L(PTK, 128, 128)
	TK = L(PTK, 256, 128)

####3.5 STA和AP的初次关联

1. AP在Beacon帧和probe response帧里添加MDIE(如果支持FT)和RSNIE(如果支持802.11i)
2. STA(Re)Association_Requese帧中添加MDIE(如果支持FT)和RSNIE(如果支持802.11i)
3. 关联期间， AP会对比两者的MDIE和RSNIE， 若不一致会导致关联失败
4. 关联成功后，AP向STA发送assoc resp帧，并告知工作站R0KH_ID和R1KH_ID.为以后生产三层密钥（PMK_R0，PMK_R1，PTK)做准备
5. 关联成功后, 若使用802.1x 则进行802.1x的验证（为PSK则跳过），802.1x验证完成后认证者将MSK发给STA
6. 收到MSK后的R0KH和S0KH开始计算PMK_R0和PMKR0Name
7. R1KH从R0KH获取到PMK_R1，后，随后计算随机值Anonce，接下来开始四次握手
8. R1KH进入FT-PTK-START阶段后，向工作站发出EAPOL_Key帧，内含随机值Anonce，这一帧不加密，但不用担心被篡改，因为一旦Anonce被篡改，那么STA和AP上计算出的PTK一定不一样
9. STA接收该帧时，S1KH已经计算出PMK_R1，随后S1KH计算出本地随机值Snonce，并计算出PTK
10. 计算完毕后，STA向AP发送一个EAPOL-KEY帧，附带自己的Snonce，PMKR1Name，MDIE，FTIE。这一帧也不加密，但附带了MIC校对，篡改内容会导致校验失败。同时MDIE，FTIE必须和Assoc response帧内的MDIE，FTIE相同
11. AP根据STA发来的snonce由计算出PTK。AP会在第三次握手中将组临时密钥（GTK）发过去，这一帧经过加密，因为此时双方拥有公共的PTK。第三次握手时附带的PMKR1Name和第二次握手时发送的PMKR1name必须相同
12. STA解密后收到GTK，同时发送带MIC的确认帧，802.1x受控端口打开，STA可以正常访问网络了，如果PTKSA超期，STA需要重新进行FT初始化关联

####3.6 BSS快速切换

从切换方式来讲，FBT可以分为Over_the_ds和Over_the_air方式

Over_the_air的切换过程如下：

1. STA向AP发送Authentication request帧， 附带在RSNIE中附带PMKR0NAME， 在FTIE中附带SNoce R0KH-ID， 以及附带MDIE， 请求帧的验证算法为FT， 不再是open或者shared
2. STA的MDIE和目标AP的MDIE必须一致， 否则验证失败， 另外若PMKR0NAME不可用或者R0KH 不可达， 则会报错
3. 目标AP的R1KH利用PMKR0Name和帧中的其它信息计算出PMKRlName。 如果AP没有PMKRlName标识的Key（PMK_R1），R1KH就会从STA指示的R0KH获得这个Key。目标AP收到一个新的PMKR1后就会删除以前的PMKR1安全关联以 及它计算出的PTKSAs
4. 如果目标R1KH经过计算找不到PMK_R1， 则会向R0KH发出请求， 获取PMK_R1
5. R1KH 计算出随机值 RNonce， 然后计算出PTK， 通过Authentication response帧RNonce将返回给STA
6. auth response帧后,S1KH计算出PTK
7. STA向AP发送reassociation request， 携带 PMKRlName、ANonee、SNonce、MIC值、R1KH．ID和R0KH_ID
8. Target AP收到reassociation request帧并且验证无误的话，回复Reassoc response帧给STA，帧内带有加密后的GTK，STA可以用PTK解密获取GTK
9. 重关联完成后，802.1x受控端口打开， 即可使用网络

Over_the_ds 快速切换的reassociation阶段和Over_the_air相同，但获取PTK的步骤不同

TA会发送一个Action request帧给current AP，帧内包含Target AP的mac地址，current AP会转发这一帧给target AP

Action帧由当前AP到目标AP的转发是通过有线传输的, IEEE802．11r专门定议了RRB(远程中继代理)帧格式。 RRB帧格式的以太网类型为0x890d, 当STA通过当前AP与目标AP通信时，首先是将所要发的内容封装在Action 管理帧中，目的地址为目标AP的地址。当前AP收到这一Action帧时就对这一帧 进行分析发现不是给自己的就进行转发。当前AP将Action帧封装在一个以太网 类型为0x890d的以太网中，并插入了当前AP的地址，然后通过以太网转发给目 标AP。这样，目标AP收到这一帧后就知道是由哪个AP转发的Action帧，于是目标AP也按照同时的帧格式将回复帧发给当前AP。当前AP收到帧后将封装在里面 的Action帧提出来转发给STA


###4. wifi漫游网络的条件

wifi 漫游需要满足不同的AP具有相同的SSID和加密方式:

使用 TL-WDR7500 和 TL-WDR6300 设置为相同的SSID， 测试结果如下：

**WEP加密：**

同时设置为“WEP 开放系统” 64位加密， 可成功roaming

**WPA/WPA2-PSK 加密：**   

WPA-PSK 加密方式的组合及漫游测试结果如下

				|	[WPA-PSK-TKIP]	[WPA-PSK-CCMP]	[WPA-PSK-CCMP+TKIP]
	--------------------------------------------------------------------------------------------
	[WPA-PSK-TKIP]		|		Y		N		Y
	[WPA-PSK-CCMP]		|		N		Y		N
	[WPA-PSK-CCMP+TKIP]	|		Y		N		Y

WPA2-PSK 加密方式的组合及漫游测试结果如下 

				|	[WPA2-PSK-TKIP]	[WPA2-PSK-CCMP]	[WPA2-PSK-CCMP+TKIP]
	---------------------------------------------------------------------------------------------
	[WPA2-PSK-TKIP]		|		Y		N		Y
	[WPA2-PSK-CCMP]		|		N		Y		N
	[WPA2-PSK-CCMP+TKIP]	|		Y		N		Y

WPA-PSK / WPA2-PSK mixed 可视为等同WPA2-PSK加密方式， 其组合及漫游测试结果如下
	
						|	[WPA2-PSK-TKIP]	[WPA2-PSK-CCMP]	[WPA2-PSK-CCMP+TKIP]	[WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP]
	-------------------------------------------------------------------------------------------------------------------------------------------------------------
	[WPA2-PSK-TKIP]				|		Y		N		Y			Y
	[WPA2-PSK-CCMP]				|		N		Y		N			N
	[WPA2-PSK-CCMP+TKIP]			|		Y		N		Y			Y
	[WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP]	|		Y		N		Y			Y
