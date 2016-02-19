---
layout: post
title: "adb manual"
description:
category: android
tags: [android, wifi, debug]
mathjax: 
chart:
comments: false
---

###1. 使用tcp/ip网络连接adb
  
adb能够通过tcp连接，将pc与android设备连接到同一个局域网中， 然后执行  
  
	/* step 1. 重启手机的 adbd， 并且 adbd 将会监听5555号tcp端口  */
	$ adb tcpip 5555
	
	/* step 2. 可建立 adb 连接, 假设手机的ip为 192.168.1.244 */    
	$ adb connect 192.168.1.244：5555
  
	/* step 3. 检查 adb devices, 因为同时有usb和网络的连接， 所以一台手机会被识别为2个adb设备 */
	$ adb devices
	F5AZCY05K459		device
	192.168.1.244:5555		device

	/* step 4. 执行 adb cmd */
	adb shell

	/* step 4. 若有多个 adb device， 执行 adb command 时还需要指定 device*/
	adb -s 192.168.1.244:555 shell

	/* step 5. 断开 tcp/ip 网络连接 */
	adb disconnect 192.168.1.244:5555
  
	/* step 6. 切换到 usb 连接的方式 */
	adb usb

如果不能使用usb连接的话，则无法在pc上执行上述的 step 1 和 step 6，此时可以在手机上直接执行如下的步骤(可以使用模拟终端， 也有现成的apk例如 adbWireless)

	/* step 1 */
	setprop service.adb.tcp.port 5555
	stop adbd
	start adbd

	/* step 6 */
	setprop service.adb.tcp.port -1
	stop adbd
	start adbd
