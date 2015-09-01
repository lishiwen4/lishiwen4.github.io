---
layout: post
title: "射频基础知识"
description:
category: wifi
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---


###1 发射功率

无线电发射机输出的射频信号，通过馈线（电缆）输送到天线，由天线以电磁波形式辐射出去。电磁波到达接收地点后，由天线接收下来（仅仅接收很小很小一部分功率），并通过馈线送到无线电接收机。因此在无线网络的工程中，计算发射装置的发射功率与天线的辐射能力非常重要

发射功率使用W或者dBW作为单位
	
	p(dBW) = 10 * lg( xW / 1W)

在无线通信中， 功率通常为mW级别， 因此， 将dBW极化为dBm

	1W = 1000mW
	p(dBm) = 10 * lg( x * 1000mW / 1mW)

可以计算得出
	
	1W = 0dBW
	1W = 30dBm

###2. 3dB法则

dB是一个表征相对值的值， 例如：

+ 若如果甲的功率为46dBm，乙的功率为40dBm，则可以说，甲比乙大6 dB
+ 如果甲天线为12dBi，乙天线为14dBi，可以说甲比乙小2 dB

3dB法则：

1. 相差10dB， 即相差10倍
2. 相差3dB， 约相差2倍
3. 相差1dB， 约相差1.25倍

例如若如果甲的功率为46dBm，乙的功率为40dBm，则可以说，甲比乙大6 dB， 即甲的功率为乙的4倍

在比较 dBW， dBm， dBi ......时都可以使用该法则

###3. 增益

天线被用来把电流波转换成电磁波，在转换过程中还可以对发射和接收的信号进行“放大”，这种能量放大的度量成为 “增益（Gain）”， 使用全向作为参考时， 天线增益的度量单位为“dBi”， 其原理如下：

当某种辐射源向空间辐射能量时，理想情况下能量是按球状体散射开来，研究和实践都发现，如果将这种能量辐射按某个方向集中发射，则能量所达到的距离以及该方向上所覆盖的范围都会有很大的提高。这种研究成果应用到无线通信中就是天线的由来。而天线增益则是用于定量的描述天线把输入功率（能量）集中辐射的程度，从通信角度讲，就是在某个方向上和范围内产生信号能力的大小。

**若使用偶极天线作为参考，则使用dBd作为单位， 在dBd转换为dBi时， 一般要加上2.15**

实际应用中，即使集中某个方向，天线还是会在空间各个方向都有大小不同的增益，天线增益通常是指产生最大增益的方向上的增益，数学上用公式

	G = 10 * lg( Pout / Pin)

为了更好的理解天线增益，用此公式举例如下：要在空间某点产生一定大小的信号，用理想的辐射源如需126 W的输入功率才能获得，假设某个天线的增益为18 dBi，则用此天线需要输入的功率大小用上面的计算公式列式为

	18 = 10 * lg( 126 / Pin)

计算可得， Pin为2W， 即18DBi的增益相当于将2 W的辐射能量放大到了126 W辐射能量所达到的效果

运用3dB法则
	
	18dB = 10dB + 3dB + 3dB + 1dB + 1dB
	10 * 2 * 2 * 1.25 * 1.25 = 62.5

而 2W * 62.5 = 125W， 约等于126W， 结果与上述计算方式相符

**通常我们使用的无线路由的天线的增益一为 3～5dBi**

###4. 等效功率EIRP

无线系统中的电磁波能量是由发射设备的发射能量和天线的放大叠加作用产生, 应该一起度量， 使用EIRP(Effective Isotropic Radiated Power)来表示

无线通信系统通常采用各方向具有相同单位增益的理想全向天线作为参考天线。EIRP定义为：EIRP=Pt*Gt，它表示同全向天线相比，可由发射机获得的在最大天线增益方向上的 发射功率。Pt表示发射机的发射功率，Gt表示发射天线的天线增益， 当然如果使用对数的话， 因该做加法而不是乘法

	EIRP = 发射功率 + 天线增益

例如， 若发射功率为 20dBm， 天线增益为 10dBi， 则

	EIRP = 20dBm + 10dBi = 30dBm = 1000mW

当然， 实际上还要考虑发射机与天线之间的cab loss

在“小功率”系统中（例如无线局域网络设备）每个 dB 都非常重要

###5. Wifi设备发射功率的限度

