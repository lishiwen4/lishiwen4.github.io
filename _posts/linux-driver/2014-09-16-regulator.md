---
layout: post
title: "linux regulator"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---


###1. linux regulator  
  
linux regulator框架是为了提供一套标准的接口来控制电压调节器和电流调节器， 这样系统可以动态控制调节器的功率输出达到省电和延长电池寿命的能力， 它能够应用在电压调节器（针对输出电压可控制的调节器）和电流调节器（针对输出电流上限可控制的调节器）。  
  
###2. regulator
  
为其它设备供电的电子设备， 大部分的regulator能够 使能/关闭 输出，有一些regulator还能够控制输出电压和输出电流
  
	input voltage	--->	regulator	--->	output voltage  
      
###3. PMIC
  
电源管理IC， 通常包含数个regulator， 还可能包含其它一些功能模块。  
  
###4. consumer
  
被regulator供电的电子设备， consumer可被分为两类：    
  
1. 固定供电： 这一类consumer不要求改变其供电电压或者供电电流， 只要求能够 使能/关闭 对其供电  。
2. 动态供电： 这一类consumer要求能够改变其供电电压或者供电电流以适应其操作需求。  
  
###5. power Domain  
  
电路中被统同一个regulator， switch， 或power Domain直接供电的部分， 例如， 针对下面的电路：  
  
		
	regulator
	|
	+--------------------------+
	|			|
	switch 1			Consumer D， consumer E
	|
	+--------------------------+
	|			|
	switch 2			consumer B， consumer C
	|
	consumer A
  
图中存在3个power domain：  
  
1. domain 1： switch 1， Consumer D， consumer E， 他们都是由regulator供电  
2. domain 2： switch 2， consumer B， consumer C， 他们都由switch 1 供电  
3. domain 3： consumer A  
  
这三个power domain的供电关系为：  
  
	domain 1 -->  domain 2 -->  domain 3  
      
在power domain中也可能存在一个regulator被另一个regulator供电， 例如：  
  
		regulator 1 
		|
		+-----------------------+
		|			|
		regulator 2		consumer B
		|
		consumer A
                
                
###6. Constraints  
  
Constraints 定义了功率级别用于性能及硬件保护，Constraints共存在三个层次：  
  
1. regulator 层  
由regulator的datasheet限定并在regulator的硬件操作参数中定义， 例如：   
  
+ 电压输出范围在 800mv - 3500mv  
+ 电流输出上限为 20mA @ 5v  
  
2. power domain 层：  
  
在kernel级的board文件初始化代码中软件定义， 用于将power domain限制在一个特定的功率级别当中， 例如：  
+ domain 1 voltage 3300mv
+ domain 2 voltage 1400mv - 1600mv  
+ domain 3 current 0mA - 20mA  
  
3. consumer 层  
  
由consumer的driver动态设定电压和电流的限制  
  
以一个consumer 的driver调节背光亮度为例， consumer请求将电流从5mA调节到10mA， power domain检查新的电流事都在power domain的功率范围内，regulator检查新的电流是否符合自己的范围。如果所有的检查都通过， 那么就可以应用新的regulator值
  
###7. machine driver interface  
  
在介绍machine driver interface时， 使用下图电路为例：  
  
		regulator 1 
		|
		+-----------------------+
		|			|
		regulator 2		consumer B (1.8v - 2.0v)
		|
		consumer A(3.3v)
  
  
在板级驱动中， 使用struct regulator_consumer_supply来描述受电设备：  
  
	struct regulator_consumer_supply {
        const char *dev_name;   /* consumer dev_name() */
        const char *supply;     /* consumer supply - e.g. "vcc" */
	};
    
    #define REGULATOR_SUPPLY(name, dev_name)
    {
    	.dev_name = dev_name,
        .supply	=	name,
    }
  
针对上图的consumer A 和 consumer B 的描述为：  
  
	static struct regulator_consumer_supply regulator1_consumers[] = {
	{
        .dev_name       = "dev_name(consumer B)",
        .supply         = "Vcc",
	},};

	static struct regulator_consumer_supply regulator2_consumers[] = {
	{
        .dev    = "dev_name(consumer A"),
        .supply = "Vcc",
	},};
  
使用 struct regulator_init_data 来描述regulator，如功率限制， regulator 和 consumer的对应关系。  
  
	struct regulator_init_data {
		const char *supply_regulator;        /* or NULL for system supply */

		struct regulation_constraints constraints;

		int num_consumer_supplies;
		struct regulator_consumer_supply *consumer_supplies;

		/* optional regulator machine specific init */
		int (*regulator_init)(void *driver_data);
		void *driver_data;	/* core does not touch this */
	};
    
针对regulator 1， 其regulator_init_data 为：  
  
	static struct regulator_init_data regulator1_data = {
        .constraints = {
                .name = "Regulator-1",
                .min_uV = 3300000,
                .max_uV = 3300000,
                .valid_modes_mask = REGULATOR_MODE_NORMAL,
        },
        .num_consumer_supplies = ARRAY_SIZE(regulator1_consumers),
        .consumer_supplies = regulator1_consumers,
	};
    
