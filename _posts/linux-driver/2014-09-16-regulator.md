---
layout: post
title: "linux regulator"
description:
category: linux-driver
tags: [linux, pm, driver]
mathjax: 
chart:
comments: false
---

###1. linux regulator  
  
linux regulator框架是为了提供一套标准的接口来控制电压调节器和电流调节器， 这样系统可以动态控制调节器的功率输出达到省电和延长电池寿命的能力， 它能够应用在电压调节器（针对输出电压可控制的调节器）和电流调节器（针对输出电流上限可控制的调节器）。  
  
###2. 相关的概念

在讲解regulator之前， 先来了解一些相关的概念

####2.1 regulator
  
为其它设备供电的电子设备， 大部分的regulator能够 使能/关闭 输出，有一些regulator还能够控制输出电压和输出电流
  
![regulator 输入输出](/images/linux-driver/regulator-0.png)  
      
####2.2 PMIC
  
电源管理IC， 通常包含数个regulator， 还可能包含其它一些功能模块。  
  
####2.3 consumer
  
被regulator供电的电子设备， consumer可被分为两类：    
  
1. 固定供电： 这一类consumer不要求改变其供电电压或者供电电流， 只要求能够 使能/关闭 对其供电  。
2. 动态供电： 这一类consumer要求能够改变其供电电压或者供电电流以适应其操作需求。  
  
####2.4 power Domain  
  
电路中被统同一个regulator， switch， 或power Domain直接供电的部分， 例如， 针对下面的电路：  
  
![power domain](/images/linux-driver/regulator-1.png) 
  
图中存在3个power domain：  
  
1. domain 1： switch 1， Consumer D， consumer E， 他们都是由regulator供电  
2. domain 2： switch 2， consumer B， consumer C， 他们都由switch 1 供电  
3. domain 3： consumer A  
  
这三个power domain的供电关系为：  
  
![power domain供电关系](/images/linux-driver/regulator-2.png) 
      
在power domain中也可能存在一个regulator被另一个regulator供电， 例如：  
  
![regulator 之间供电](/images/linux-driver/regulator-3.png)                
                
####2.5 Constraints  
  
Constraints 定义了功率级别用于性能及硬件保护，Constraints共存在三个层次：  
  
**regulator 层**
由regulator的datasheet限定并在regulator的硬件操作参数中定义， 例如：   
  
+ 电压输出范围在 800mv - 3500mv  
+ 电流输出上限为 20mA @ 5v  
  
**power domain 层**  
  
在kernel级的board文件初始化代码中软件定义， 用于将power domain限制在一个特定的功率级别当中， 例如：  
+ domain 1 voltage 3300mv
+ domain 2 voltage 1400mv - 1600mv  
+ domain 3 current 0mA - 20mA  
  
**consumer 层**
  
由consumer的driver动态设定电压和电流的限制  
  
以一个consumer 的driver调节背光亮度为例， consumer请求将电流从5mA调节到10mA， power domain检查新的电流事都在power domain的功率范围内，regulator检查新的电流是否符合自己的范围。如果所有的检查都通过， 那么就可以应用新的regulator值
  
###3. regulator的实现

regulator的实现将分为6个部分来分别介绍

1. regulator driver的实现
2. 使用 machine driver interface 描述 regulator 和 consumer
3. 使用 dts 描述 regulator 和 consumer 
4. regulator 提供给consumer driver的接口
5. regulator 在sysfs中导出的属性
6. regulator 在 debugfs 中导出的调试信息

####4. regulator driver的实现

regulator driver的实现涉及到多个数据结构

####4.1 struct regulator_dev

struct regulator_dev 用于表示一个regulator

	struct regulator_dev {
		const struct regulator_desc *desc;				/* regulator的相关描述 */
		int exclusive;						/* 该regulator 是否被某一个 consumer独占 */
		u32 use_count;
		u32 open_count;
		u32 bypass_count;

		struct list_head list; 					/* 用于将regulator挂在全局的 regulator_list 链表 */
		struct list_head consumer_list;				/* 用于将该regulator的consumer形成链表 */

		struct blocking_notifier_head notifier;			/* regulator的内核通知链 */
		struct mutex mutex; 					/* 访问consumer的锁 */
		struct module *owner;
		struct device dev;
		struct regulation_constraints *constraints;
		struct regulator *supply;					/* 为该regulator供电的父regulator的句柄 */
		struct regmap *regmap;

		struct delayed_work disable_work;
		int deferred_disables;

		void *reg_data;						/* 私有数据 data */

		struct dentry *debugfs;

		struct regulator_enable_gpio *ena_pin;			/* 该regulatoer的enable gpio的描述 */
		unsigned int ena_gpio_state:1;
		struct proxy_consumer *proxy_consumer;
	};

**struct regulator_dev由regulator core 根据struct regulator_desc 和 struct regulator_config 来分配并且初始化，自己分配struct regulator_dev是无效的**

