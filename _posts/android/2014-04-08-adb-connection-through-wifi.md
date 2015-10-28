---
layout: post
title: "wifi网络上建立adb连接"
description:
category: android
tags: [android, wifi, debug]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. wifi连接adb
  
adb能够通过tcp连接， 使用命令：  
  
	adb tcpip 5555
    
即可重启adbd， 并且监听5555号tcp端口  
  
将pc与android设备连接在同一个局域网中， 假设android设备的ip为192.168.1.244, 则使用  
  
	adb connect 192.169.244：5555
    
即可建立adb连接， 然后， 即可执行  adb命令， 例如
  
	adb shell 
    
即可连接 adb shell
