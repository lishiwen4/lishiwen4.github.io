---
layout: post
title: "linux suspend resume 流程"
description:
category: linux-driver
tags: [linux, pm, driver]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 如何让linux进入睡眠  
  
用户可以通过读写sys文件 "/sys/power/state" 控制系统进入休眠, 例如， `echo mem > /sys/power/state`可以让系统休眠(状态保存到内存), 可选的state如下：  

1. state : Freeze / Low-Power Idle
   + ACPI state:     S0  
   + String:         "freeze"     
2. State : Standby / Power-On Suspend  
   + ACPI State:     S1  
   + String:         "standby"     
3. State : Suspend-to-RAM  
   + ACPI State:     S3  
   + String:         "mem"     
4. State : Suspend-to-disk  
   + ACPI State:     S4  
   + String:         "disk"  
        
具体可用的state在每个系统上有可能不一样， 使用`cat /sys/power/state`可以获取系统支持的sate， 例如在fedora 19上， 支持的state为“freeze mem disk”， 而在ubuntu 12.10上， 支持的state为“mem disk”  

###2. suspend的入口函数

在"kernel/power/suspend.c"中定义了suspend的入口函数 pm_suspend() 

	int pm_suspend(suspend_state_t state)
	{
    		......
			error = enter_state(state);
    		......
	}
	EXPORT_SYMBOL(pm_suspend);

调用enter_state进入suspend状态， 具体的状态根据state值而定， 如果执行成功的话， 一直到系统退出suspend的状态后此函数才退出。  

###3. 电源管理操作的两个重要结构   

	struct dev_pm_ops {
		int (*prepare)(struct device *dev);
		void (*complete)(struct device *dev);
		int (*suspend)(struct device *dev);
		int (*resume)(struct device *dev);
		int (*freeze)(struct device *dev);
		int (*thaw)(struct device *dev);
		int (*poweroff)(struct device *dev);
		int (*restore)(struct device *dev);
		int (*suspend_late)(struct device *dev);
		int (*resume_early)(struct device *dev);
		int (*freeze_late)(struct device *dev);
		int (*thaw_early)(struct device *dev);
		int (*poweroff_late)(struct device *dev);
		int (*restore_early)(struct device *dev);
		int (*suspend_noirq)(struct device *dev);
		int (*resume_noirq)(struct device *dev);
		int (*freeze_noirq)(struct device *dev);
		int (*thaw_noirq)(struct device *dev);
		int (*poweroff_noirq)(struct device *dev);
		int (*restore_noirq)(struct device *dev);
		int (*runtime_suspend)(struct device *dev);
		int (*runtime_resume)(struct device *dev);
		int (*runtime_idle)(struct device *dev);
	};
    
	struct platform_suspend_ops {
		int (*valid)(suspend_state_t state);
		int (*begin)(suspend_state_t state);
		int (*prepare)(void);
		int (*prepare_late)(void);
		int (*enter)(suspend_state_t state);
		void (*wake)(void);
		void (*finish)(void);
		bool (*suspend_again)(void);
		void (*end)(void);
		void (*recover)(void);
	};
    
###4. suspend的几个阶段  
  
如果在编译内核时打开了”CONFIG_PM_DEBUG“， 则会在出现“/sys/power/pm_test”属性文件， 在pm_test中定义了一下几个阶段：  
  
	int pm_test_level = TEST_NONE;
    
	static const char * const  pm_tests[__TEST_AFTER_LAST] = {
       [TEST_NONE] = "none",
       [TEST_CORE] = "core",
       [TEST_CPUS] = "processors",
       [TEST_PLATFORM] = "platform",
       [TEST_DEVICES] = "devices",
       [TEST_FREEZER] = "freezer",
	};
  
向“/sys/power/pm_test”中写入0～5， 可以使得系统进入对应的（0对应"none"， 5对应"freezer"）suspend阶段，进入suspend状态时， 依次所经历的阶段为 'freezer' -> 'devices' -> 'processors' -> 'core' 。  
`cat /sys/power/pm_test`可以获取系统当前所处的阶段(使用‘[]’括起来的状态)。   

###5. suspend各阶段工作  
  
suspend和resume时， 各个阶段的工作如下：  
  
	/*suspend-[freezer]*/
    
			pm_suspend() -->
    			enter_state() -->
	[1]				suspend_sync() -->
            		suspend_prepare() -->
	[2]        			pm_prepare_console() -->
	[3]            		pm_notifier_call_chain() -->
	[4]            		suspend_freeze_processes() -->
    				suspend_prepare() <--
               