所有的 struct regulator_dev 都被保存到一个链表 regulator_list 中

	static LIST_HEAD(regulator_list);

####4.2 struct regulation_constraints

struct regulation_constraints用于描述regulator的限制范围

	struct regulation_constraints {

		/* 限制条件的名称， 用于显示 */
		const char *name;			

		/* 输出电压的范围 (电压调节型的regulator) */
		int min_uV;
		int max_uV;

		/* consumer的电压补偿值(考虑损耗的压降) */
		int uV_offset;

		/* 输出电流的范围 (电流调节型的regulator) */
		int min_uA;
		int max_uA;

		/* 有效的操作模式的掩码 */
		unsigned int valid_modes_mask;

		/* 有效的操作的掩码 */
		unsigned int valid_ops_mask;

		/* regulator input voltage - only if supply is another regulator */
		int input_uV;

		/* suspend to disk 时regulator的状态 */
		struct regulator_state state_disk;

		/* suspend to mem 时regulator的状态 */
		struct regulator_state state_mem;

		/* suspend to standby 时regulator的状态 */
		struct regulator_state state_standby;

		/* suspend时default状态 */
		suspend_state_t initial_state;

		/* 初始化时的操作模式 */
		unsigned int initial_mode;

		unsigned int ramp_delay;

		/* constraint flags */
		/* 该regulator永远不能被disable */
		unsigned always_on:1;	

		/* 该regulator在系统初始化之后就应该被enable， 如果硬件上没有被enable， 那么在该constriant被应用时， 会被linux regulator core enable*/
		unsigned boot_on:1;	

		/* 在初始化时就应用电压限制值 */
		unsigned apply_uV:1;
	};

其中 regulator_constraint.valid_ops_mask 限定了该 regulator 可以执行的操作， 可选的值如下：

	#define REGULATOR_CHANGE_VOLTAGE		0x1	/* 改变电压 */
	#define REGULATOR_CHANGE_CURRENT		0x2	/* 改变电流 */
	#define REGULATOR_CHANGE_MODE		0x4	/* 改变工作模式 */
	#define REGULATOR_CHANGE_STATUS		0x8	/* 可以enable/disable */
	#define REGULATOR_CHANGE_DRMS		0x10	/* 可以动态改变工作模式 */
	#define REGULATOR_CHANGE_BYPASS		0x20	/* 可以置为bypass模式 */
	
若不支持上述的任何操作， 则保持 regulator_constraint.valid_ops_mask 为0

其中 regulator_constraint.valid_modes_mask 限定了该 regulator 支持的mode， 可选的值如下

	#define REGULATOR_MODE_FAST			0x1
	#define REGULATOR_MODE_NORMAL		0x2
	#define REGULATOR_MODE_IDLE			0x4
	#define REGULATOR_MODE_STANDBY		0x8

对于不支持多种工作模式的regulator， 通常只需要声明 regulator_constraint.valid_modes_mask 为 REGULATOR_MODE_NORMAL 即可

对于支持多种工作模式的 regulator 其driver实现的 regulator 操作方式 struct regulator_ops 中， regulator_ops.get_optimum_mode() 方法可以根据当前 regulator 上的电流负载总和(一般工作时电压是不变的)， 计算出最佳的操作模式

**对于支持 REGULATOR_CHANGE_DRMS 操作的regulator， 并且实现了所需的操作方法， 在regulator enable/disable时， rergulator会更根据负载选择最佳的操作模式， 并且切换到该模式， consumer 也可以调用 regulator_set_optimum_mode() 来触发重新选择最佳的工作模式**

####4.3 struct regulator_enable_gpio

regulator通常由一个pin脚连接到gpio口来控制， 该gpio口使用struct regulator_enable_gpio来描述

	struct regulator_enable_gpio {
		struct list_head list;
		int gpio;				/* 全局的gpio编号 */
		u32 enable_count;			/* 该regulator的enable计数 */
		u32 request_count;			/* 该gpio的request计数， 因为可能有多个regulator共用同一个enbale gpio */
		unsigned int ena_gpio_invert:1;	/* 该gpio的状态 */
	};

所有的struct regulator_enable_gpio 都被保存到一个链表 regulator_ena_gpio_list 中

	static LIST_HEAD(regulator_ena_gpio_list);
	

####4.4 struct regulator_desc

