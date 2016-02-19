---
layout: post
title: "linux-PM wakelock"
description:
category: linux-driver
tags: [linux, pm, driver]
mathjax: 
chart:
comments: false
---

###1. wakelock  
  
wakelocks最初出现在Android为linux kernel打的一个补丁集上，该补丁集实现了一个名称为“wakelocks”的系统调用，该系统调用允许调用者阻止系统进入低功耗模式（如idle、suspend等）。同时，该补丁集更改了Linux kernel原生的电源管理执行过程（kernel/power/main.c中的state_show和state_store），转而执行自定义的state_show、state_store  
  
kernel的开发者可是有原则的，死活不让这种机制合并到kernel分支（换谁也不让啊），直到kernel自身的wakeup events framework成熟后，这种僵局才被打破。因为Android开发者想到了一个坏点子：不让合并就不让合并呗，我用你的机制（wakeup source），再实现一个就是了。至此，全新的wakelocks出现了  
  
wakelock实现代码位于 “kernel/power/wakelock.c”

###2. 新旧wakelock的对比  
  
**Android实现的旧式wakelock：**  

1. 一个sysfs文件：/sys/power/wake_lock，用户程序向文件写入一个字符串，即可创建一个wakelock，该字符串就是wakelock的名字。该wakelock可以阻止系统进入低功耗模式  
2. 一个sysfs文件：：/sys/power/wake_unlock，用户程序向文件写入相同的字符串，即可注销一个wakelock  
3. 当系统中所有的wakelock都注销后，系统可以自动进入低功耗状态(PowerManagerService中写触发/sys/power/state)  
4. 向内核其它driver也提供了wakelock的创建和注销接口，允许driver创建wakelock以阻止睡眠、注销wakelock以允许睡眠  

	void wake_lock_init(struct wake_lock *lock, int type, const char *name);
	void wake_lock_destroy(struct wake_lock *lock);
	void wake_lock(struct wake_lock *lock);
	void wake_lock_timeout(struct wake_lock *lock, long timeout);
	void wake_unlock(struct wake_lock *lock);
  
Android的wakelock，真是一个lock，用户程序创建一个wakelock，就是在系统suspend的路径上加了一把锁，注销就是解开这把锁。直到suspend路径上所有的锁都解开时，系统才可以suspend  
  
**linux基于wakeup source的新式wakelock**  
  
1. driver可以创建wakeup source， active wakeup source可以阻止suspend， deactive可以允许suspend  
2. 由autosleep来实现自动休眠  
3. wake_lock 和 wake_unlock， 将wakeup source导出到用户空间  
  
而Kernel的wakelock，是基于wakeup source实现的，因此创建wakelock的本质是在指定的wakeup source上activate一个wakeup event，注销wakelock的本质是deactivate wakeup event。因此，/sys/power/wake_lock和/sys/power/wake_unlock两个sysfs文件的的功能就是：

+ 写wake_lock： 相当于以wakeup source为参数调用\__pm_stay_awake  
+ 写wake_unlock： 相当于以wakeup source为参数， 调用\__pm_relax  
+ 读wake_lock： 获取系统中所有的处于active状态的wake source列表  
+ 读wake_unlock: 返回系统中所有的处于非active状态的wake source信息  
  
###3. wake_lock/wake_unlock  
  
在写wake_lock时， 后面可接timeout值， 例如:  
  
	echo "wakelock_test 1000" > /sys/power/wake_lock  
    
wake_lock属性文件的store()方法由pm_wake_lock()实现:  
  
	int pm_wake_lock(const char *buf)
	{
		......

		if (!capable(CAP_BLOCK_SUSPEND))
			return -EPERM;

		......
        
        mutex_lock(&wakelocks_lock);
		wl = wakelock_lookup_add(buf, len, true);
		if (IS_ERR(wl)) {
			ret = PTR_ERR(wl);
			goto out;
		}
        
		if (timeout_ns) {
			u64 timeout_ms = timeout_ns + NSEC_PER_MSEC - 1;

			do_div(timeout_ms, NSEC_PER_MSEC);
			__pm_wakeup_event(&wl->ws, timeout_ms);
		} else {
			__pm_stay_awake(&wl->ws);
		}

		wakelocks_lru_most_recent(wl);

 	out:
		mutex_unlock(&wakelocks_lock);
		return ret;
	}
    
1. 调用capable()检查当前进程是否有阻止系统suspend的权限  
2. 调用wakelock_lookup_add()查找是否有相同名称的wakelock，存在则直接返回指针， 不存在则分配一个wakelock， wakelock的实现中使用红黑树来保存所有的wakelock  
3. 若指定了timeout值， 则调用\__pm_wakeup_event()来active该wakelock对应的wakeup source, 若未指定timeout值， 则调用\__pm_stay_awake()来wakeup该wakelock对应的wakeup source  
4. wakelocks_lru_most_recent()将刚刚操作的wakelock设置为最近一个访问到的, 和wakelock的垃圾回收有关， 后面会详细说  
  
wake_lock属性文件的store()方法由pm_wake_unlock()实现: 
  
	int pm_wake_unlock(const char *buf)
	{
		......
        
		if (!capable(CAP_BLOCK_SUSPEND))
			return -EPERM;
	
		mutex_lock(&wakelocks_lock);

		wl = wakelock_lookup_add(buf, len, false);
		if (IS_ERR(wl)) {
			ret = PTR_ERR(wl);
			goto out;
		}
		__pm_relax(&wl->ws);

		wakelocks_lru_most_recent(wl);
		wakelocks_gc();

 	out:
		mutex_unlock(&wakelocks_lock);
		return ret;
	}  
    
1. 调用capable()，检查当前进程是否具备阻止系统suspend的权限  
2. wakelock_lookup_add()查找指定名称的wakelock的指针
3. 调用\__pm_relax()来deactive指定wakelock对应的wakeup source  
4. 调用wakelocks_lru_most_recent()将刚刚操作的wakelock设置为最近一个访问到的, 和wakelock的垃圾回收有关， 后面会详细说  
5. 调用wakelocks_gc()， 执行wakelock的垃圾回收动作  
  
一个wakelock的生命周期，应只存在于wakeup event的avtive时期内，因此如果它的wakeup source状态为deactive，应该销毁该wakelock。但销毁后，如果又产生wakeup events，就得重新建立。如果这种建立->销毁->建立的过程太频繁，效率就会降低。

因此，最好不销毁，保留系统所有的wakelocks（同时可以完整的保留wakelock信息），但如果wakelocks太多（特别是不活动的），将会占用很多内存，也不合理。折衷方案，保留一些非active状态的wakelock，到一定的时机时，再销毁，这就是wakelocks的垃圾回收（GC）机制。

wakelocks GC功能可以开关（由CONFIG_PM_WAKELOCKS_GC控制），如果关闭，系统会保留所有的wakelocks，如果打开，它的处理逻辑也很简单：  

1. 使用一个链表, 保存所有的wakelock指针， 按照访问顺序， 最近访问的wakeclock放在头部
2. 当链表中的wakelock数目大于WL_GC_COUNT_MAX时， 从链表的尾部（最不活跃的），依次取出wakelock，判断它的idle时间（通过wakeup source lst_time和当前时间计算）是否超出预设值（由WL_GC_TIME_SEC指定，当前为300s，好长）
3. 如果超出且处于deactive状态，调用wakeup_source_remove，注销wakeup source，同时把它从红黑树、GC list中去掉，并释放memory资源  
  
wake_lock和wake_unlock的show()方法由pm_show_wakelocks()实现， 其中会查询红黑树，列出所有处于activate或者deactivate状态的wakeup source  