针对regulator 2， 其regulator_init_data 为：  
  
	static struct regulator_init_data regulator2_data = {
        .supply_regulator = "Regulator-1",
        .constraints = {
                .min_uV = 1800000,
                .max_uV = 2000000,
                .valid_ops_mask = REGULATOR_CHANGE_VOLTAGE,
                .valid_modes_mask = REGULATOR_MODE_NORMAL,
        },
        .num_consumer_supplies = ARRAY_SIZE(regulator2_consumers),
        .consumer_supplies = regulator2_consumers,
	};
    
regulator 1 为 regulator 2供电， 因此， regulator 2的regulator_init_data.supply_regulator需指明regulator 1.  
  
最后， 将regulator的信息作为platform device注册：  
  
	static struct platform_device regulator_devices[] = {
		{
        	.name = "regulator",
        	.id = DCDC_1,
        	.dev = {
                .platform_data = &regulator1_data,
        	},
		},
        
		{
        	.name = "regulator",
        	.id = DCDC_2,
        	.dev = {
                .platform_data = &regulator2_data,
        	},
		},
	};

	/* register regulator 1 device */
	platform_device_register(&regulator_devices[0]);

	/* register regulator 2 device */
	platform_device_register(&regulator_devices[1]);
    
在platform device的driver中使“regulator_dev *regulator_register()”用注册regulator  
  
	struct regulator_dev *regulator_register(struct regulator_desc *regulator_desc,
                                         const struct regulator_config *config);

	void regulator_unregister(struct regulator_dev *rdev);
    
如果你需要监听regulator event， 可向系统注册notifier call chain：  
  
	int regulator_notifier_call_chain(struct regulator_dev *rdev,
                                  unsigned long event, void *data);

###8. Regulator Consumer Driver Interface  
  
####8.1 获取regulator  
  
在consumer的driver中需要访问regulator时， 获取/释放regulator的方法：  
  
	regulator = regulator_get(dev, "Vcc");
    regulator_put(regulator);  
    
get/put 会维护regulator的引用计数， 当多个regulator具有父子关系时， 使能子regulator时， 引用计数能够保证父regulator也enable。  
  
####8.2 使能/关闭regulator  
  
enable/disable regulator使用  
  
	int regulator_enable(regulator);
	int regulator_disable(regulator);
    
regulator_disable()并不一定正真disable regulator， 它会减少regulator的引用计数， 只有当引用计数减少为0时， 才会正真的disable, 不过， 你可以强制disable regulator：  
  
	int regulator_force_disable(regulator);  
      
你可以查询regulator的当前状态：  
  
	int regulator_is_enabled(regulator);
  
####8.3 设置输出电压  
  
一些consumer driver需要动态调节supply的电压来满足操作需要， 比如cpufreq driver需要根据频率来调节电压达到省电的目的， sdio driver需要选择合适的电压。使用下面的方法来控制regulator的输出电压  
  
	int regulator_set_voltage(regulator, min_uV, max_uV);
  
如果在enable状态下改变电压， 会立即生效， 如果是在disable状态下改变电压， 则会在下次enable时生效。  
需要注意的是， 设置的电压要满足regulator， power domain 和 consumer的要求范围。  
  
获取regulator配置的输出电压  
  
	int regulator_get_voltage(regulator);  
      
返回的是配置的输出电压， 无论regilator是否enable。  
  
####8.4 设置电流限制  
  
一些consumer driver需要动态调节supply的电流来满足操作需求， 比如， LCD背光驱动需要调节电流来改变背光亮度， usb driver需要将供电电流限定到500mA。  
  
	int regulator_set_current_limit(regulator, min_uA, max_uA);  
      
如果在enable状态下改变电流， 会立即生效， 如果是在disable状态下改变电流， 则会在下次enable时生效。  
需要注意的是， 设置的电流要满足regulator， power domain 和 consumer的要求范围。  
  
获取配置的输出电流  
  
	int regulator_get_current_limit(regulator);
    
####8.5 设置运行模式  
  
一些consumer能在自身模式改变时， 切换regulator的运行模式提高省电的效率  
非直接的切换运行模式
  
	int regulator_set_optimum_mode(struct regulator *regulator, int load_uA);
    
load_uA 可以根据consumer的datasheet来计算， 调用该方法后， regulator core会重新计算regulator的负载， 如果有必要， regulator会切换运行模式  
  
一些consumer希望能够直接切换regulator的运行模式  
  
	int regulator_set_mode(struct regulator *regulator, unsigned int mode);  
      
你必须清楚regulator的模式， 并且独占regulator才能直接指定regulator的运行模式。  
  
###9. regulator事件  
  
regulator使用kernel 的notifier framework来通知事件， consumer可以注册监听：  
  
	int regulator_register_notifier(struct regulator *regulator,
                              struct notifier_block *nb);

	int regulator_unregister_notifier(struct regulator *regulator,
                                struct notifier_block *nb);