struct regulator_desc用于描述一个regulator的电气特性

	struct regulator_desc {
		const char *name;			/* 该regulator的名称 */
		const char *supply_name;		/* 为该regulator的供的父regulator的名称 */
		int id;				/* 该regulator的编号*/
		bool continuous_voltage_range;	/* 该regulator是否可在范围内设置电压为任意值*/
		unsigned n_voltages;		/* 该regulator电压有几个值可选 */
		struct regulator_ops *ops;		/* 该regulator的操作方法 */
		int irq;				/* 该regulator的irq */
		enum regulator_type type;		/* 该regulator的类型 */
		struct module *owner;

		unsigned int min_uV;		/* 该regulator 可选的最低电压 */
		unsigned int uV_step;		/* 该regulator 可选的电压值的步进*/
		unsigned int linear_min_sel;	/* 该regulator 可选的电压值的编号*/
		unsigned int ramp_delay;		/* 该regulator 在电压改变后多久可以平稳 */

		const unsigned int *volt_table;	/* 该regulator 的电压值的映射表， 即一个电压值映射到一个编号 */

		unsigned int vsel_reg;		/* 该regulator 选择电压值的寄存器 */
		unsigned int vsel_mask;		/* 该regulator 选择电压值的掩码 */
		unsigned int apply_reg;		/* 该regulator 应用电压值设定的寄存器 */
		unsigned int apply_bit;		/* 该regulator 应用电压值设定的寄存器的掩码 */
		unsigned int enable_reg;		/* 该regulator 的enable寄存器 */
		unsigned int enable_mask;		/* 该regulator 的enable寄存器掩码 */
		bool enable_is_inverted;		/*  */
		unsigned int bypass_reg;
		unsigned int bypass_mask;

		unsigned int enable_time;		/* 该regulator*/
	};

regulator可以分为2种类型(regulator_desc.type)

	enum regulator_type {
		REGULATOR_VOLTAGE,
		REGULATOR_CURRENT,
	};

####4.5 struct regulator_ops

	struct regulator_ops {

		/* 获取某一个电压值的选择子对应的电压值  单位为mV */
		int (*list_voltage) (struct regulator_dev *, unsigned selector);

		/* 设置regulator电压值的范围， driver应当选择最接近min_uV的值 */
		int (*set_voltage) (struct regulator_dev *, int min_uV, int max_uV, unsigned *selector);

		/* 将一个给定的电压值映射到*/
		int (*map_voltage)(struct regulator_dev *, int min_uV, int max_uV);

		/* 设置电压值， 使用指定的选择子(index) */
		int (*set_voltage_sel) (struct regulator_dev *, unsigned selector);

		/* 获取当前设置的电压值 */
		int (*get_voltage) (struct regulator_dev *);

		/* 获取当前设置的电压值对应的选择子 */
		int (*get_voltage_sel) (struct regulator_dev *);

		/* 设置一个电流regulator的电流限值， driver应当将当前的电流调整到尽量接近 min_uA  */
		int (*set_current_limit) (struct regulator_dev *, int min_uA, int max_uA);

		/* 获取当前设置的电流限值 */
		int (*get_current_limit) (struct regulator_dev *);

		/* enable该regulator */
		int (*enable) (struct regulator_dev *);	

		/* disable 该regulator */
		int (*disable) (struct regulator_dev *);

		/* 检查该regulator是否被enable */
		int (*is_enabled) (struct regulator_dev *);

		/* 设置该regulator当前的操作模式 */
		int (*set_mode) (struct regulator_dev *, unsigned int mode);

		/* 获取该regulator当前的操作模式 */
		unsigned int (*get_mode) (struct regulator_dev *);

		/* Time taken to enable or set voltage on the regulator */
		int (*enable_time) (struct regulator_dev *);
		int (*set_ramp_delay) (struct regulator_dev *, int ramp_delay);
		int (*set_voltage_time_sel) (struct regulator_dev *,
				     unsigned int old_selector,
				     unsigned int new_selector);

		/* 获取该regulator当前的状态 */
		int (*get_status)(struct regulator_dev *);

		/* 获取给定负载条件下， 该regulator的最佳操作模式 */
		unsigned int (*get_optimum_mode) (struct regulator_dev *, int input_uV, int output_uV, int load_uA);

		/* 设置regulator bypass模式(regulator处于enbale状态， 但是不进行控制)是否启用 */
		int (*set_bypass)(struct regulator_dev *dev, bool enable);

		/* 检查regulator bypass模式(regulator处于enbale状态， 但是不进行控制)是否启用 */
		int (*get_bypass)(struct regulator_dev *dev, bool *enable);

		/* 当为该regulator供电的PMIC进入STABDBY/HIBERNATE模式后，设置该regulator的suspend电压值 */
		int (*set_suspend_voltage) (struct regulator_dev *, int uV);

		/* 设置该regulator在其父PMIC进入STABDBY/HIBERNATE模式后仍然处于enable */
		int (*set_suspend_enable) (struct regulator_dev *);

		/* 设置该regulator在其父PMIC进入STABDBY/HIBERNATE模式后处于disable */
		int (*set_suspend_disable) (struct regulator_dev *);

		/* 设置该regulator在其父PMIC进入STABDBY/HIBERNATE模式后的操作模式 */
		int (*set_suspend_mode) (struct regulator_dev *, unsigned int mode);
	};

