---
layout: post
title: "android wifi 组播"
description:
category: wifi
tags: [wifi, linux, android]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. android wifi 组播

android 上出于省电的考虑， 默认会过滤掉接收到的组播包， 若需要接收组播包， 则需要持有 MultiCast 锁

	// create MulticastLock
	WifiManager wifiManager = (WifiManager) getSystemService(Context.WIFI_SERVICE);
	MulticastLock multicastLock = wifiManager.createMulticastLock("multicast.test");

	// MulticastLock 加锁
	multicastLock.acquire();

	// MulticastLock 释放
	multicastLock.release();

APP还需要在Manifest中添加权限

	&lt;uses-permission android:name="android.permission.CHANGE_WIFI_MULTICAST_STATE"/&gt;

###2. Multicast 的实现

	MulticastLock.acquire()	-->
		WifiService.acquireMulticastLock()	-->
			WifiStateMachine.stopFilteringMulticastV4Packets()	-->
				WifiNative.startFilteringMulticastV4Packets()	


	MulticastLock.release()	-->
		WifiService.releaseMulticastLock()	-->
			WifiStateMachine.startFilteringMulticastV4Packets()	-->
				WifiNative.stopFilteringMulticastV4Packets

WIfiNative 实际上是调用 wpa_supplicant 的cmd，组播包过滤要依靠wifi driver来实现， android上的wifi driver必须实现wpa_supplicant 的 DRIVER RXFILTER-xxx" command

其使用规则为

	// 停止过滤 Num 类型的数据包
	DRIVER RXFILTER-STOP
	DRIVER RXFILTER-ADD Num
	DRIVER RXFILTER-START

	// 继续过滤 Num 类型的数据包
	DRIVER RXFILTER-STOP
	DRIVER RXFILTER-REMOVE Num
	DRIVER RXFILTER-START

Num类型的取值有

0. 单播包
1. 广播包
2. ipv4组播包
3. ipv6组播包

例如， 在 WifiNative 中有

	public boolean startFilteringMulticastV4Packets() {
		return doBooleanCommand("DRIVER RXFILTER-STOP")
			&& doBooleanCommand("DRIVER RXFILTER-REMOVE 2")
			&& doBooleanCommand("DRIVER RXFILTER-START");
	}

###3. 组播地址

+ 224.0.0.0～224.0.0.255  为预留的组播地址（永久组地址），地址224.0.0.0保留不做分配，其它地址供路由协议使用
+ 224.0.1.0～224.0.1.255  是公用组播地址，可以用于Internet
+ 224.0.2.0～238.255.255.255 为用户可用的组播地址（临时组地址），全网范围内有效
+ 239.0.0.0～239.255.255.255 为本地管理组播地址，仅在特定的本地范围内有效