---
layout: post
title: "linux-PM wakeup count"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. wakeup count  
  
wakeup count的实现存在于（drivers/base/power/wakeup.c），实现了两个功能:  
  
1. 判断系统当前正在处理的wakeup event 计数(wakeup events in progress)是否为空, 并且返回系统中已处理完成的wakeup event的计数(registered wakeup events)  
2. 保存系统中当前已经处理完成的wakeup event的计数(registered wakeup events)到saved_count中  
  
###2. wakeup count 接口  
  
对应这两个功能，提供了两个kernel API 以及一个sysfs属性文件：  
  
+ bool pm_get_wakeup_count(unsigned int *count, bool block)；

当系统中无正在处理的wakeup event(wakeup events in progress计数为0)时， 返回true， 否则返回false  
count携带系统中已经处理完的wakeup event的个数(registered wakeup events计数)  
当参数block为true时， 等到当前无正在处理的wakeup event时才返回，单返回值不一定为true， 因为返回之前可能产生了wakeup event， 当block为false时会立即返回  
事实上，该函数的实现中，当参数block为true时，会在wakeup_count_wait_queue队列上等待， 然后被pm_relax()唤醒， 再检查wakeup events in progress计数， 若不为0， 则继续等待
  
+ bool pm_save_wakeup_count(unsigned int count)；
>设置saved_count计数， 一般应该调用pm_get_wakeup_count()获取系统中已经处理完的wakeup event计数， 然后调用pm_get_wakeup_count()来设置saved_count计数  
  
另外， wakeup count还在sysfs中提供一个文件"/sys/power/wakeup_count"供用户空间使用, 该文件的wakeup_count_show()/wakeup_count_store()方法分别使用上述的pm_get_wakeup_count()/pm_save_wakeup_count来实现, 需要注意的是， 读取该文件时会阻塞(见pm_get_wakeup_count()的解释)，若读取失败说明在返回前系统又接受到了wakeup event， 此时应该重新读取  
  
###3. wakeup count的功能  
  
任何想要发起电源状态切换的实体(例如用户空间的电源管理进程， 或者内核空间的线程如autosleep),在发起状态切换之前， 先读取系统的wakeup count， 若检测到没有正在处理的wakeup event， 则将读取到的wakeup count写入到saved count， 然后发起状态切换， 在suspend的过程中， 会调用pm_wakeup_pending()来检查是否存在wakeup event待处理  
  
###4. 用户空间使用wakeup count来suspend  
  
伪码如下：  
  
	do {
		ret = read(&cnt, "/sys/power/wakeup_count");
        		if (ret) {
        			ret = write(cnt, "/sys/power/wakeup_count");
        		} else {
        			countine;
        		}
    	} while (!ret);
    
    	write("mem", "/sys/power/state");  