上述的几种数据结构的关系如下图 ![regulator数据结构的关系](/images/linux-driver/regulator-structuer.png)

####4.6 struct regulator_config

一个regulator (使用struct regulator_dev来描述)， 其相关信息分别由主要的连个数据结构来表示

1. struct regulator_desc   ： 其内容静态确定的
2. struct regulator_config ： 其内容在注册regulator_dev前动态生成

其定义如下

	struct regulator_config {
		struct device *dev;
		const struct regulator_init_data *init_data;
		void *driver_data;
		struct device_node *of_node;
		struct regmap *regmap;

		int ena_gpio;
		unsigned int ena_gpio_invert:1;
		unsigned int ena_gpio_flags;
	};

####4.7 struct regulator

struct regulator 作为struct regulator_dev 的句柄， consumer driver需要操作其regulator时， 都需要先获取其句柄

	struct regulator {
		struct device *dev;
		struct list_head list;
		unsigned int always_on:1;
		unsigned int bypass:1;
		int uA_load;
		int min_uV;
		int max_uV;
		int enabled;
		char *supply_name;
		struct device_attribute dev_attr;
		struct regulator_dev *rdev;
		struct dentry *debugfs;
	};

####4.8 struct regulator_map

struct regulator_map 用于描述consumer与regulator的对应关系

	static LIST_HEAD(regulator_map_list);

	struct regulator_map {
		struct list_head list;		/* 用于挂在 regulator_map_list 中 */
		const char *dev_name;   		/* consumer的名称， 由struct regulator_consumer_supply.dev_name 初始化 */
		const char *supply;		/* supply 的名称 */
		struct regulator_dev *regulator;	/* 对应的regulator */
	};

consumer driver 在需要操作regulator时， 可以通过遍历regulator_map_list, 匹配 regulator_map.dev_name 和 regulator_map.supply, 找到对应的struct regulator_dev， 然后获取其句柄， 进行相关的操作

**对于使用 machine driver interface 的系统， 会根据 struct regulator_consumer_supply 来生成 regulator_map_list， 但是使用dts的系统， 则不会生成 regulator_map_list 的内容**

####4.9 enum regulator_status

linux regulator core 为 regulator 定义了如下几种状态

	enum regulator_status {
		REGULATOR_STATUS_OFF,
		REGULATOR_STATUS_ON,
		REGULATOR_STATUS_ERROR,

		/* 如下4种状态属于 "on" 状态的细分 */
		REGULATOR_STATUS_FAST,
		REGULATOR_STATUS_NORMAL,
		REGULATOR_STATUS_IDLE,
		REGULATOR_STATUS_STANDBY,

		/* 这种状态下 regulator 被 enable 但是不参与调节 */
		REGULATOR_STATUS_BYPASS,

		/* in case that any other status doesn't apply */
		REGULATOR_STATUS_UNDEFINED,
	};

regulator driver在实现 struct regulator_ops时， 在 regulator_ops.get_status()方法中需要返回如上定义的状态值

###5. regulator 的 machine driver interface  
  
一块具体的电路板上面， regulator和consumer的的关系， 作为板级信息， 需要在板级初始化将这些信息告知给regulator core, regulator core提供了一套数据结构用于描述regulator和consumer的关系

+ struct regulator_consumer_supply : 描述受电设备
+ struct regulator_init_data : 描述regulator的限制信息以及regulator和其受电设比的对应关系

通过这些数据结构， 可以构造出 struct regulator_desc 和 struct regulator_config, 然后向regulator core来注册regulator

通常的做法是， 在板级初始化时， 将regulator的consumer的相关信息， 作为一个platform_device的私有数据， 然后注册该platorm_device, 在该platform_device的driver中， 根据regulator的consumer的相关信息向regulator core来注册regulator

在介绍machine driver interface时， 使用下图电路为例：  
  
![regulator示例](/images/linux-driver/regulator-4.png) 
  
在板级驱动中， 使用struct regulator_consumer_supply来描述受电设备：  
  
	struct regulator_consumer_supply {
		const char *dev_name;   /* 为受电设备命名 */
		const char *supply;     /* 为这一供电关系命名 e.g. "vcc" */
	};
    
有一个宏用于初始化struct regulator_consumer_supply

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
		},
	};

	static struct regulator_consumer_supply regulator2_consumers[] = {
		{
			.dev    = "dev_name(consumer A"),
			.supply = "Vcc",
		},
	};
  