1.同步文件系统， 将高速缓存内容回写到磁盘  
2.分配一个控制台， 用于输出消息  
3.发送PM_SUSPEND_PREPARE消息，激活pm注册的内核通知链（内核通知链一般用于内核子系统之间的消息通知）  
4.冻结任务, 包括用户进程， usermode helper进程（内核中启动的用户进程），以及可以冻结的内核线程
       
    
	/*suspend-[device]*/
    
    				suspend_devices_and_enter() -->  
	[5]					suspend_ops->begin() -->
	[6]					suspend consle() -->
    					dpm_suspend_start() -->
	[7]					dpm_prepare() -->
	[8]	                    		dpm_suspend() -->
    					dpm_suspend_start() <--
                            
5.调用suspend_ops(板级电源管理操作)的begin回调    
6.suspend consle子系统， 此后，不能输出内核消息  
7.调用pm_ops(电源管理操作)的prepare回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->prepare)    
8.调用pm_ops(电源管理操作)的suspend回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->suspend)  
          

	/*suspend-[platform]*/    
    
		suspend_enter() -->
	[9]	      suspend_ops->prepare() -->
		dpm_suspend_end() -->
    [10]                    	dpm_suspend_late() -->
    [11]                        dpm_suspend_noirq() -->
    	                    dpm_suspend_end() <--
    [12]                    suspend_ops->prepare_late() -->
                            
9.调用suspend_ops(板级电源管理操作)的prepare回调  
10.调用pm_ops(电源管理操作)的suspend_late回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->suspend_late回调)    
11.调用pm_ops(电源管理操作)的suspend_noirq回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->suspend_noirq回调)    
12.调用suspend_ops(板级电源管理操作)的prepare_late回调 
      

    /*suspend-[cpu]*/
    
    [13] 					disable_nonboot_cpus() -->
    
13.关闭多核cpu中的非启动cpu  
      
    
    /*suspend-[core]*/						
    [14]                    arch_suspend_disable_irqs() -->
    [15]					syscore_suspend() -->
    [16]                    suspend_ops->enter() -->
    
14.禁止中断  
15.调用所有system core（例如cpu频率， timer等）注册的supend回调   
16.调用suspend_ops(板级电源管理操作)的enter回调, 真正进入suspend,当被唤醒时， 才返回  
         
    
    /*resume-[core]*/
    
    [17]					syscore_resume() -->
    [18]                    arch_suspend_enable_irqs() -->  
    
17.调用所有system core（例如cpu频率， timer等）注册的resume回调, 执行的时候，只有一个cpu核工作且中断关闭  
18.打开中断  
  
    
    /*resume-[cpu]*/
    
    [19] 					enable_nonboot_cpus() -->

19.打开多核cpu中的非启动cpu  
  
    
    /*resume-[platform]*/
    
    [20]					suspend_ops->wake() -->
    	                    dpm_resume_start() -->
    [21]	                   	dpm_resume_noirq() -->
    [22]	                    dpm_resume_early() -->
    	                    dpm_resume_start() <--
    [23]                    suspend_ops->finish() -->
    					suspend_enter() <--
                            
20.调用suspend_ops(板级电源管理操作)的wake回调  
21.调用pm_ops(电源管理操作)的resume_noirq回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->resume_noirq回调)
22.调用pm_ops(电源管理操作)的resume_early回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->resuem_early回调)   
23.调用suspend_ops(板级电源管理操作)的finish回调  
      
    /*resume-[device] */
    
    [24]				suspend_ops->recover() -->
    	                dpm_resume_end() -->
    [25]					dpm_resume() -->  
	[26]					dpm_complete() -->  
    	                dpm_resume_end() <--
    [27]                resume_console() -->
    [28]                suspend_ops->end()-->
    	            suspend_devices_and_enter() <--
                                        
24.调用suspend_ops(板级电源管理操作)的recover回调  
25.调用pm_ops(电源管理操作)的resume回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->resume)  
26.调用pm_ops(电源管理操作)的complete回调(依次检查dev, dev->type, dev->class, dev->bus, dev->driver, 调用第一个可用的pm_ops->complete)  
27.恢复用于输出消息的控制台  
28.调用suspend_ops(板级电源管理操作)的recover回调 
  

    /*resume-[freezer]*/
    
					suspend_finish() -->
    [29]       			suspend_thaw_processes() -->
	[30]				pm_notifier_call_chain() -->
	[31]				pm_restore_console() -->
    	       		suspend_finish() <--
    	   		enter_state() <--
    		pm_suspend() <--
    
29.解冻冻结的任务   
30.发送PM_POST_SUSPEND消息， 激活pm注册的内核通知链（内核通知链一般用于内核子系统之间的消息通知）  
31.恢复用于输出消息的控制台
