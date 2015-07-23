---
layout: post
title: "linux-PM wakeup source"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux wakeup source    
  
**wakeup source 用于阻止系统suspend**  
  
系统在suspend的状态时， 可以通过wakeup event来唤醒， wakeup event可能是按键被按下， usb插入， 或者是网卡接收到数据包， 产生这些wakeup event的设备则是wakeup source  
  
在suspend的过程中， 若又产生了wakeup event，需要系统进行处理，因为此时并未真正suspend， 会停止suspend流程并且resume  
  
**wakeup event用于阻止系统的suspend流程，而不是用于唤醒系统， suspend后唤醒系统的是中断**
  
###2. wakeup source  
  
在linux kernel中， 只有device才能唤醒系统(并不是所有的设备能够唤醒系统)，能够唤醒系统的设备就是wakeup source  
  
	struct device {
    	......
        struct dev_pm_info power;
        ......
    }
    
	struct dev_pm_info {
    	......
        unsigned int can_wakeup:1;
        struct wakeup_source    *wakeup;
        unsigned int            should_wakeup:1;
    }

	struct wakeup_source {
		const char 		*name;
		struct list_head	entry;
		spinlock_t		lock;
		struct timer_list	timer;
		unsigned long		timer_expires;
		ktime_t total_time;
		ktime_t max_time;
		ktime_t last_time;
		ktime_t start_prevent_time;
		ktime_t prevent_sleep_time;
		unsigned long		event_count;
		unsigned long		active_count;
		unsigned long		relax_count;
		unsigned long		expire_count;
		unsigned long		wakeup_count;
		bool			active:1;
		bool			autosleep_enabled:1;
	};
    
dev_pm_info.can_wakeup 标识了设备是否具有唤醒能力， 对于具有唤醒能力的设备， 会在sysfs的power目录中提供wakeup的信息， 例如  
  
	$ ls /sys/class/net/eth0/device/power/
	async                 runtime_status          wakeup_active_count
	autosuspend_delay_ms  runtime_suspended_time  wakeup_count
	control               runtime_usage           wakeup_expire_count
	runtime_active_kids   wakeup                  wakeup_last_time_ms
	runtime_active_time   wakeup_abort_count      wakeup_max_time_ms
	runtime_enabled       wakeup_active           wakeup_total_time_ms  
      
wakeup_source结构体中:  
  
+ name ： 该wakeup source名称， 默认为device name  
+ timer timer_expires : wakeup event在处理时， 将wakeup source置为activate状态， 在处理完后， 需要置为deactivate状态， 可以设置一个定时器， 到期后由PM core自动将状态设置为deactivate  
+ total time ： wakeup source在activate状态的总时间  
+ max_time ： wakeup source在activate状态的最长时间  
+ last_time : wakeup source最后一次进入activate状态的时间点  
+ start_prevent_time ： wakeup source开始阻止系统自动睡眠（auto sleep）的时间点
+ prevent_sleep_time ： wakeup source阻止系统自动睡眠的总时间
+ event_count ： wakeup source上报的event个数
+ active_count ： wakeup source activate的次数
+ relax_count ： wakeup source deactivate的次数
+ expire_count ： wakeup source timeout到达的次数
+ wakeup_count ： suspend过程中， wakeup source终止suspend过程的次数
+ active ： wakeup source的activate状态
+ autosleep_enabled ： 系统auto sleep的使能状态  
  
在/sys/kernel/debug/wakeup_sources文件中保存了系统中所有wakeup source的状态, 当系统不能suspend时， 可通过该文件的信息来debug  
  
需要注意的是：  
  
+ event_count代表产生wakeup event的次数
+ 在处理wakeup event时， 需将wakeup source切换到activate的状态， 但若是已经处于activate的状态， 则无需切换， 因此， active_count可以小于event_count  
+ wakeup_count wakeup source在suspend过程中，终止suspend过程的次数  
  
###3. wakeup source API  
  
+ void \__pm_stay_awake(struct wakeup_source *ws);
+ void \__pm_relax(struct wakeup_source *ws);
+ void \__pm_wakeup_event(struct wakeup_source *ws, unsigned int msec);
  
这几个API作用如下：