使用 struct regulator_init_data 来描述regulator，如功率限制， regulator 和 consumer的对应关系。  
  
	struct regulator_init_data {

		/* 若该regulator由一个父regulator来供电， 则设置为父regulator name 该name需与
		 * /sys/class/regulator/regulator.x/name 相同， 通常为struct regulation_constraints.name
		 * 若该regulator不通过regulator来为其供电， 则设置为NULL */
		const char *supply_regulator;

		/* 该regulator 的约束条件*/
		struct regulation_constraints constraints;
		
		/* 该regulator的直接受电设备的数目*/
		int num_consumer_supplies;

		/* 该regulator的受电设备描述 (数组) */
		struct regulator_consumer_supply *consumer_supplies;

		/* 当该regulator被注册后， 触发该回调， 可根据需要来使用 */
		int (*regulator_init)(void *driver_data);

		/* 该regulator的私有数据， 由driver使用 */
		void *driver_data;
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
			.name = "Regulator-2"
			.min_uV = 1800000,
			.max_uV = 2000000,
			.valid_ops_mask = REGULATOR_CHANGE_VOLTAGE,
			.valid_modes_mask = REGULATOR_MODE_NORMAL,
		},
		.num_consumer_supplies = ARRAY_SIZE(regulator2_consumers),
		.consumer_supplies = regulator2_consumers,
	};
    
regulator 1 为 regulator 2供电， 因此， regulator 2的regulator_init_data.supply_regulator需指明regulator 1， 另外regulator 2可调节电压, 因此和regulator 1 的regulator_init_data 有所差异
  
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
    
在platform device的driver中根据 regulator1_data 和 regulator2_data 构造对应的 struct regulator_desc 和 struct regulator_config， 然后调用  regulator_register()  来注册对应的 regulator  
  
	struct regulator_dev *regulator_register(struct regulator_desc *regulator_desc, const struct regulator_config *config);

###6. 使用dts来描述regulator和consumer

arm平台上的linux开始采用dts来描述板级资源和配置， dts同样可以用来描述regulator和consumer的信息

####6.1 dts描述regulator

例如：

	xyzreg: regulator@0 {
        		regulator-min-microvolt = <1000000>;
        		regulator-max-microvolt = <2500000>;
        		regulator-always-on;
        		......
    	};  

定义了一个regulator， 其编号为0, 电压范围为 1000000uV ～ 2500000uV, 并且不能被disable

如下可选的dts属性用于描述regulator

+ regulator-name : regulator 名称
+ regulator-min-microvolt : 可调节的最小电压
+ regulator-max-microvolt : 可调节的最大电压
+ regulator-microvolt-offset : 电压补偿值
+ regulator-min-microamp : 可调节输出的最小电流
+ regulator-max-microamp : 可调节输出的最大电流
+ regulator-always-on : regulator不能被disable
+ regulator-boot-on : 初始化后enable该regulator
+ &lt;name&gt;-supply : 引用为该regulator供电的设备(可以是另一个regulator)的dts节点， 
+ regulator-ramp-delay : 调节的速度(in uV/uS)

使用dts时， 和使用platform_device的方式相比， 只是上报信息的方式不同而已， 在是需要在合适的地方， 使用Open Firmware提供的of-xxx()接口，获取dts中， regulator的相关属性来初始化 struct regulator_desc 和 struct regulator_config, 然后调用 register_regulator() 来注册regulator

####6.2 dts描述consumer

在regulator的受电设备的dts节点中， 使用“&lt;name&gt;-supply”属性来引用对其供电的regulator, 其中“name”为对这一供电关系的命名

例如， 一个mmc设备， 由两个regulator来控制

1. twl_reg1
2. twl_reg2

这两个regulator的dts节点为

	twl_reg1: regulator@0 {
        		...
        		...
        		...
	};  

	twl_reg2: regulator@1 {
		...
		...
		...
	};

而在该mmc  设备的tds节点中， 可以描述

	mmc: mmc@0x0 {
        		...
        		...
        		vmmc-supply = <&twl_reg1>;
        		vmmcaux-supply = <&twl_reg2>;
    	}; 

即将regulator twl_reg1的供电命名为“vmmc”， 将regulator twl_reg2的供电命名为“vmmcaux”

Open Firware提供了一个接口， 通过supply名称来获取对应的device node

	struct device_node *of_get_regulator(struct device *dev, const char *supply);

regulator core提供的接口“regulator_get()”能够处理dts, 因此在该mmc设备的driver中， 可以通过如下方式获取这两个regulator的句柄

	reg1 = regulator_get(xxx, "vmmc");
	reg2 = regulator_get(xxx, "vmmcaux");

###7. Consumer Driver的regulator 接口  
  
####7.1 获取regulator  
  
在consumer的driver中需要访问regulator时， 获取/释放 regulator 句柄的方法：  
  
	struct regulator *regulator_get(struct device *dev, const char *id);

	/* 资源管理版本的 regulator_get() */
	struct regulator *devm_regulator_get(struct device *dev, const char *id);

	/* 独占版本的 regulator_get(), 即该consumer申请独占该regulator， 即只能获取该regulator的一个句柄 */
	struct regulator *regulator_get_exclusive(struct device *dev, const char *id)

