---
layout: post
title: "android上wifi扫描间隔"
description:
category: wifi
tags: [wifi, linux]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. android wifi 循环扫描

在android系统中， 在不同的场景下， 不同的功能模块会发起循环扫描

大部分扫描功能需要调用WPAS的接口来完成， 可先阅读 “WPAS” 分类中的 “WPAS 中的循环扫描”

###2. 亮屏时Wifi Settings界面的扫描

**只要打开wifi， 进入wifi settings界面， 无论是否连接了AP， 都会开始定时扫描**

相关源码为 “pacakes/app/Settings/src/com/android/settings/wifi/WifiSettings.java”
Wifi Setting 中由 class  Scanner来负责发起扫描， calss Scaner继承了Handlerl来实现定时扫描:

+ 延时向自己发送一个空message
+ 接收到message后， 立即调用WifiManager执行一次扫描， 然后再次延时向自己发送空message
+ 向自己发送空message的延时值即为扫描的间隔， 默认为10s  (int WIFI_RESCAN_INTERVAL_MS = 10 * 1000;)

其向自己延时(WIFI_RESCAN_INTERVAL_MS)发送message， 接收到mesage后， 执行一次扫描来实现定时扫描, 其提供了3个接口

	Scanner.pause()		//移除还未被发送的空message， 中止定时扫描
	Scanner.resume()		//向自己发送一个空的message， 恢复扫描
	Scanner.forceScan()	//移除还未被发送的的空message， 并立即发送一个空message， 导致马上进行一次扫描

+ 在wifi Settings的onpause()阶段， 中止定时扫描
+ 在wifi Settings的onresume()阶段， 恢复定时扫描
+ 点击wifi Settings菜单的scan选项， 则进行一次强制扫描
+ 关闭wifi会中止定时扫描
+ 打开wifi会恢复定时扫描
+ 在wifi 处于DHCP状态时， 中止定时扫描， 变为其它状态后再恢复定时扫描

###3.亮屏时 wpa_supplicant 的周期扫描

在亮屏， 未连接wifi， WPAS中保存了AP的情况下，WPAS会周期扫描，android上调整间隔为15s， 可在 frameworks/base/core/res/res/values/config.xml 文件中修改。

        <integer translatable="false" name="config_wifi_supplicant_scan_interval">15000</integer>

###4. 灭屏时 PNO 扫描

如果未连接AP， 但是保存有AP， 在灭屏后， 会开启PNO扫描， 标准的PNO扫描机制为

+ 以10秒时间间隔扫描6次
+ 以20秒扫描间隔扫描6次
+ 以40秒时间间隔扫描6次
+ 以80秒时间间隔扫描6次
+ 以160秒时间间隔扫描6次
+ 以后将以320秒时间间隔进行扫描

###5. autojoin 扫描

如果打开了autojoin机制， 且保存了AP， 无论是亮屏还是灭屏， autojoin会周期扫描， 间隔为10s 

###6. batch scan

batch scan 通过从applicantion offloading scan 到 wifi firmware 来达到节省电力的目标， applicantion可以请求wifi firmware在指定的时间间隔内发起指定次数的扫描， wifi firmware根据限定的频率持续进行扫描并且缓存扫描结果， 定期将扫描结果返回给 ACPU

batch scan 需要wifi driver 和 wifi firmware支持， wifiManager提供了接口来使用batch scan，例如：

+ requestBatchedScan()
+ isBatchedScanSupported()
+ stopBatchedScan()
+ getBatchedScanResults()
+ pollBatchedScan()

**batch scan主要应用于wifi 辅助定位**