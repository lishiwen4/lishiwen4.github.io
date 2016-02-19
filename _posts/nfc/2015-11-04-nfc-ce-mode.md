---
layout: post
title: "NFC CE 模式"
description:
category: nfc
tags: [nfc]
mathjax: 
chart:
comments: false
---

###1. NFC CE mode

NFC CE模式允许具备NFC芯片的设备能够充当智能卡(例如信用卡)来使用， 该模式所支持的应用场景极具前景， 例如， 支持该功能的Android智能手机来完成购票， 支付， 公交卡， 门禁卡等

NFC CE mode有两种实现：

+ 虚拟卡模拟 : 基于硬件，这种模式下，需要提供SE单元， nfc芯片作为通信的前端模块， 将收到的操作命令， 转发到SE， 由SE负责处理
+ 主机卡模拟 : 基于软件，这种模式下，由运行在主机上的应用来充当SE

关于NFC CE mode， NFC Forum暂时还没有相关的规范

###2. 虚拟卡模拟

虚拟卡模拟能够模拟的卡类型， 取决于nfc 芯片， 例如NXP的芯片，可以方便的模拟M1和ULTRALIGHT等

####2.1 SE与nfc芯片的连接

虚拟卡模拟依赖SE单元， 安全单元为NFC设备上专用的微处理芯片。该芯片可以与NFC控制器集成在一起。另外也可以集成在NFC设备中的其它智能卡/安全设备中， 目前常见的SE单元的形式有

+ nfc 芯片内部集成的SE单元， 例如 NXP 的 PN65N
+ Secure SD 卡， 实际上是在SD卡内部嵌入了一个安全模块， 相关的应用可以在其上运行， 相关的国际标准为 ISO 7816, 该方案被称为NFC-SD
+ UICC， 即手机的SIM卡充当SE，该方案被称为NFC-SIM

目前使用较多的方案是 NFC-SIM 的方案

**很多安全单元（例如NXP的SmartMX）使用的是标准的智能卡芯片，包括接触式或非接触式智能卡**, 其软件和硬件架构是相同的,唯一的区别是它们对外的接口有所不同,普通的智能卡具有ISO/IEC 7816-3 (接触式)或天线(非接触式)接口,而安全单元在这些接口之外，还通过一个直接的接口与NFC控制器连接, 主要的2种接口有:

+ S2C : (SignalIn/SignalOut Connection Interface) 通常用于nfc芯片与其内部集成的SE单元通信
+ SWP : (Single Wire Protocol) 通常用于 NFC-SD 和 NFC-SIM 方案

**Host Cpu 也可以子通过 ISO 7816 协议和SE单元交互**

SE单元的应用接口是 APDU命令 (APDU是智能卡与智能卡读卡器之间传送的信息单元), 对应的协议是 ISO7816-4

安全单元具备与常规智能卡同样的高安全标准, 它可以提供安全存储，安全执行环境和基于硬件的加密算法, 安全单元芯片用于对存储数据的读取和操作，并可以抵御各种攻击。其芯片，操作系统和设计流程都经过高安全标准的评估和认证. 例如智能卡芯片通用标准保护条例（Common Criteria protection profiles for smartcardmicrochips）等. 因此，安全单元满足安全相关的应用，例如支付和门禁管理系统

####2.3 SWP 接口

SWP连接方案基于ETSI(欧洲电信标准协会)的SWP标准，该标准规定了SIM卡和NFC芯片之间的通信接口

在SWP方案中，接口界面包括三根线：VCC、GND、SWP，其中SWP 一根信号线上基于S1信号(电压调制)和S2信号(负载调制,电流调制)的叠加实现了UICC(Universal Integrated Circuit Card，通用集成芯片卡)和CLF(Contactless Front end,非接触前端)之间的全双工通讯

![SWP 接口](/images/nfc/swp-interface.png)