其中的第二个参数为supply name, 根据 supply name 来查找对应的 regulator_dev 由 regulator_dev_lookup() 函数来完成， 其查找顺序为

1. 若系统使用 DTS 并且consumer设备的 DTS 节点中以“&lt;name&gt;-supply”属性指明了所引用的regulator，则可使用 &lt;name&gt; 值作为参数来获取对应的regulator_dev并分配regulator句柄
2. 遍历 regulator_list 链表， 并且匹配regulator的名称(/即对应的 sys/class/regulator/regulator.x/name ， 通常为 struct regulator_constriant.name 或者 struct regulator_desc.name) 来寻找对应的regulator_dev并分配regulator句柄
3. 遍历 regulator_map_list 链表， 若通过 machine interface 来初始化regulator和consumer的关系， 则此时可匹配传递的参数和  struct regulator_map.supply 来寻找对应的regulator_dev并分配regulator句柄

申请独占某一个regulator时， 如果该regulator已经被get过， 试图独占该regulator的操作会失败

####7.2 释放regulator

要释放一个 regulator 句柄， 使用

	void regulator_put(struct regulator *regulator);

	/* 资源管理版本的 regulator_put() */
	void devm_regulator_put(struct regulator *regulator);      

get/put 会维护regulator的引用计数， 当多个regulator具有父子关系时， enable child regulator 时， 引用计数能够保证 parent regulator 也被enable。  
  
####7.2 使能/关闭regulator  
  
支持 REGULATOR_CHANGE_STATUS 操作的regulator 可以被 enable/disable regulator， 接口如下
  
	int regulator_enable(regulator);

	/* disable regulator， 会根据 enable/disable 计数， 当计数为0时， 才真正disable regulator */
	int regulator_disable(regulator);

	/* 延时 disable regulator */
	int regulator_disable_deferred(struct regulator *regulator, int ms)

	/* 忽略引用计数， 强制 disable regualtor */
	int regulator_force_disable(struct regulator *regulator);

	/* 查询regulator的当前是否被enable */
	int regulator_is_enabled(regulator);
  
####7.3 设置/获取regulator的输出电压值  
  
一些consumer driver需要动态调节supply的电压来满足操作需要， 比如cpufreq driver需要根据频率来调节电压达到省电的目的， sdio driver需要选择合适的电压

支持 REGULATOR_CHANGE_VOLTAGE 操作的 regulator， 可使用下面的方法来控制regulator的输出电压  
  
	/* 查询regulator是否支持改变输出电压 */
	int regulator_can_change_voltage(struct regulator *regulator);
	
	/* 检查regulator是否支持该某个电压值范围 */
	int regulator_is_supported_voltage(struct regulator *regulator, int min_uV, int max_uV);

	/* 查询regulator可以设置几种电压值 (即几种选择子可以传递给 regulator_list_voltage() ) */
	int regulator_count_voltages(struct regulator *regulator);

	/* 给出对应选择子对应的电压值 */
	int regulator_list_voltage(struct regulator *regulator, unsigned selector);

	/* 设置regualator的输出电压范围， 若有多个consumer要求设置不同的输出电压， 则满足限定的， 最小的电压值会被应用 */
	int regulator_set_voltage(regulator, min_uV, max_uV);

	/* 设置起始电压值和结束电压值， regulator core会根据所需的时间， 将电压值从 起始值 上升或者下降到 结束值 */
	int regulator_set_voltage_time(struct regulator *regulator, int old_uV, int new_uV);

	/* 恢复regulator的 输出电压为该regulator句柄最后一次配置的输出电压范围，用于其它控制源改变电压shuchu 值后进行恢复 */
	int regulator_sync_voltage(struct regulator *regulator);
	
	/* 获取regulator配置的输出电压，返回的是配置的输出电压， 无论regilator是否enable */
	int regulator_get_voltage(regulator);

**如果在enable状态下改变电压， 会立即生效， 如果是在disable状态下改变电压， 则会在下次enable时生效，需要注意的是， 设置的电压要满足regulator， power domain 和 consumer的限制  **
  
####7.4 设置/获取regulator的输出电流值
  
一些consumer driver需要动态调节supply的电流来满足操作需求， 比如， LCD背光驱动需要调节电流来改变背光亮度， usb driver需要将供电电流限定到500mA。  
  
支持 REGULATOR_CHANGE_CURRENT 操作的regulator可以通过如下接口 设置/获取 输出电流值

	/* 设置输出电流的限制 */
	int regulator_set_current_limit(regulator, min_uA, max_uA);  
      
	/* 获取输出电流的限值 */
	int regulator_get_current_limit(struct regulator *regulator);

**如果在enable状态下改变电流， 会立即生效， 如果是在disable状态下改变电流， 则会在下次enable时生效, 需要注意的是， 设置的电流要满足regulator， power domain 和 consumer的限制**
    
