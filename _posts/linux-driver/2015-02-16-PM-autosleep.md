---
layout: post
title: "linux-PM autosleep"
description:
category: linux-driver
tags: [linux, pm, driver]
mathjax: 
chart:
comments: false
---
###1. autosleep  
  
autosleep 基于wakeup source实现， 用于取代android wakelock中的自动休眠功能  
autosleep实现代码位于“kernel/power/autosleep.c”  
编译内核时， 必须配置CONFIG_PM_AUTOSLEEP选项才能enable autosleep  
  
###2. /sys/power/autosleep  
  
autosleep在sysfs中提供了一个属性文件“/sys/power/autosleep”用以配置autosleep， 类似于“/sys/power/state”文件可以控制suspend到不同的状态， autosleep也可配置不同的状态:  
  
1. freeze : auto freeze
2. standby : auto standby
3. mem : auto mem
4. disk : auto disk
5. off : disable autosleep  
  
向“/sys/power/autosleep”写入以上5个值的任意一个可将autosleep切换到指定的状态，一旦写入非“off”值， autosleep就开始运行  
  
读取“/sys/power/autosleep”可以获得当前的状态， 特别的， 如果系统不支持autosleep， 读取该文件会输出"error"  
  
"/sys/power/autosleep"的show()/store()方法由pm_autosleep_state()/pm_autosleep_set_state()来实现，在下面会详细解释  
  
###3. pm_autosleep_init  
  
autosleep的初始化函数pm_autosleep_init()在PM的初始化时(kernel/power/main.c:pm_init)被调用：  
  
	int __init pm_autosleep_init(void)
	{
		autosleep_ws = wakeup_source_register("autosleep");
		if (!autosleep_ws)
			return -ENOMEM;

		autosleep_wq = alloc_ordered_workqueue("autosleep", 0);
		if (autosleep_wq)
			return 0;

		wakeup_source_unregister(autosleep_ws);
		return -ENOMEM;
	}
    
1. 注册一个名为“autosleep”的wakeup source， 用于在autosleep执行关键代码执行时阻止系统休眠  
2. 分配一个名为“autosleep”的有序workqueue用于调度工作， 触发休眠  
  
autosleep中定义了一个work：  
  
	static DECLARE_WORK(suspend_work, try_to_suspend);

	void queue_up_suspend_work(void)
	{
		if (autosleep_state > PM_SUSPEND_ON)
			queue_work(autosleep_wq, &suspend_work);
	}
    
可以看到， queue_up_suspend_work()中会调度work， 运行try_to_suspend()来触发休眠， 那么什么时候会调用queue_up_suspend_work()呢？ 是在“/sys/power/autosleep”的store()方法pm_autosleep_state()中调用的  
  
###4. pm_autosleep_state()  
  
pm_autosleep_state()的代码比较简单  
  
	int pm_autosleep_set_state(suspend_state_t state)
	{

		#ifndef CONFIG_HIBERNATION
			if (state >= PM_SUSPEND_MAX)
				return -EINVAL;
		#endif

		__pm_stay_awake(autosleep_ws);

		mutex_lock(&autosleep_lock);

		autosleep_state = state;

		__pm_relax(autosleep_ws);

		if (state > PM_SUSPEND_ON) {
			pm_wakep_autosleep_enabled(true);
			queue_up_suspend_work();
		} else {
			pm_wakep_autosleep_enabled(false);
		}

		mutex_unlock(&autosleep_lock);
		return 0;
	}
  
+ 首先使用\_pm_stay_awake()/\__pm_relax()操纵名为“autosleep”的wakelock, 在其中修改autosleep_state为向"/sys/power/autosleep"写入的值， (读取"/sys/power/autosleep"会返回autosleep_state)  
+ 若是写入“/sys/power/autosleep"的值不为“off”,则调用pm_wakep_autosleep_enabled(true)， 然后调用queue_up_suspend_work()触发工作队列“autosleep”开始运行  
+ 若是写入“/sys/power/autosleep"的值为“off”,则调用pm_wakep_autosleep_enabled(true)  
  
pm_wakep_autosleep_enabled()主要用于更新所有wakeup source的autosleep_enabled标志(为何不使用全局的autosleep_enabled标记？)， 对于为active状态的wakeup source，说明它在阻止autosleep， 若此时参数为true， 则更新wakeup source的start_prevent_time(开始组织suspend的时间)， 若参数为false,则更新wakeup source的prevent_sleep_time(总共阻止睡眠的时间)  
  
###5. try_to_suspend()  
  
	static void try_to_suspend(struct work_struct *work)
	{
		unsigned int initial_count, final_count;

		if (!pm_get_wakeup_count(&initial_count, true))
			goto out;

		mutex_lock(&autosleep_lock);

		if (!pm_save_wakeup_count(initial_count) ||
			system_state != SYSTEM_RUNNING) {
			mutex_unlock(&autosleep_lock);
			goto out;
		}

		if (autosleep_state == PM_SUSPEND_ON) {
			mutex_unlock(&autosleep_lock);
			return;
		}
		if (autosleep_state >= PM_SUSPEND_MAX)
			hibernate();
		else
			pm_suspend(autosleep_state);

		mutex_unlock(&autosleep_lock);

		if (!pm_get_wakeup_count(&final_count, false))
			goto out;
	
		if (final_count == initial_count)
			schedule_timeout_uninterruptible(HZ / 2);

 	out:
		queue_up_suspend_work();
	}  
      
+ 该接口是wakeup count的一个例子， 如wakeup count中所描述， 调用pm_get_wakeup_count（block为true），阻塞等待， 直到没有wakeup event在处理才返回count， 如果读取失败， 则调度workqueue， 重新进入try_to_suspend()    
+ 将获取wakeup count，保存在initial_count中  
+ 将initial_count写入saved_count， 如果失败， 说明产生了wakeup event， 则则调度workqueue， 重新进入try_to_suspend()  
+ 如果成功，且当前系统状态为running，根据autosleep状态，调用hibernate或者pm_suspend，suspend系统  
+ 运行到mutex_unlock(&autosleep_lock)这一行时，说明是suspend过程中遇到了wakeup event， 或者是suspend之后被唤醒
+ 非阻塞情况下在读取wakeup count， 保存到final count中， 如果此时有wakeup event在处理， 则调度workqueue， 重新进入try_to_suspend()  
+ 如果initial_count和final count相同， 说明suspend之间和resume之后的wakeup count相同， 比较奇怪， 需要等待0.5s后再尝试则调度workqueue， 重新进入try_to_suspend()  
  
在queue_up_suspend_work()中运行try_to_suspend(), try_to_suspend()中又调用queue_up_suspend_work()， 这样就形成了一个环， 尝试suspend， resume后又继续尝试suspend， 只要通过“/sys/power/autosleep"enable autosleep之后， 这个环就开始运行， 那么何时停止autosleep呢， 那就是向“/sys/power/autosleep"写如”off"后， 在try_to_suspend()中判断到autosleep_state == PM_SUSPEND_ON时， 直接返回， 不再调用queue_up_suspend_work()， 这个环就断开了， autosleep也就停止了  
  
autosleep的源码很少， 只有100行左右， 可自行阅读  
  
