---
layout: post
title: "linux-PM PM callback"
description:
category: linux-driver
tags: [linux, pm, driver]
mathjax: 
chart:
comments: false
---


###1.  linux的电源管理架构
   
linux的电源管理的粗略的架构如下：  
  
![simple_pm_arc](/images/simple_pm_arc.png)  
  
在用户接口之下， 是由kernel中的 power core(位于kernel/power/)来处理与硬件无关的核心逻辑， 往下则是相关的driver，一部分是硬件体系结构相关的driver， 另一部分则是各设备驱动实现的电源管理功能   
  
###2.  原始的linux电源管理  
  
在移动设备爆发之前， linux大部分被安装到pc机和服务器上面， 性能是首要的追求目标， 而电源管理则不是重点， 早期的linux电源管理极为粗放， 仅仅只是提供了 关机， 重启， 休眠(hibernate， 保存到硬盘)， 睡眠(sleep， 保存到内存)， 关闭显示器这些功能

这些功能的用户接口如下：  
  
+ 关机/重启 : 使用系统调用reboot()来实现， 另外sysRq(组合键或者写/proc/sys-trigger)也可用于触发reboot    
+ hibernate : echo  disk > /sys/power/state
+ sleep : echo mem > /sys/power/state
  
###3. reboot的流程  
  
reboot系统调用根据第3个参数的不同， 可执行halt， poweroff， restart等不同的动作：  
  
+ LINUX_REBOOT_CMD_CAD_OFF ： 禁用 Ctrl + Alt + Delet组合键重启
+ LINUX_REBOOT_CMD_CAD_ON ： 启用 Ctrl + Alt + Delet组合键重启
+ LINUX_REBOOT_CMD_HALT 
+ LINUX_REBOOT_CMD_KEXEC : 直接运行另一个kernel，可用于快速重启
+ LINUX_REBOOT_CMD_POWER_OFF
+ LINUX_REBOOT_CMD_RESTART
+ LINUX_REBOOT_CMD_RESTART2
  
`echo c > /proc/sysrq-trigger`和`echo b > /proc/sysrq-trigger` 分别用于crash和reboot系统  
  
![reboot_step](/images/reboot_step.png)
  
+ kernel_shutdown_prepare 中发出reboot通知， 调用所有device的shutdown回调
+ migrate_to_reboot_cpu 用于将task迁移到boot cpu上并且禁止在其它的cpu上调度任务  
+ syscore_shutdown 调用所有的syscore_ops的shutdown回调  
+ machine_power_xxx 和体系机构相关， 调用个体系结构定应的machine_ops结构体中的对应的回调  
+ sysrq直接调用machine_ops的回调来处理， 这样会丢失系统运行期间的数据  
  
machine_ops的具体实现和系统架构相关  
  
###4. 设备和驱动的电源管理回调
  
设备的电源管理是linux的电源管理的核心内容， 设备的电源管理回调实现定义好的接口， PM core在合适的时机调用相关的接口将设备置为合适的状态以节省电源  
  
旧版的内核中， PM callback散布在设备模型的各个地方  
  
	struct bus_type {
    	......
        void (*shutdown)(struct device *dev);
		int (*suspend)(struct device *dev, pm_message_t state);
        int (*resume)(struct device *dev);
        ......
    }
    
    struct device_driver {
    	......
    	void (*shutdown) (struct device *dev);
		int (*suspend) (struct device *dev, pm_message_t state);
		int (*resume) (struct device *dev);
        ......
    }
    
    struct class {
    	......
        int (*suspend)(struct device *dev, pm_message_t state);
		int (*resume)(struct device *dev);
        ......
    }
    
这么做的后果就是一旦需要添加新的电源管理接口， 就需要修改这些数据结构， 破坏了数据的封装特性, 新的内核中， 将这些回调统一成一个新的数据结构 struct dev_pm_ops, 上述的数据结构只需要包含 dev_pm_ops 即可， 当然， 为了保持兼容性， 旧的callback还是被保留了， 但是不建议使用了  
  
dev_pm_ops在 bus_type, device_driver, device_type结构中  
  