####7.5 设置regulator的工作模式  
  
在consumer在自身工作模式改变时，对应的regulator的负载通常也发生变化，对于支持动态切换工作模式的regulator (即支持REGULATOR_CHANGE_DRMS操作), 切换regulator到最佳的工作模式可以节省电力

	/* 获取regulator 当前的工作模式 */
	unsigned int regulator_get_mode(struct regulator *regulator);

支持 REGULATOR_CHANGE_DRMS 操作的regulator可以非直接地切换运行模式
  
	/* 指定当前consumer所需的工作电流， regulator core会根据该regulator的所有consumer的负载总和选择最佳的工作模式并切换 */
	int regulator_set_optimum_mode(struct regulator *regulator, int load_uA);
    
支持 REGULATOR_CHANGE_MODE 操作的regulator还可以被强制改变工作模式

	/* 直接改变工作模式， 必须是由该consumer独占regulator， 否则会影响其它的设备 */
	int regulator_set_mode(struct regulator *regulator, unsigned int mode);  
        
在直接改变regulator的工作模式时， 你必须清楚regulator支持的模式，并且regulator可选则支持的模式是标准的， 为如下4种：

+ REGULATOR_MODE_FAST
+ REGULATOR_MODE_NORMAL
+ REGULATOR_MODE_IDLE
+ REGULATOR_MODE_STANDBY
  
####7.6 enbale/disable regulator bypass 模式

支持 REGULATOR_CHANGE_BYPASS 模式的regulator， 可以使用如下接口enable/dsiable bypass模式

	int regulator_allow_bypass(struct regulator *regulator, bool enable)

####7.7 regulator 的 bulk 操作

regulator支持批量操作， 使用如下的结构体来描述批量操作的信息

		struct regulator_bulk_data {
			const char *supply;		/* supply 名称， 用于寻找对应的regulator */
			struct regulator *consumer;		/* 保存regulator 句柄 */

			/* private: Internal use */		/* 由regulator core使用， 标识操作结果*/
			int ret;
		};

以 regulator_bulk_get()为例， 可以一次获取多个regulator的名称， 只需要提供一个 regulator_bulk_data 数组， 填充所需要get的regulator的suplly name， 即可获取所有对应的regulator的句柄

有如下的bulk 操作接口

	/* regulator_get() 的批量操作版本, 若有任一操作失败则返回， 已经成功完成的操作会被撤销 */
	int __must_check regulator_bulk_get(struct device *dev, int num_consumers, struct regulator_bulk_data *consumers);

	/* devm_regulator_get 的批量操作版本 若有任一操作失败则返回， 已经成功完成的操作会被撤销 */
	int __must_check devm_regulator_bulk_get(struct device *dev, int num_consumers, struct regulator_bulk_data *consumers);
	
	/* regulator_enable() 的批量操作版本， 若有任一操作失败则返回， 已经成功完成的操作会被撤销 */
	int __must_check regulator_bulk_enable(int num_consumers, struct regulator_bulk_data *consumers);

	/* regulator_set_voltage() 的批量操作版本， 若有任一操作失败则返回， 已经成功完成的操作不会被撤销 */
	int regulator_bulk_set_voltage(int num_consumers, struct regulator_bulk_data *consumers);
	
	/* regulator_disable() 的批量操作版本, 若有任一操作失败则返回， 已经成功完成的操作会被撤销 */
	int regulator_bulk_disable(int num_consumers, struct regulator_bulk_data *consumers);

	/* regulator_force_disable() 的批量操作版本, 若有任一操作失败则返回， 已经成功完成的操作不会被撤销 */
	int regulator_bulk_force_disable(int num_consumers, struct regulator_bulk_data *consumers);

	/* regulator_put() 的批量操作版本, 不会检查是否出错 */
	void regulator_bulk_free(int num_consumers, struct regulator_bulk_data *consumers);

####7.8 监听regulator的改变  
  
如果需要监听regulator的变化， 可以注册regulator noyifier， regulator使用kernel 的notifier framework来通知事件， consumer可以注册监听：  
  
	int regulator_register_notifier(struct regulator *regulator,
                              struct notifier_block *nb);

	int regulator_unregister_notifier(struct regulator *regulator,
                                struct notifier_block *nb);

regulator 支持如下事件

	#define REGULATOR_EVENT_UNDER_VOLTAGE	0x01	/* regulator 输出电压不足 */
	#define REGULATOR_EVENT_OVER_CURRENT		0x02	/* regulator 输出电流太高 */
	#define REGULATOR_EVENT_REGULATION_OUT	0x04	/* regulator 输出超限 */
	#define REGULATOR_EVENT_FAIL		0x08	/* regulator 输出失败 */
	#define REGULATOR_EVENT_OVER_TEMP		0x10	/* regulator 温度过高 */
	#define REGULATOR_EVENT_FORCE_DISABLE	0x20	/* regulator 被软件强制disable */
	#define REGULATOR_EVENT_VOLTAGE_CHANGE	0x40	/* regulator 输出电压改变 */
	#define REGULATOR_EVENT_DISABLE 		0x80	/* regulator 被disable */