+ 电压调制信号S1 :  传送CLF到UICC的数据
+ 电流调制信号S2 :  传送UICC到CLF的数据

SWP协议要求UICC的工作电压为1.8～3.3 V, 有3种传输速率: 212kbps / 424 kbps / 848 kbps，对数据位进行扩展之后，传输速率可以达到 1696 kbps

SWP接口可工作于两种功耗模式之下： 低功耗模式 / 全功耗模式

#####2.3.1 SWP 协议

SWP协议是关于物理层和数据链路层的协议 :

+ 物理层负责UICC和CLF之间物理链路的激活、保持、解除等工作
+ 数据链路层分为MAC(媒介访问控制)层和链路控制层
   + 在MAC层采用位填充的成帧方法
   + 链路控制层包括3种类型的帧协议：ACT协议、SHDLC协议以及CLT(非接触通道)协议, 在SWP接口的设计中，使用了前两种协议

ACT协议是接口激活协议，用于激活SWP接口。在没有射频场时，SWP接口处于去激活状态。在标签模式下，感应到外界存在射频场后，NFC芯片被激活，UICC收到NFC芯片的高电平信号后，使用ACT帧建立物理链路的连接。

SHDLC协议是ISO制定的高级数据链路控制规范的简单版本，也是面向位的同步链路。该协议主要用来传输交互的数据信息，其信息帧承载上层HCP(主机控制协议)的包数据。此外，SHDLC协议还具有流控管理、错误检查、出错后数据重传等功能。为了保证数据的正确发送与接收，兼容NFC芯片与UICC不同速率传输的通信能力，在使用SHDLC协议通信前，首先要建立数据链路，双方协商滑动窗口的大小。

####2.4 UICC 接口

UICC (Universal Integrated Circuit Card, 通用集成电路卡), 是定义了物理特性的智能卡的总称，UICC和终端的接口都是标准的， 其物理接口如下

![uicc 接口](/images/nfc/uicc-interface.png)

UICC的应用接口为 APDU 指令， 协议为 ISO7816-4

UICC可以包括多种逻辑应用：

+ SIM (Subscriber Identity Module) 用户标识模块, 通常被称为 SIM 卡
   + 指UICC卡上存储GSM用户签约信息的应用
   + 包含 国际移动用户标识IMSI
   + 包含 移动用户ISDN号
   + 包含 加密密钥Ki，加密算法A3， A8
   + 包含 移动国家码MCC
   + 包含 移动网络码MNC
+ UIM (User Identity Module) 用户识别模块, 通常被称为 UIM 卡
   + 联通公司倡导的移动通信终端用户识别及加密技术
   + 包含用户识别信息和鉴权信息
   + 包含 业务信息
   + 包含 与移动台相关的信息
+ USIM (Universal Subscriber Identity Module) 通用用户标识模块， 通常被称为 USIM 卡
   + 指UICC卡上存储通用移动通信系统的参数的应用
   + 包含 国际移动用户标识IMSI
   + 包含 移动用户ISDN号
   + 包含 加密密钥CK， 完整性密钥IK
   + 包含 短消息
   + 包含 短消息参数
   + 包含 多媒体消息业务
   + 包含 MMS用于优选信息
+ ISIM (IP Multimedia Service Identity Module) IP多媒体业务标识模块
   + 可以和 SIM 或者 USIM 共存在一张 UICC 卡上
   + 包含 用户私有标识
   + 包含 公共用户标识
   + 包含 归属网络域URI
   + 包含 长期加密

###2.5 NFC-SIM SWP连接方案示例

NFC-SIM 的SWP连接方案使用SIM卡的3个引脚连接NFC芯片：C1(VCC)、C5(GND)、C6(SWP), (SIM卡不使用 C4和C8引脚)如下图

![NFC-SIM SWP连接方案](/images/nfc/nfc-swp-sample.png)

UICC的电源由CLF来提供， 因此， 即使是手机的电池耗尽， 依然可以通过外部电磁感应来给CLF和UICC供电

