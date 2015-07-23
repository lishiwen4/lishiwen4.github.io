---
layout: post
title: "linux-PM runtime PM"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---
  
###1. runtime PM  
  
runtime PM用于动态电源管理， 在设备不需要工作时， 进入低功耗状态， 相较于传统的电源管理方式只能让所有的设备同时suspend，同时resume， runtime PM可以单独控制device， 粒度更为精细  
  
**runtime PM只是提供了机制, 具体何时动态调节设备进入低功耗状态， 则由driver自己决定**  
  
runtime PM的实现代码在“drivers/base/power/runtime.c”中  
  
###2. runtime PM回调  
  
linux RPM 定义了设备的几个状态  
  
	enum rpm_status {             
		RPM_ACTIVE = 0,       
		RPM_RESUMING,         
		RPM_SUSPENDED,        
		RPM_SUSPENDING,       
	};
    
device的RPM状态会被初始化为RPM_SUSPENDED状态， 因此， driver需要根据device的实际状态调用相关api设置为正确的值  
在dev_pm_ops中需要提供3个回调：  
  
	struct dev_pm_ops {
    	......
		int (*runtime_suspend)(struct device *dev);
		int (*runtime_resume)(struct device *dev);
		int (*runtime_idle)(struct device *dev);
        ......
    }
  
1. runtime_suspend() ： 并不是要求设备一定要进入低功耗状态， 而是要求设备在suspend后，不再处理数据，不再和CPUs、RAM进行任何的交互，直到设备的.runtime_resume被调用。因为此时设备的parent（如bus controller）、CPU是、RAM等，都有可能因为suspend而不再工作，如果设备再有任何动作，都会造成不可预期的异常  
2. runtime_resume() ： 当接受到硬件产生的wakeup event或者软件的请求后， 将设备置于active的状态，如果需要的话， 将设备置于full-power的状态并且恢复所有的寄存器  
3. runtime_idle() : runtime_idle不是必须的， 它的存在， 是为了给suspend一个缓冲，有些设备， 例如网卡， 如果连接正常， 那么可能会在很短的时间内， 设备又会active(传输网络数据)， 这个时候， 其实是不适合频繁suspend， 不仅降低了性能， 而且也不会省电， 这个时候， 我们应该等待一段时间后，再发起suspend， 这个时候， PM core不会制动去调用runtime_suspend回调， 应该在runtime_idle中使用helper函数(如pm_schedule_suspend来请求延时suspend)； 若不指定runtime_idle， 则会直接调用runtime_suspend  
  
在device; device_type; class; bus; device_driver结构体中， 都存在dev_pm_ops， 他们的优先级从左到右依次降低， PM 会使用存在并且优先级最高的dev_pm_ops中的runtime回调  
  
###3. runtime PM运行机制  
  
1. 为每个设备维护一个引用计数（device->power.usage_count），用于指示该设备的使用状态
2. 需要使用设备时，device driver调用pm_runtime_get（或pm_runtime_get_sync）接口，增加引用计数；不再使用设备时，device driver调用pm_runtime_put（或pm_runtime_put_sync）接口，减少引用计数
3. 每一次put，RPM core都会判断引用计数的值。如果为零，表示该设备不再使用（idle）了，则使用异步（ASYNC）或同步（SYNC）的方式，调用设备的.runtime_idle回调函数  
4. runtime_idle的存在，是为了在idle和suspend之间加一个缓冲，避免频繁的suspend/resume操作。因此它的职责是：判断设备是否具备suspend的条件，如果具备，在合适的时机，suspend设备
5. pm_runtime_autosuspend、pm_request_autosuspend等接口，会起一个timer，并在timer到期后，使用异步（ASYNC）或同步（SYNC）的方式，调用设备的.runtime_suspend回调函数，suspend设备，同时记录设备的PM状态
6. 每一次get，RPM core都会判断设备的PM状态，如果不是active，则会使用异步（ASYNC）或同步（SYNC）的方式，调用设备的.runtime_resume回调函数，resume设备  
  
RPM的运行机制较为简单， 但是RPM的回调很多采用异步形式调用， 并且， Generic PM的suspend/resume和RPM共存， 因此， RPM core中有很多代码来处理这些请求之间的同步  
  
何时执行get(增加计数)/put(减小计数)才合适  
  
+ 主动访问设备时，如写寄存器、发起数据传输等等，get，增加引用计数，告诉RPM core设备active；访问结束后，put，减小引用计数，告诉RPM core设备可能idle  
+ 设备有事件通知时，get（可能在中断处理中）；driver处理完事件后，put  
  
由于设备之间的级联关系， runtime PM core必须处理:  
  
1. 任何一个子设备处于active状态， 其parent设备必须处于active状态  
2. 任何一个子设备idle后， 必须通知其parent设备  
3. parent设备的所有子设备都idle后， 其才能idle  
  
###4. runtime PM的同步/异步执行  
  
设备驱动代码可在进程和中断两种上下文执行，因此put和get等接口，要么是由用户进程调用，要么是由中断处理函数调用。由于这些接口可能会执行device的.runtime_xxx回调函数，而这些接口的执行时间是不确定的，有些可能还会睡眠等待。这对用户进程或者中断处理函数来说，是不能接受的。

因此，RPM core提供的默认接口（pm_runtime_get/pm_runtime_put等），采用异步调用的方式（由ASYNC flag表示），启动一个work queue，在单独的线程中，调用.runtime_xxx回调函数，这可以保证设备驱动之外的其它模块正常运行。

另外，如果设备驱动清楚地知道自己要做什么，也可以使用同步接口（pm_runtime_get_sync/pm_runtime_put_sync等），它们会直接调用.runtime_xxx回调函数，不过，后果自负  
  
###5. runtime PM API  
  
由于runtime PM的API较多， 仅列出几个常用  
  
基础API(driver开发时不直接使用，但是其它api都是基于这几个api)：

+ extern int \__pm_runtime_idle(struct device *dev, int rpmflags);   
+ extern int \__pm_runtime_suspend(struct device *dev, int rpmflags);  
+ extern int \__pm_runtime_resume(struct device *dev, int rpmflags);   
  
控制设备的runtime PM开关(所有设备会被初始化disable RPM， driver应根据设备状态在合适时机enable RPM)： 

+ extern void pm_runtime_enable(struct device *dev);  
+ extern void pm_runtime_disable(struct device *dev);  
  
提供给subsystem的RPM driver使用(直接调用device driver的RPM回调)：  

+ extern int pm_generic_runtime_idle(struct device *dev);  
+ extern int pm_generic_runtime_suspend(struct device *dev);  
+ extern int pm_generic_runtime_resume(struct device *dev);  
  
driver中用于增加/减少引用计数来触发runtime suspend/resume :  

+ static inline int pm_runtime_get(struct device *dev);  
+ static inline int pm_runtime_put(struct device *dev);  
+ static inline int pm_runtime_get_sync(struct device *dev);  
+ static inline int pm_runtime_put_sync(struct device *dev);   
  
在runtime_idle回调中延时执行runtime_suspend :  

+ extern int pm_schedule_suspend(struct device *dev, unsigned int delay);  
  