###8. regulator 的sysfs接口

在sysfs下的 “/sys/class/regulator/regulator.x” 目录下导出了regulator类的属性， 其中“x”为regulator的编号， 在注册时分配， 依次递增， regulator类的设备在sysfs下导出了如下的属性

	# ls /sys/class/regulator/regulator.0/
	device			/* 对应的device的链接， 通常为platformdevice */
	name			/* regulator的 supply name */
	num_users			/* 该regulator的 enable 计数*/
	power			
	subsystem			
	suspend_disk_state		/* 在suspend to disk 时， 该regulator的 enable/disable 状态 */
	suspend_mem_state		/* 在suspend to mem 时， 该regulator的 enable/disable 状态 */
	suspend_standby_state	/* 在suspend to standby 时， 该regulator的 enable/disable 状态 */
	type			/* 该regulator是电流调节型还是电压调节型的regulator */
	uevent

###9. regulator的debugfs接口

regulator在debugfs 的 “/sys/kernel/debug/regulator/” 中导出了更多的信息, 每一个regulator在其中有一个单独的目录， 例如

	# ll /sys/kernel/debug/regulator/
	drwxr-xr-x root     root              1969-12-31 19:00 8916_l1
	drwxr-xr-x root     root              2015-04-03 06:29 8916_l10
	drwxr-xr-x root     root              1969-12-31 19:00 8916_l11
	drwxr-xr-x root     root              1969-12-31 19:00 8916_l12
	drwxr-xr-x root     root              1969-12-31 19:00 8916_l13
	......
	-r--r--r-- root     root            0 1969-12-31 19:00 supply_map

其中， supply_map 中的内容是 regulator_map_list 链表的反映， 对于使用 machine driver interface 的系统， 会根据 struct regulator_consumer_supply 来生成 regulator_map_list， 但是使用dts的系统， 则不会生成 regulator_map_list 的内容， 因此， 在使用dts的系统上， supply_map 中的内容为空

在每一个regulator单独的目录中， 导出了该regulator的信息， 例如

	# ll /sys/kernel/debug/regulator/8916_l1
	drwxr-xr-x root     root              1969-12-31 19:00 0-0060-vdd		/* 该regulator的consumer的负载信息*/
	drwxr-xr-x root     root              1969-12-31 19:00 1a98000.qcom,mdss_dsi-vdd	/* 该regulator的consumer的负载信息*/
	drwxr-xr-x root     root              1969-12-31 19:00 7824900.sdhci-vdd		/* 该regulator的consumer的负载信息*/
	drwxr-xr-x root     root              1969-12-31 19:00 8916_l8			/* 该regulator的负载信息的负载信息*/
	-r--r--r-- root     root            0 1969-12-31 19:00 bypass_count		/* bypass 计数 */
	-r--r--r-- root     root            0 1969-12-31 19:00 consumers		/* 该regulator的consumer列表 */
	-rw-r--r-- root     root            0 1969-12-31 19:00 enable			/* 该regulator的enable状态 */
	-rw-r--r-- root     root            0 1969-12-31 19:00 force_disable		/* 该regulator的force disable计数*/
	-rw-r--r-- root     root            0 1969-12-31 19:00 mode			/* 该regulator的工作模式 */
	-r--r--r-- root     root            0 1969-12-31 19:00 open_count		/* 该regulator的get 计数*/
	-rw-r--r-- root     root            0 1969-12-31 19:00 optimum_mode		/* 该regulator的最优工作模式*/
	-r--r--r-- root     root            0 1969-12-31 19:00 use_count		/* 该regulator的enable计数 */
	-rw-r--r-- root     root            0 1969-12-31 19:00 voltage			/* 该regulator的输出电压 */

其中， consumer 属性导出了所有的consumers，。 例如

	# cat /sys/kernel/debug/regulator/8916_l8/consumers
	Device-Supply                    EN    Min_uV   Max_uV  load_uA
	7824900.sdhci-vdd                Y    2900000  2900000   400000
	0-0060-vdd                       Y    2850000  2900000        0
	1a98000.qcom,mdss_dsi-vdd        N    2850000  2900000      100
	8916_l8                          N          0        0        0

在每一个consumer的目录中， 导出了该consumer的负载信息， 例如
	
	#cat /sys/kernel/debug/regulator/8916_l8/0-0060-vdd/max_uV
	2900000

	#cat /sys/kernel/debug/regulator/8916_l8/0-0060-vdd/min_uV
	2850000

	#cat /sys/kernel/debug/regulator/8916_l8/0-0060-vdd/uA_load
	0