新的 dev_pm_ops 结构如下 :  
  
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
    
+ prepare
+ complete
+ suspend
+ resume
+ freeze
+ thaw
+ poweroff
+ restore
+ suspend_late
+ resume_early
+ freeze_late
+ thaw_early
+ poweroff_late
+ restore_early
+ suspend_noirq
+ resume_noirq
+ freeze_noirq
+ thaw_noirq
+ poweroff_noirq
+ restore_noirq
+ runtime_suspend
+ runtime_resume
+ runtime_idle
  
以suspend为例， 其调用流程为： prepare() -> suspend() -> suspend_late() -> suspend_noirq()
而resume的流程则为 : resume_noirq() -> resume_early() -> resume() -> complete()  
  
runtime_xxx 属于后来引入的runtime_pm， 后续会介绍  
驱动工程师应该根据熟悉每一个callback的使用场景， 根据需要来实现  
  
###5. dev_pm_info和dev_pm_domain  
  
dev_pm_info的成员达到40多个， 保存了设备的PM状态， 例如能否唤醒， 是否已经suspend， 此处不一一列举  
  
pm_doamin只包含了一个dev_pm_ops的结构， 可能后续会继续添加其它成员， dev_pm_ops 存在于bus_type
, device_driver, class, device_type中， 他们都属于device driver， 对于没有driver的device， 若要处理它的电源管理， 就需要dev_pm_domain  
  
###6. PM callback的辅助函数  
  
PM core在调用device的PM callback时， 需要判断其是否存在，若存在则执行， 为此， 提供了一堆辅助API：  
  
	int pm_generic_prepare(struct device *dev);
	int pm_generic_suspend_late(struct device *dev);
	int pm_generic_suspend_noirq(struct device *dev);
	int pm_generic_suspend(struct device *dev);
	int pm_generic_resume_early(struct device *dev);
	int pm_generic_resume_noirq(struct device *dev);
	int pm_generic_resume(struct device *dev);
	int pm_generic_freeze_noirq(struct device *dev);
	int pm_generic_freeze_late(struct device *dev);
	int pm_generic_freeze(struct device *dev);
	int pm_generic_thaw_noirq(struct device *dev);
	int pm_generic_thaw_early(struct device *dev);
	int pm_generic_thaw(struct device *dev);
	int pm_generic_restore_noirq(struct device *dev);
	int pm_generic_restore_early(struct device *dev);
	int pm_generic_restore(struct device *dev);
	int pm_generic_poweroff_noirq(struct device *dev);
	int pm_generic_poweroff_late(struct device *dev);
	int pm_generic_poweroff(struct device *dev);
	void pm_generic_complete(struct device *dev);
  
以pm_generic_prepare为例，就是查看dev->driver->pm->prepare接口是否存在，如果存在，直接调用并返回结果  
  
PM core需要遍历所有的设备， 来执行其PM callback， 可以使用如下API  
  
	void device_pm_lock(void);
	void dpm_resume_start(pm_message_t state);
	void dpm_resume_end(pm_message_t state);
	void dpm_resume(pm_message_t state);
	void dpm_complete(pm_message_t state);
	void device_pm_unlock(void);
	int dpm_suspend_end(pm_message_t state);
	int dpm_suspend_start(pm_message_t state);
	int dpm_suspend(pm_message_t state);
	int dpm_prepare(pm_message_t state);
	int device_pm_wait_for_dev(struct device *sub, struct device *dev);
	void dpm_for_each_dev(void *data, void (*fn)(struct device *, void *));
  
设备驱动模型在add device时， 会调用device_pm_add将其添加到全局链表dpm_list中， 调用上述API会遍历该链表， 对每一个device调用其callback   
  
###7. PM callback的调用顺序  
  
以device_prepare为例， 可以看到， 依次寻找：  
  
1. dev->pm_domain
2. dev->type->pm
3. dev->class->pm
4. ev->bus->pm
5. dev->driver->pm
  
越往后优先级越高， 优先及最高的将被调用， 其它的不会被调用
其它回调也是如此
