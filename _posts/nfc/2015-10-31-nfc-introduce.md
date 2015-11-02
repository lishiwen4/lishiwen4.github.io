---
layout: post
title: "NFC 概述"
description:
category: nfc
tags: [nfc]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. NFC

NFC (Near Filed Communication, 近场通信)也叫近距离无线通信技术，由RFDI技术， 磁条卡技术融合发展而来，这一技术标准最早由sony和philips在2002年推出， 在2004年sony，philips， nokia， 等公司共同成立了名为 NFC Forum 非盈利性组织(类似于 Wi-Fi Alliance)来推广NFC技术， 负责制定技术标准和认证测试

![nfc](/images/nfc/nfc-tech.png)

**NFC和RFID在物理层有相似之处，但其本身和RFID是两个领域的技术，RFID仅仅是一种通过无线对标签进行识别技术，而NFC是一种无线通信方式，这种通信方式是交互的**

###2. NFC 

+ 有效距离约为 4cm
+ 工作频率为 13.56MHz
+ 传输速率有 106Kbps， 212Kbps， 424Kbps 3种

###3. NFC 的工作模式

虽然NFC的前身 RFID 和 词条卡 中有着明确的 “卡/读卡器” 的概念， 但是NFC协议中希望能够淡化这一概念， 而是统称为NFC device， 每一个NFC device支持如下的一种或者多种工作模式：

+  Reader/Writer 模式: 简称R/W， 作为NFC的读写器, 可以读写另一个无源的NFC标签
+  P2P 模式 : 简称P2P，即两个都工作在这一模式下的设备直接进行交互
+  Card Emulation 模式 : 简称CE， 即把一个支持NFC的设备模拟为一个smart card， 用于实现手机支付，门禁

###4. NFC的射频规范

NFC的3中工作模式都依赖于NFC的的无线射频技术， 其相关的标准有3种

+ ISO 18092 : NFCIP-1 (NFC Interface and Protocol1) 定义了NFC RF层的工作流程
+ ISO 14443 : 非接触式IC卡标准， 在RF层面上定义了如何与不同的非接触式IC(RFID, NFC Tag, smart card)卡交互， 它定义了Type A， Type B种非接触式IC卡
   + Type A : 由philips制订， 后归从philips独立出的NXP所有， 商标名为 MIFARE， 市场占有率约为70%
   + Type B : 由其它公司制订， 主要在法国市场
+ Felica : 也称Type F， 由sony开发， 没有成为ISO标准， 只是作为日本的工业标准 JIS 6319-4

Type A， B， F 的主要区别在于RF层的信号调制解调方法， 传输速率以及数据编码率上面

**ISO 18092 比 ISO 14443 晚， 兼容且包含 ISO 14443**

###5. NFC技术框架

NFC forum 所定义的技术框架如下

![nfc 技术框架](/images/nfc/nfc-forum-stack.png)

####5.1 底层的comon规范

最底的3层是RF层的规范：

+ Analog Specifications : 该规范描述了NFC设备的RF层的电气特性
   + NFC-A, NFC-B, NFC-F
+ Digital Protocol Specification : 该规范在 ISO-19092 ISO-14443以及JIS X6319-4 之上， 定义了NFC设备之间的数字通信协议， 使得基于不同的底层协议(例如Type A或者Type F的NFC设备或者NFC与其它使用ISO 18092等规范的设备之间)之间能够交互
   + NFC-A, NFC-B, NFC-F 编码规范
   + 基于ISO 18092的NFC-DEP (用于P2P mode)， 基于ISO 14443的 ISO-DEP (用于CE mode和 R/W mode)， 2者差异不大
+ NFC Activities Specification : 该规范为运行各协议对应的协议栈提供支持(比如P2P模式下， 两个设备如何建立连接)

####5.2 CE mode 的相关规范

CE mode下， 直接在RF层的 NFC Activities Specification 规范上开发应用

####5.3 P2P mode 的相关规范

+ LLCP : P2P协议栈的最底层， 用于处理物理地址寻址，链路管理， 以及数据传输
+ SNEP : 	该协议使得两个NFC device之间能够直接交换NDEF消息
+ CHP : Connection Handover Protocol, 用于通过NFC来配置wifi， bluetooth的连接
+ Protocol Bindings : 用于让NFC支持更高层次的协议， 例如IP协议， OBEX协议

在SNEP协议或者其它的协议之上， 可以开发应用程序

####5.4 R/W mode 的相关规范

+ NDEF ： 根据NFC Forum的规定， R/W模式下， NFC设备之间每一次交互的数据都会封装在一个NDEF Message之中
+ RTD ：  RTD规范定义了NFC Forum规定的七大数据类型中的 Well known type 和 External type