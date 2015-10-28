---
layout: post
title: "linux虚拟网络接口 —— 多网卡绑定"
description:
category: network
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux 虚拟网络接口

在linux的网络设备驱动框架里面， 使用一个net_device来代表一个网络设备接口， 因此， 一个物理网卡对应着一个net_device结构

虚拟网络接口是指一个网络接口的net_device没有直接对应的物理设备

###2. 多网卡绑定

http://blog.chinaunix.net/uid-198791-id-4231902.html