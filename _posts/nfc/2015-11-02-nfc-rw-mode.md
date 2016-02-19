---
layout: post
title: "NFC RW 模式"
description:
category: nfc
tags: [nfc]
mathjax: 
chart:
comments: false
---

###1. NFC R/W mode

NFC R/W mode 是指NFC device对NFC tag进行读写操作， 在R/W 模式中， 交互操作的发起方只能是 NFC reader/writer， 因此称作 Active Device，NFC tag需要NFC reader为其提供电能进行操作， 因此称作Passive Device

###2. NFC Tag的类型

NFC Forum定义了四种类型的tag， 分别为 Type 1，Type 2，Type 3，Type 4， 差别在于存储空间的大小， 数据传输率，以及底层的协议， 它们的差异如下：

![NFC Tag的差异](/images/nfc/nfc-tags.png)

###3. 数据交换的基本结构

NFC Forum定义了2种通用的数据结构用于NFC Device之间传递数据：

+ NDEF
+ NFC Record

####3.1 NDEF (NFC Data Exchange Format)

NFC Forum定义， R/W 模式下， NFC设备之间的每一次的交互数据都会封装在一个NDEF Message中， 每一个NDEF Message中可以包含一个或者多个NFC Record， 真正的数据包含在NFC Record中

![NDEF](/images/nfc/NDEF.png)

+ NFC Record的 MB 成员置“1”时标识该NFC Record属于NDEF Message中的第一个NFC Record
+ NFC Record的 MB 成员置“1”时标识该NFC Record属于NDEF Message中的最后一个NFC Record

####3.2 NFC Record

NFC Record的结构如下：

![NFC Record](/images/nfc/NFC-Record.png)

+ MB : 标识为第一个NFC Record
+ ME : 标识为最后一个NFC Record
+ CF : 标识该Record是否为分片Record
+ SR : 标识是否为Short Record (payload小于255Byte)， 若是， 则该Record没有 Payload Length 1 ~ 3 这几个字段
+ IL : 标识是该record中否有 ID Length 和 ID 这两个字段
+ TNF : 标识 payload 的类型， 其取值和意义如下， 除最后的 Reserved 外， NFC 都支持
   + 0x00 Empty 表示 Record 中没有数据， 是一个空的Record 
   + 0x01 NFC Forum Well-Known Type 表示是由NFC Forum定义的常用的数据， 例如 URI， TEXT等， 遵守RTD规范
   + 0x02 MIME 表示MIME类型，遵守RFC2046规范，由Type 字段指明子类型
   + 0x03 Absolute URI 表示绝对URI， 遵守RFC3986规范
   + 0x04 NFC Forum External Type 也遵守NFC RTD规范
   + 0x05 Unkown 代表Record的数据类型未知
   + 0x06 Unchanged 表示NFC Record分片， 当数据太大， 需要要存放在多个NFC Record中时， 除第一个Record外， 其它的Record都需要设置TNF字段为 Unchanged
   + 0x07 Reserved
+ Type length : 指明record 中 Type字段的长度
+ Payload length 3 ~ 0 : 指明 Payload 字段的长度
+ ID Length : 指明 ID 字段的长度， 若IL字段未设置， 则 ID 和 ID Length 字段都不存在
+ Type : TNF 字段指示payload的类型， 而该字段指明了payload的子类型例如， Well-Known Type时有
   + "U" 代表为 URI Record Type 
   + "T" 代表为 Text record Type
   + "Sig" 代表为 Signature Record Type
   + "Sp" 代表为 Smart Poster Record Type
   + "Gc" 代表为 Generic Control Record Type
   + "Di" 代表 Device information
   + "Hr" 代表 Handover Request 	
   + "Hs" 代表 Handover Select
   + "Hc" 代表 Handover Carrier
+ ID : 需要配合URI类型的payload一起使用， 它使得一个NFC Record能够通过ID来指向另外一个NFC Record

###4. URI Record Type

URI Type Record 的Payload字段的定义如下

+ 第一个字节存储 Identifier Code (即前缀)， 取值如下
   + 0x00 : 无前缀 
   + 0x01 : "http://www"
   + 0x02 : "https://www"
   + 0x03 : "http://"
   + 0x04 : "https://"
   + 0x05 : "tel:"
   + 0x06 : "mailto:"
   + 0x07 : "ftp://anonymous:anonymouse@"
   + 0x08 : "ftp://ftp."
   + 0x09 : "ftps://"
+ 后续字节 : URI 的值

以一个 URI 消息 “http:/www.nfc.com” 为例， 其封装为 NDEF后， 各字段的值如下

![URI record 实例](/images/nfc/NFC-Record-sample-uri.png)

+ 因为数据较少， 只需要一个Record， 因此 MB=1 ME=1， SR=1， CF=0
+ 不需要 ID 和 ID Length 字段， 因此 IL 为0
+ URI 为 NFC Forum 定义好的数据类型， 因此TNF 为 0x01
+ Type 为“U” 代表 URI record， 因此 Type Length 为 1
+ 前缀为 “http://www”， 因此Identifier Code为1
+ URI 值为 “nfc.com”， 7byte， 加上Identifier Code, 1Byte,  因此， Payload Length 为8

###5. Text Record Type

Text Record Type 的payload字段的定义如下:

+ 第一个字节为Status字节
   + bit[7] ： 为1表示后面存储的Text为UTF-8编码， 否则为UTF-16编码
   + bit[5:0] ： 指明语言码字段的长度
+ 语言码字段： 长度由 Status[5:0] 指明， 可选值有 “en”， “jp” 等
+ 最后是Text字段

以一个 “Hello World” 字串为例， 其封装为NDEF 后， 各字段值如下

![URI record 实例](/images/nfc/NFC-Record-sample-text.png)