+ \__pm_stay_awake()，通知PM core，ws产生了wakeup event，且正在处理，因此不允许系统suspend（stay awake）  
+ \__pm_relax()，通知PM core，ws没有正在处理的wakeup event，允许系统suspend（relax）  
+ \__pm_wakeup_event()，为上边两个接口的功能组合，通知PM core，ws产生了wakeup event，会在msec毫秒内处理结束（wakeup events framework自动relax）  
  
上述部分的API为PM core使用， device driver不直接使用， 下面的部分API经常在device driver中使用  
  
+ int device_wakeup_enable(struct device *dev);
+ int device_wakeup_disable(struct device *dev);
+ void device_set_wakeup_capable(struct device *dev, bool capable);
+ int device_init_wakeup(struct device *dev, bool val);
+ int device_set_wakeup_enable(struct device *dev, bool enable);
+ void pm_stay_awake(struct device *dev);
+ void pm_relax(struct device *dev);
+ void pm_wakeup_event(struct device *dev, unsigned int msec);
  
这些api的作用如下：

+ device_set_wakeup_capable() : 设置dev的can_wakeup标志（enable或disable), 并增加或移除该设备在sysfs相关的power文件  
+ device_wakeup_enable() / device_wakeup_disable() / device_set_wakeup_enable() : 它们用于对can_wakeup的设备，使能或者禁止wakeup功能  

在enable wakeup时，调用wakeup_source_register()分配一个wakeup source, 然后调用device_wakeup_attach()接口，将新建的wakeup source保存在dev->power.wakeup指针中  
在disable wakeup时， 调用device_wakeup_dettach(), 从dev->power.wakeup中移除， 然后调用wakeup_source_unregister()销毁该wakeup source  
  
+ device_init_wakeup() : 用于设置dev的can_wakeup标志，若是enable，同时调用device_wakeup_enable   
+ pm_stay_awake() / pm_relax() / pm_wakeup_event() : 直接调用上面的wakeup source操作接口，操作device.power->ws变量，处理wakeup events(下一小结会详细说明)  
  
###4. wakeup source阻止suspend  
  
在进行suspend的过程中， 系统随时会接收到wakeup event, 此时， 必须终止suspend过程并处理wakeup event， PM core使用combined_event_count和saved_count来实现这一目的  
  
**combined_event_count**  

drivers\base\power\wakeup.c 中， 定义了combined_event_count原子变量，这一计数内部其实包含了两个计数：  
1. registered wakeup events ： 系统中处理完的的wakeup event次数  
2. wakeup events in progress ： 正在处理的wakeup event个数  
使用了一个小技巧将两个计数存储在一个32bit的combined_event_count里面，当处理完一个wakeup event后， wakeup events in progress计数减1， registered wakeup events计数加1， 这一过程在一个原子操作中完成  
  
**saved_count**  

当系统开始suspend时，会记录registered wakeup events计数到saved_count中, 在suspend的各个阶段，如果检查到该计数变化， 则会终止suspend    
  
当设备有wakeup event在处理时， 需要调用pm_stay_awake来通知PM core，pm_stay_awake最终会调用wakeup_source_activate()来处理，  其中会:  

1. 调用freeze_wake()将系统从suspend to freezer状态下唤醒(suspend过程分为freezer, devices, platform, processors, core 5个阶段)  
2. 设置wakeup source的active标记为true  
3. 更新wakeup source的相关计数  
4. 更新combined_event_count， 增加wakeup events in progress的计数， 当正在处理的event不为0的时候，系统不能suspend
  
当设备处理完wakeup event后， 调用pm_relax来通知PM core， pm_relax则会最终调用到wakeup_source_deactivate()来处理， 其中会:  

1. 设置wakeup source的active标记为false  
2. 更新wakeup source的相关计数   
3. wakeup events in progress减1，registered wakeup events加1  
  
调用pm_stay_awake()后， 当wakeup event处理完之后， 需要手动调用pm_relax(), pm_wakeup_event()是它们的组合版, 在指定的timeout到期后， 自动调用pm_relax()  
  
前面提到过， 当存在还未处理完的wakeup event的时候， suspend流程必须被终止， PM core在suspend的过程中会频繁调用pm_wakeup_pending()来判断是否有正在处理的wakeup event， 其中会:  

1. 比较系统当前的registered wakeup events计数和saved_count， 不相同则判定有pengding wakeup event  
2. 检查wakeup events in progress计数是否为0， 不为0则认为存在pengding wakeup event  
  
**当调用pm_wakeup_pending()检测到正在处理的wakeup event时， PM core会停止suspend过程并resume**