FCC就有规定电磁波的发射功率限度，规定路由器的输出功率最大值是1个瓦特（等效30dBm），而加上天线的“聚焦”作用后，等效强度不能超过36dBm， 参见[FCC电磁波发射功率限度](http://www.air802.com/files/FCC-Rules-and-Regulations.pdf)

对于wifi来说， 手持终端的发射功率一般都在20dBm以下， 目前只有AP的发射功率会超过20dBm， 而IEEE规定发射功率超过20dBm的WiFi设备应该提供功率控制， 最多可以提供4种功率级别， 最小应该能将发射功率前换回100mW或者更小 

中国及欧洲规定最大功率不能超过20dBm， 而日本为22dBm

###6. 无线路由的穿墙模式

1. 硬穿墙

硬件设计上增加PA，放大机器的发射功率，不按照国家CCC的标准做。简称“硬穿墙”，

2. 软穿墙

软件设计上调高发射功率，牺牲EVM，简称“软穿墙”；这类的作法属于只看信号不看品质的作法，达到的效果是确实可以传输的远，但有丢包，延迟比较大

**无论是2.4G频段还是5G频段， 穿墙能力都较弱， 主要还是依靠信号的反射绕墙， 因此， 在放置无线路由时可根据这一规则合理放置路由**

###7. RSSI

RSSI(Received Signal Strength Indication) 代表接收的信号强度指示, 因为直接使用dBm来表示实际接收功率的话， 无法直观了解信号强度的好坏， 

**需要注意的是， 在802.11-2007标准中， RSSI与实际接收功率的映射是由芯片厂商独立实施的， 即无线局域网供应商可以按照私有的方式定义RSSI值**， RSSI的范围可由供应商自己选择从0到最大值(小于等于255)， 在比较不同制造商的无线网卡的RSSI值时会出现两个问题：

1. 不同厂商可能选择不同的RSSI最大值，例如A厂商商可能选择RSSI范围从[0,100]，而B厂商选择[0,30]，于是A厂商表明当前信号为25时，B厂商针对相同信号可能表达为8
2. 厂商商可能采用不同的RSSI计数和比较范围, 例如A厂商可能的计数范围为100，关联-100到0的dBm值，而B厂商可能的计数范围为60，关联-95到-35的dBm值。计数方案不同，取值范围也可能不同

尽管Wi-Fi芯片厂商采用私有的RSSI值，但在RSSI阈值取值方面是相似的, RSSI阈值可用于漫游和速率选择

在802.11协议中, RSSI和SQ可以通过PHY_CCA.indicate通报给MAC层

###8. 接收灵敏度

灵敏度是指接收器能够正确将信号转换为数据的最微弱信号， 当接收端的信号能量小于标称的接收灵敏度时，接收端将不会接收任何数据， 灵敏度愈高(数值越低代表灵敏度越高),传输距离就愈长(提高灵敏度也有助于改善其他效能,不过传输距离是最便于讨论的一个。)

对于wifi而言灵敏度系由 802.11 硬件层所决定， 在802.11中接收灵敏度通常被定义为一个与传输速率相关的函数

+ 对于FHSS 1.0Mbps 物理层中规定， 在随机生成400字节的PSDU数据， 误码率<=3%时，灵敏度的数值应该<=-80dBm， 并且能够接收强度在标称灵敏度到-20dBm之间的信号
+ 对于FHSS 2.0Mbps 物理层中规定， 在随机生成400字节的PSDU数据， 误码率<=%3时，灵敏度的数值应该<=-75dBm， 并且能够接收强度在标称灵敏度到-20dBm之间的信号
+ 对于DSSS 使用 DQPSK 2.0Mbps编码时， 在MPDU为1024字节， 误码率<=8%时， 灵敏度的数值应该<=-80dBm， 并且能够接收强度在标称灵敏度到-4dBm之间的信号
+ 对于HR/DSSS 使用 CCK 11.0Mbps编码时， 在PSDU为1024字节， 误码率<=8%时， 灵敏度的数值应该<=-76dBm， 并且能够接收强度在标称灵敏度到-10dBm之间的信号

	数 据 速 率	|	接收信号强度
	--------------------------------------
	54Mbps		|	-50dBm
	48Mbps		|	-55dBm
	36Mbps		|	-61dBm
	24Mbps		|	-74dBm
	18Mbps		|	-70dBm
	12Mbps		|	-75dBm
	9Mbps		|	-80dBm
	6Mbps		|	-86dBm


###9. 天线增益对接收距离的影响

802.11b产品在11Mbps传输模式下的测试结果， 可做为定性参考

横向为接收端天线增益， 纵向为发送端天线增益， 其它不变量未列出

	天线增益(dbi)	|	18	14	8	6	5
	------------------------------------------------------------
	18		|	5.5Km	4Km	1.5Km	1.0Km	0.6Km
	14		|	2.5Km	2.5Km	1.5Km	0.8Km	0.6Km
	8		|	1.0Km	1.0Km	1.0Km	0.8Km	0.6Km
	6		|	0.8Km	0.8Km	0.8Km	0.8Km	0.6Km
	5		|	0.8Km	0.6Km	0.6Km	0.6Km	0.5Km


###10. 距离与发射功率以及天线增益的关系

	Pr = Pt * Gt * Gr * [ λ /(4πd)]^2

+ pt : 发送端输出功率
+ pr : 接收端灵敏度
+ Gt : 发送端天线增益
+ Gr : 接收端天线灵敏度
+ d  : 发送端天线和接收端天线的距离
+ λ  : 载波波长

###11. FEM(Front End Modules)

RF的前端模块，也被称为FEM， 是PA 和 LNA的总称

1. PA: 主要功能是功率放大, 考虑其高的线性区和高增益, 输出功率可以很高， 但同时效率低，PA工作时电流较大， 一点噪声很难影响到信噪比, 常用于发射电路的最后一级
2. LNA ： 是低噪声放大器，LNA的线性区增益较低，最大输入功率不是很高， 但是其效率较高， LNA在放大信号的时候基本上电流很小，一方面是因为信号小，另一方面就是其效率高bias低， LNA常用于接收电路，因为接收电路中的信噪比通常是很低的，往往信号远小于噪声，通过放大器的时候，信号和噪声一起被放大的话非常不利于后续处理，这就要求放大器能够抑制噪声

###12. iPA iLNA 与 xPA xLNA

+ iPA iLNA : 制造工艺为CMOS, 常采用8英寸的Wafer， 实现CMOS工艺的技术难度非常大。一个是CMOS工艺对功率有要求，如果电压功率太大会烧掉;第二个是截止频率低的时候惰性强，为了提高功率需要选择更厚的材料;第三是转换效率不高，但是CMOS的集成度高， 并且CMOS工艺在性能上也具有优势，包括射频性能、灵敏度、热导性、噪声指数都具有天然的优势。通过平均功率追踪(APT)、封包追踪(ET)、数字预失真(DPD)、天线调谐等技术，CMOS解决方案已能满足载波聚合对线性及功耗的需求
+ xPA xLNA ： 制造工艺为GaAs(砷化镓), 常采用6英寸的Wafer，单颗die成本要高， 但是，砷化镓工艺的导电性能更好，接收频率高，在高频传输领域，砷化镓的物理性质有其不可替代性，GSM和3G上强调的是功率，不强调线性度。而WIFI、LTE非常强调线性度， 因此CMOS工艺PA LNA当前还是多用在WIFI， LTE上

总结起来就是，砷化镓性能好，功耗低， 但是面积大，集成度低， 成本高，由于砷化镓工艺所需要的材料比较稀缺，不管是材料成本和制造成本都比较高，对于生产线的要求也很高， 通常产能受限， 且良率也较CMOS工艺低，而CMOS面积小，集成度高， 价格低， 且原材料不受限制， 但是性能和功耗要差， 当前市场主流还是砷化镓， 但是CMOS正在慢慢渗透

###13. 信噪比SNR

信噪比SNR(Signal-to-noise ratio) 即信号与噪声的比值， 例如接收信号强度为-85dBm，而本底噪声为-100dBm时，两者之间的差异就是15dB，SNR就是15dB

对于wifi来说25dB或以上的信噪比被认为信号质量很好，而10dB或以下的信噪比被认为信号质量很糟糕，芯片无法区分背景噪声和WIFI信号

**需要记住的是， 802.11无线网卡不能测量出真实的本底噪声和信噪比， 可以真实测量非编码RF能量的只有频谱分析仪**， 很多时候， 可以看到802.11设备显示出信号强度和信噪比， Wifi芯片厂商使用一些办法来估测本底噪声, 因为802.11无线网卡只能处理比特，所以他们需要某种算法利用通过无线网卡的比特来计算噪声值

与RSSI测量一样，每家wifi芯片厂商计算噪声的方法不同。有些供应商拒绝只依靠比特流估测噪声值(可能会假设噪声功率为固定值， 通常会假设底噪为-99dBm ~ -95dBm之间)，有些供应商则开发出了复杂的算法计算噪声值, 例如有些802.11芯片制造商利用了关闭编码过滤器的方法，直接利用从天线进入的RF信号，将802.11芯片变成一台初级的频谱分析仪，但这取代了802.11网卡处理数据的能力。这些新型芯片要么作为轻量型的频谱分析仪，要么作为普通网卡处理数据，永远无法同时拥有两个功能。因为芯片前端的过滤器能够识别802.11信号，把它交给802.11协议栈处理，而不是协议分析仪
