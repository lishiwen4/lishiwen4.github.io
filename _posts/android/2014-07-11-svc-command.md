---
layout: post
title: "svc 命令"
description:
category: android
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. svc命令  
  
svc命令，位置在/system/bin目录下，用来管理电源控制，无线数据，WIFI  
  
###2. svc power  
  
####2.1 设置屏幕在某种条件下常亮

		svc power stayon [true|false|usb|ac|wireless]
    
####2.2 设备重启
  
    	svc power reboot [reason]  
    
####2.3 设备关机
  
    	svc power shutdown
  
###3. svc data  
  
####3.1 打开/关闭移动数据
  
		svc data [enable|disable]
  
####3.2 设置移动数据优先于wifi
  
       	svc data prefer
  
###4. svc wifi  
  
####4.1 打开关闭wifi
  
		svc wifi [enable|disable]
      
####4.2 wifi数据优先移动数据
  
       	svc wifi prefer  
        
###5. svc usb  
  
####5.1 获取当前usb功能
  
		svc usb setFunction [function]
  
####5.2. 获取当前usb功能
  
		svc usb getFunction