支持SWP的SIM卡必须同时支持两个协议栈：“ISO7816” 和 “SWP协议栈“，这需要SIM卡的COS(片上操作系统)是多任务系统， 并且由于SIM卡需要单独管理这两个协议栈。SWP方案加入SIM卡系统后，不能影响ISO7816接口。举个例子，SIM卡有8个引脚，RST引脚用于复位SIM卡的ISO7816接口，SWP方案加入SIM卡后，RST引脚的Reset信号对SWP接口没有作用，SWP接口通过SWP引脚复位

###3. 主机卡模拟

主机卡模拟是在支持NFC功能的手机上进行NFC卡模拟的新方法，由黑莓首先引进， Android 4.4也加入了支持， 主机卡模拟一般只支持ISO14443-4以上的卡

![nfc 主机卡模拟示例](/images/nfc/nfc-hce-sample.png)

在android上面， 主机卡模拟遵循NFC Forum 的 ISO-DEP 规范(基于ISO/IEC 14443-4)， 并且处理 APDU cmd (ISO/IEC 7816-4定义)， android强制要求必须能够模拟 ISO/IEC 14443-3 Type A， ISO/IEC 14443-3 Type B 是可选的

[android的主机卡模式官方介绍](https://developer.android.com/guide/topics/connectivity/nfc/hce.html#HceServices)

###4. 虚拟卡模拟/主机卡模拟 对比

虚拟卡模拟依赖SE提供较高的安全性，没有密钥，根本无法完成对卡的写操作， 因此需要SE的操作密钥, 但是，这些密钥都掌握在手机厂商(内置SE的情况下)和运营商(NFC-SIM的情况下), 鉴于NFC卡模拟的前景， 这些厂商都想要分一杯羹， 绝不会将密钥共享给其它厂商(尤其是竟争对手)， 这就导致了SE只能被一家厂商所掌握，因此实现对安全单元访问面临着不小的障碍

主机卡模拟不再依赖SE， 而在Host cpu上运行应用, 任何开发者都可以开发基于卡模拟技术的应用程序, 为开发与现存具有固定读写器的基础设施交互的应用程序的提供了可能. 也就是说, 开发者可以开发基于手机的门禁，支付，公共交通和票务应用程序，用于使用RFID票卡和智能卡的系统中

软件卡模拟的另外一个好处是可以实现与不具备全功能点对点模式的NFC设备通讯, 例如，Android系统仅支持Android Beam来实现点对点通讯, 然而Android Beam是基于谷歌的NDEF推协议（NDEF Push Protocol NPP）和简单NDEF交换协议 （Simple NDEF ExchangeProtocol SNEP），在两个NFC手机接触时仅能实现单方向单条的消息通讯。因此，软件卡模拟可以作为NFC手机之间点对点通讯的替代方式

软件卡模拟突破了现有的基于安全单元解决方案的束缚，为开发人员提供了新的机会, 该技术可以方便的将具有NFC功能的手机和现存的非接触智能卡系统集成在一起。而且与点对点模式相比，读写器模式设备和卡模拟设备之间的通讯更具优势，因为读写器模式的应用比点对点模式更为普遍, 但是这些优点的代价是安全上的问题 :

+ 手机应用处理器中运行的应用程序不具有安全单元中的数据安全存储和可信度执行环境。除非应用处理器自己能够提供某种形式的可信计算技术。但是目前的绝大多数手机没有这个功能。
+ 没有安全存储功能，因此卡模拟应用程序不得不自行保存敏感的数据
+ 另外软件卡模拟还存在技术和安全相关的限制。在NXP的NFC控制器中，基于软件的卡模拟只能使用一段范围内的UID(被保留为随机号码的UID号码段）

软件卡模拟也可以实现某种程度上的安全应用。一个关键是将“虚拟卡（virtual card）”存放在一个安全的远程的地点，而手机仅仅充当一个访问代理
