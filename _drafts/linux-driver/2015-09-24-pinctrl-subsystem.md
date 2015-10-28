---
layout: post
title: "linux pinctrl 子系统"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux pinctrl 子系统

在一个嵌入式的系统里面 通常存在着多种大量的io口， 有soc自身的io口， 也有一些外部chip扩展的io口(例如pmic上通常会有扩展gpio)， 并且这些io口存在不小的差异：

+ soc的io口一般直接通过寄存器来配置， 而扩展io口的chip大多通过i2c， spi之类的总线连接到soc上， 配置过程中可能会sleep， 因此不能在访问此类io口时不能使用spinlock来加锁
+ 有些io口支持中断触发， 这一类io口中有一些还支持唤醒系统
+ 有些io口有多中复用功能， 例如既可以作为gpio， 也可以作为 i2c总线中的引脚
+ 有些io口还可以配置上/下拉

因此， 在gpio的driver中， 通常需要完成这些功能：

+ 配置io口作为gpio还是特殊功能引脚
+ 作为gpio时， 还需要能够配置gpio的方向， 上/下拉， drive-strength， 作为输出口时，还要能够设置输出电平， 作为输入口时， 还要能够读取输入电平
+ 作为中断脚时，还要能够设置中断触发方式， enable/disable 中断， 清除中断的状态

在linux kernel(3.14之前)， 并没有提供统一的接口来配置pin脚， 有些会在bootloader或者firmware中完成pin脚的配置， 有些会在device driver中， 利用pin controller driver提供的接口来完成pin脚的配置，不同的厂商有不同的实现，并且， gpio子系统的实现也依赖这些pin脚配置接口， 在新的linux kernel(3.14之后)，引入了pinctrl subsystem来完成pin脚相关的hw配置， 避免各厂商实现重复的代码， gpio子系统也使用pinctrl 子系统提供的统一接口来完成hw相关的操作

###2. pin & pin controller

pin指的是pad，金手指， ball等希望控制的输入/输出线， 使用无符号整数来编号， 从0开始(在一个pin controller自己的命名空间内)命名，命名可以是稀疏的， 即中间可以存在空洞

pin controller是一块硬件， 通常是一组寄存器， 用于控制一个或者一组pin脚的 复用， 偏置， 负载电流， 驱动能力等


###3. pin controller driver

####3.1 struct pinctrl_dev

struct pinctrl_dev 用于表示一个pin controller类型的device

	struct pinctrl_dev {
		struct list_head node;			//用于形成链表
		struct pinctrl_desc *desc;			//描述 pin controller， 下面小节有详细解释
		struct radix_tree_root pin_desc_tree;
		struct list_head gpio_ranges;		//指出哪一范围的gpio 范围映射的pin脚由该pin controller来处理
		struct device *dev;			//device data
		struct module *owner;			//THIS_MODULE
		void *driver_data;				//driver data
		struct pinctrl *p;				//该设备的pin control state 链表， pin control state在后面的章节有解释
		struct pinctrl_state *hog_default;		//该设备的名为 “default” 的 pin control state
		struct pinctrl_state *hog_sleep;		//该设备的名为 “slepp” 的 pin control state
		struct mutex mutex;
	#ifdef CONFIG_DEBUG_FS
		struct dentry *device_root;			//debugfs中的根目录
	#endif
	};


####3.2 描述pin & pin controller

当一个pin controller实例化之后， 需要向pinctrl subsytem 注册一个描述符 struct pinctrl_desc， 这一描述符通常还包含一组描述符struct pinctrl_pin_desc， 用于描述该pin controller所控制的pin脚

	// 用于描述一个pin脚
	struct pinctrl_pin_desc {
		unsigned number;		//在pin controller命名空间内的该pin脚的编号，从0开始 
		const char *name;		//pin脚name
	};

	// 用于描述一个pin controller
	struct pinctrl_desc {
		const char *name;				//pin controller name
		struct pinctrl_pin_desc const *pins;		//pin controller控制的所有pin脚的描述
		unsigned int npins;			//pin controller控制的pin脚的数量
		const struct pinctrl_ops *pctlops;		//pin controller的操作函数指针表
		const struct pinmux_ops *pmxops;		//pin controller的pin脚复用操作的函数指针表
		const struct pinconf_ops *confops;		//pin controller的pin脚配置操作的函数指针表
		struct module *owner;			//module owner， 通常设置为THIS_MODULE
	};

使用如下的api可以向pinctrl subsystem 注册一个 struct pinctrl_desc

	struct pinctrl_dev *pinctrl_register(struct pinctrl_desc *pctldesc, struct device *dev, void *driver_data)
其中

+ dev ： pin controller 的父设备， 若连接到某总线上， 则父设备为该总线
+ driver_data ： driver的private data

假设一个PGA chip pin脚如下：

	       A   B   C   D   E   F   G   H
	   
	  
	8      o   o   o   o   o   o   o   o

	7      o   o   o   o   o   o   o   o

	6      o   o   o   o   o   o   o   o

	5      o   o   o   o   o   o   o   o

	4      o   o   o   o   o   o   o   o

	3      o   o   o   o   o   o   o   o

	2      o   o   o   o   o   o   o   o

	1      o   o   o   o   o   o   o   o

则在pin controller driver可按照如下的方式来定义pin和pin controller的描述

	#include <linux/pinctrl/pinctrl.h>

	const struct pinctrl_pin_desc foo_pins[] = {
      		PINCTRL_PIN(0, "A8"),
      		PINCTRL_PIN(1, "B8"),
      		PINCTRL_PIN(2, "C8"),
      		...  
      		PINCTRL_PIN(61, "F1"),
      		PINCTRL_PIN(62, "G1"),
      		PINCTRL_PIN(63, "H1"),
	};

	static struct pinctrl_desc foo_desc = {
    		.name = "foo",
    		.pins = foo_pins,
    		.npins = ARRAY_SIZE(foo_pins),
    		.owner = THIS_MODULE,
	};

	int __init foo_probe(void)
	{
    		struct pinctrl_dev *pctl;

    		pctl = pinctrl_register(&foo_desc, <PARENT>, NULL);
    		if (IS_ERR(pctl))
        			pr_err("could not register foo pin driver\n");
	}

**示例只关注pin和pin controller的描述，因此未列出pin controller的相关的ops成员**

####3.3 pin group

对于支持pin脚功能复用的pin controller， 某些管脚可能参与多种功能， 例如 { 0, 8, 16, 24 }可以作为spi端口的 CLK， RXD， TXD， FRM， 而{24, 25}可以作为i2c总线的CLK， SDA， 因此无法同时使用这两个spi和i2c接口

pinctrl 子系统提供机制将这些共同参与某一功能的pin脚归类为1个group， 例如， 上述的 { 0, 8, 16, 24 } 为1个group， {24, 25}为一个group， 并且呢个够获取某个group包含的所有pin脚

**pinctrl 子系统并未定义数据结构来存储group相关的信息， 通常可以定义类似的如下的数据结构来表示**

	struct foo_group {
    		const char *name;		//group的名称
    		const unsigned int *pins;	//group所包含的pin脚的id
    		const unsigned num_pins;	//group所包含的pin脚数目
	};

在 pin controller描述符的 pctlops成员(即struct pinctrl_ops变量) 中定义了3个和pin group相关的回调函数， 需要pin controller driver 来实现它

	struct pinctrl_ops {

		int (*get_groups_count) (struct pinctrl_dev *pctldev);

		const char *(*get_group_name) (struct pinctrl_dev *pctldev, unsigned selector);

		int (*get_group_pins) (struct pinctrl_dev *pctldev, unsigned selector,
			       const unsigned **pins, unsigned *num_pins);

		......
	}

+ get_groups_count() ： 获取pin controller中划分的group的总数
   + 参数 pctldev ： pin controller描符
+ get_group_name() : 获取某个pin group的名称
   + 参数 pctldev ： pin controller描符
   + 参数 selector ： group 的index
+ get_group_pins() : 获取某个pin group所包含的所有pin脚的编号
   + 参数 pctldev ： pin controller描符
   + 参数 selector ： group 的index
   + 参数 pins ： 用于返回group所包含所有的pin脚id的数组
   + 参数 num_pins ： 用于返回group所包含的pin脚的数目

例如， 对于上述的 { 0, 8, 16, 24 } 和 {24, 25} 可以在pin contrller driver中如下描述

	#include <linux/pinctrl/pinctrl.h>

	struct foo_group {
    		const char *name;
    		const unsigned int *pins;
    		const unsigned num_pins;
	};

	static const unsigned int spi0_pins[] = { 0, 8, 16, 24 };
	static const unsigned int i2c0_pins[] = { 24, 25 };

	static const struct foo_group foo_groups[] = {
    		{
        			.name = "spi0_grp",
        			.pins = spi0_pins,
        			.num_pins = ARRAY_SIZE(spi0_pins),
    		},
    		{
        			.name = "i2c0_grp",
        			.pins = i2c0_pins,
        			.num_pins = ARRAY_SIZE(i2c0_pins),
    		},
	};

	static int foo_get_groups_count(struct pinctrl_dev *pctldev)
	{
    		return ARRAY_SIZE(foo_groups);
	}

	static const char *foo_get_group_name(struct pinctrl_dev *pctldev,
                       unsigned selector)
	{
    		return foo_groups[selector].name;
	}

	static int foo_get_group_pins(struct pinctrl_dev *pctldev, unsigned selector,
                   unsigned ** const pins,
                   unsigned * const num_pins)
	{
    		*pins = (unsigned *) foo_groups[selector].pins;
    		*num_pins = foo_groups[selector].num_pins;
    		return 0;
	}

	static struct pinctrl_ops foo_pctrl_ops = {
    		.get_groups_count = foo_get_groups_count,
    		.get_group_name = foo_get_group_name,
    		.get_group_pins = foo_get_group_pins,
	};


	static struct pinctrl_desc foo_desc = {
       		...
       		.pctlops = &foo_pctrl_ops,
	};

####3.4 从dts中获取pin controller信息

pin controller相关的dts文件中， 通常采用如下结构：

	pinctrl@xxxxx {

		pin controller 的相关描述

		pin configurations 的相关描述
	};

例如在qualcomm平台上， pin controller的描述， 通常放置在 “arch/arm64/boot/dts/qcom/*-pinctrl.dts*” 文件中， 例如

	&soc {
    		tlmm_pinmux: pinctrl@1000000 {
        			compatible = "qcom,msm-tlmm-8916";			//pin controller相关的描述
        			reg = <0x1000000 0x300000>;
        			interrupts = <0 208 0>;

        			/*General purpose pins*/				//其中gpio相关的描述， 一个pin controller中的pin， 可以被sw划分到多个gpio chip上
        			gp: gp { 
            			qcom,num-pins = <145>;
            			#qcom,pin-cells = <1>; 
            			msm_gpio: msm_gpio {
                				compatible = "qcom,msm-tlmm-gp";
                				gpio-controller;
                				#gpio-cells = <2>; 
                				interrupt-controller;
                				#interrupt-cells = <2>; 
                				num_irqs = <145>;
            			};
        			};
		};
		......
	};
 
“pinctrl@1000000” 这一node描述了pin controller， 其相关的信息有：

+ compatible ： driver可以通过该字符串来匹配device
+ reg ： 可寻址的设备的地址范围， 从0x1000000 ～ 0x300000
+ interrupts 

####3.5 pin脚复用

通常， 在SoC上，pin脚的数目有限， 很多pin脚都提供了复用功能，例如， 某个pin脚可既可以作为i2c的SDA线， 也可以作为gpio， 甚至有些pin脚可以配置为5，6种不同的功能

**需要注意的是， 在同一个pin controller中， 可能有多个pin ctroller属于同一功能， 例如2个pin group {24, 25}, {30, 31}分别是两条i2c总线的， 则它们是相同的功能**

pinctrl subsystem中，功能使用从0开始的连续数字枚举， pinctrl subsystem并未定义数据结构用于保存功能相关的信息， pin controller driver可以定义类似如下的数据结构来表示

pin controller的描述符中的成员 pmxops (即struct pinmux_ops变量) 中存储了和pin脚复用相关的回调函数， 需要由pin controller driver来实现它(若该pin controller中的pin脚支持功能复用的话)

	struct pinmux_ops {
		int (*request) (struct pinctrl_dev *pctldev, unsigned offset);
		int (*free) (struct pinctrl_dev *pctldev, unsigned offset);
		int (*get_functions_count) (struct pinctrl_dev *pctldev);
		const char *(*get_function_name) (struct pinctrl_dev *pctldev, unsigned selector);

		int (*get_function_groups) (struct pinctrl_dev *pctldev,
				  unsigned selector,
				  const char * const **groups,
				  unsigned * const num_groups);

		int (*enable) (struct pinctrl_dev *pctldev, unsigned func_selector, unsigned group_selector);
		void (*disable) (struct pinctrl_dev *pctldev, unsigned func_selector, unsigned group_selector);

		int (*gpio_request_enable) (struct pinctrl_dev *pctldev,
				    struct pinctrl_gpio_range *range,
				    unsigned offset);

		void (*gpio_disable_free) (struct pinctrl_dev *pctldev,
				   struct pinctrl_gpio_range *range,
				   unsigned offset);

		int (*gpio_set_direction) (struct pinctrl_dev *pctldev,
				   struct pinctrl_gpio_range *range,
				   unsigned offset,
				   bool input);
	};

其中各回调函数及其参数的意义：

1. request() : 由pinctrl subsystem core来调用， 以确定某一个pin脚是否支持功能复用， 返回负值代表“no”，该函数还表示有user请求了该pin脚
   + 参数 pctldev ： pin controller描符
   + 参数 offset ： pin脚的编号
2. free() : 同request()相反， 当request 某一pin脚成功之后， 再次request该pin脚就会失败， 需要等到free()之后才能request， 这两个操作用于锁定pin脚资源
   + 参数 pctldev ： pin controller描符
   + 参数 offset ： pin脚的编号
3. get_functions_count() : 获取该pin controller支持的可选功能(命名了的)的总数
   + 参数 pctldev ： pin controller描符
4. get_function_name() ： 根据功能号获取某一功能的名称
   + 参数 pctldev ： pin controller描符
   + 参数 selector ： 功能号
5.get_function_groups() : 根据功能号， 查询属于该功能的pin group， 例如， 两条i2c总线， 则有2个pin group属于 i2c功能
   + 参数 pctldev ： pin controller描符
   + 参数 selector ： 功能号
   + 参数 groups ： 用于返回pin group name 字符串指针的数组
   + 参数 num_groups : 用于返回该功能包含多少pin group
6. enable() : 启用某一pin group上的某一功能
   + 参数 pctldev ： pin controller描符
   + 参数 func_selector ： 功能号
   + 参数 group_selector ： pin group编号(注意是在同一function中的编号， 而不是在整个pin controller中的编号)
7. disable() : 同enable()相反， 参数和enable()相同
8. gpio_request_enable() : 请求并且enable pin脚的gpio功能
   + 参数 pctldev ： pin controller描符
   + 参数 range ： gpio 和 pin脚的映射关系， 将在下一小节解释
   + 参数 offset ： gpio在range中的offset
9. gpio_disable_free() : 同gpio_request_enable()相反， disable gpio功能并且取消请求
   + 参数 pctldev ： pin controller描符
   + 参数 range ： gpio 和 pin脚的映射关系， 将在下一小节解释
   + 参数 offset ： gpio在range中的offset
10. gpio_set_direction() : 设置gpio的输入输出方向
   + 参数 pctldev ： pin controller描符
   + 参数 range ： gpio 和 pin脚的映射关系， 将在下一小节解释
   + 参数 offset ： gpio在range中的offset
   + 参数 input ： 输入还是输出

例如， 一个pin ctroller中的几组pin 可以复用为两条spi总线和mmc总线

	#include <linux/pinctrl/pinctrl.h>
	#include <linux/pinctrl/pinmux.h>

	struct foo_group {
    		const char *name;
    		const unsigned int *pins;
    		const unsigned num_pins;
	};

	static const unsigned spi0_0_pins[] = { 0, 8, 16, 24 };
	static const unsigned spi0_1_pins[] = { 38, 46, 54, 62 };
	static const unsigned mmc0_1_pins[] = { 56, 57 };
	static const unsigned mmc0_2_pins[] = { 58, 59 };

	static const struct foo_group foo_groups[] = {
    		{
        			.name = "spi0_0_grp",
        			.pins = spi0_0_pins,
        			.num_pins = ARRAY_SIZE(spi0_0_pins),
    		},
    		{
        			.name = "spi0_1_grp",
        			.pins = spi0_1_pins,
        			.num_pins = ARRAY_SIZE(spi0_1_pins),
    		},
    		{
        			.name = "mmc0_1_grp",
        			.pins = mmc0_1_pins,
        			.num_pins = ARRAY_SIZE(mmc0_1_pins),
    		},
    		{
        			.name = "mmc0_2_grp",
        			.pins = mmc0_2_pins,
        			.num_pins = ARRAY_SIZE(mmc0_2_pins),
    		},
	};

	
	//示例关注的是 pin mux， 因此省略pin group等其它相关的回调函数

	static const char * const spi0_groups[] = { "spi0_0_grp", "spi0_1_grp" };
	static const char * const mmc0_groups[] = { "mmc0_1_grp", "mmc0_2_grp"};

	static const struct foo_pmx_func foo_functions[] = {
    		{
        			.name = "spi0",
        			.groups = spi0_groups,
        			.num_groups = ARRAY_SIZE(spi0_groups),
    		},
    		{
        			.name = "mmc0",
        			.groups = mmc0_groups,
        			.num_groups = ARRAY_SIZE(mmc0_groups),
    		},
	};

	int foo_get_functions_count(struct pinctrl_dev *pctldev)
	{
    		return ARRAY_SIZE(foo_functions);
	}

	const char *foo_get_fname(struct pinctrl_dev *pctldev, unsigned selector)
	{
    		return foo_functions[selector].name;
	}

	static int foo_get_groups(struct pinctrl_dev *pctldev, unsigned selector,
              const char * const **groups,
              unsigned * const num_groups)
	{
    		*groups = foo_functions[selector].groups;
    		*num_groups = foo_functions[selector].num_groups;
    		return 0;
	}

	int foo_enable(struct pinctrl_dev *pctldev, unsigned selector, unsigned group)
	{
    		u8 regbit = (1 << selector + group);

    		writeb((readb(MUX)|regbit), MUX)
    		return 0;
	}

	void foo_disable(struct pinctrl_dev *pctldev, unsigned selector, unsigned group)
	{
    		u8 regbit = (1 << selector + group);

    		writeb((readb(MUX) & ~(regbit)), MUX)
    		return 0;
	}

	struct pinmux_ops foo_pmxops = {
    		.get_functions_count = foo_get_functions_count,
    		.get_function_name = foo_get_fname,
    		.get_function_groups = foo_get_groups,
    		.enable = foo_enable,
    		.disable = foo_disable,
	};

	static struct pinctrl_desc foo_desc = {
    		...
    		.pmxops = &foo_pmxops,
	};

####3.6  gpio 到 pin脚的映射

某些可以复用为gpio的pin脚， 已经注册为 pin ctroller的pin， 而gpio driver也希望对其进行操作， 对于这样的pin脚， pinctrl subsystem 和gpio subsystem可以对其进行正交操作(即不冲突)， 但是需要建立gpio和pin脚的映射关系

对于gpio， 通常有一个全局的gpio号， 根据gpio号可以找到对应的gpio chip， 以及该gpio在gpio chip内部的编号， pinctrl subsystem作为gpio subsystem的后端， 需要找到该gpio对应的pin脚所在的pin controller才能最终完成操作

**一个pin controller所包含的gpio脚， 可以被划分在多个 gpio chip中， 例如一个soc的pin脚， 被分为几组gpio**

gpio和pin脚的映射关系， 使用如下的数据结构来描述

	struct pinctrl_gpio_range {
		struct list_head node;	//用于将该结构形成链表
		const char *name;		//gpio chip名称
		unsigned int id;		//chip的id， 当某个gpio chip的一段连续的gpio号要映射到2段pin脚编号上时， 需要2个该结构， id用于记录编号
		unsigned int base;		//gpio的起始编号
		unsigned int pin_base;	//pin脚的编号
		unsigned int npins;	//pin脚的数目
		struct gpio_chip *gc;	//gpio chip
	};

在pinctrl subsystem中， 映射关系保存在pinctrl_dev中, 形成链表， 通过pinctrl_add_gpio_range()来添加映射关系

	struct pinctrl_dev{
		......
		struct list_head gpio_ranges;
		......
	}

	void pinctrl_add_gpio_range(struct pinctrl_dev *pctldev, struct pinctrl_gpio_range *range)

在gpio subsystem中， 映射关系保存在gpio chip中， 形成链表， 通过 gpiochip_add_pin_range()来添加映射关系

	struct gpio_chip	{
		......
		struct list_head pin_ranges;	//用于将struct gpio_pin_range形成链表
	}

	struct gpio_pin_range {
		struct list_head node;
		struct pinctrl_dev *pctldev;
		struct pinctrl_gpio_range range;
	};

	int gpiochip_add_pin_range(struct gpio_chip *chip, const char *pinctl_name,
			   unsigned int gpio_offset, unsigned int pin_offset,
			   unsigned int npins)

**pinctrl subsystem 和 gpio subsystem 中保存的gpio 和pin脚的映射关系是一致的**

例如一个 pin controller 中的gpio脚由多个gpio chip来管理， 其中 pin 32 ～ pin 47 映射到 gpio 32 ～ gpio 47， 软件上划归gpio chip a来管理， pin 64 ～ pin 71 映射到gpio 48 ～ gpio 55，软件上划归gpio chip b来管理， 则you 

	struct gpio_chip chip_a;
	struct gpio_chip chip_b;

	static struct pinctrl_gpio_range gpio_range_a = {
    		.name = "chip a",
    		.id = 0,
    		.base = 32,
    		.pin_base = 32,
    		.npins = 16,
    		.gc = &chip_a;
	};

	static struct pinctrl_gpio_range gpio_range_b = {
    		.name = "chip b",
    		.id = 0,
    		.base = 48,
   		.pin_base = 64,
    		.npins = 8,
    		.gc = &chip_b;
	};

	{
    		struct pinctrl_dev *pctl;
    		...
    		pinctrl_add_gpio_range(pctl, &gpio_range_a);
    		pinctrl_add_gpio_range(pctl, &gpio_range_b);
	}


####3.7 pin configuration

很多pin脚可以使用软件进行配置， 例如与电器特性相关， 使一个pin脚处于高阻态或者3态， 或者设置pin脚为内部上拉/下拉， 以便在pin脚未连接或者未接信号时， 处于一个确定的值

例如， 设置一个pin为内部上拉

	#include <linux/pinctrl/consumer.h>

	ret = pin_config_set("foo-dev", "FOO_GPIO_PIN", PLATFORM_X_PULL_UP);

其中

+ “foo-dev” 为 pin controller描述符的名称， 将通过该名称寻找到对应的pin controller的描述符
+ "FOO_GPIO_PIN" 为要操作的pin脚的名称
+ PLATFORM_X_PULL_UP 为要设置的值， 由driver指定

pin controller描述符的confops成员 (即一个struct pinconf_ops变量)中定义了pin configuration相关的回调函数， 由pin controller的driver来实现

	struct pinconf_ops {
		int (*pin_config_get) (struct pinctrl_dev *pctldev, unsigned pin, unsigned long *config);
		int (*pin_config_set) (struct pinctrl_dev *pctldev, unsigned pin, unsigned long config);
		int (*pin_config_group_get) (struct pinctrl_dev *pctldev, unsigned selector, unsigned long *config);
		int (*pin_config_group_set) (struct pinctrl_dev *pctldev, unsigned selector, unsigned long config);
		int (*pin_config_dbg_parse_modify) (struct pinctrl_dev *pctldev, const char *arg, unsigned long *config);
		void (*pin_config_dbg_show) (struct pinctrl_dev *pctldev, struct seq_file *s, unsigned offset);
		void (*pin_config_group_dbg_show) (struct pinctrl_dev *pctldev, struct seq_file *s, unsigned selector);
		void (*pin_config_config_dbg_show) (struct pinctrl_dev *pctldev, struct seq_file *s, unsigned long config);
	};

各回调函数的意义如下：

1. pin_config_get() ： 获取一个pin脚的config值
   + 参数 pctldev ： pin controller描符
   + 参数 pin ： pin脚的编号
   + 参数 config ： 返回获取到的config
2. pin_config_set() : 
   + 参数 pctldev ： pin controller描符
   + 参数 pin ： pin脚的编号
   + 参数 config ： 要设置的config值
3. pin_config_group_get() : 获取一个pin group的config
   + 参数 pctldev ： pin controller描符
   + 参数 selector ： pin group的编号
   + 参数 config ： 返回获取到的config
4. pin_config_group_set() : 设置一个pin group中所有pin脚
   + 参数 pctldev ： pin controller描符
   + 参数 selector ： pin group的编号
   + 参数 config ： 要设置的config值

5. pin_config_dbg_parse_modify()
6. pin_config_dbg_show()
7. pin_config_group_dbg_show()
8. pin_config_config_dbg_show()


###4. pinctrl subsystem提供给gpio subsystem的接口

pinctrl subsystem 作为 gpio的back-end， 向gpio subsystem提供了统一的接口来进行hw上的pin脚的设置

	int pinctrl_request_gpio(unsigned gpio);	//请求将某一个pin脚设置为gpio功能， 使用该pin脚对应的全局gpio编号
	int pinctrl_free_gpio(unsigned gpio);	//同pinctrl_request_gpio()相反

	int pinctrl_gpio_direction_input(unsigned gpio);	//将某一个gpio设置为输入， 使用该gpio的全局gpio编号
	int pinctrl_gpio_direction_output(unsigned gpio);	//将某一个gpio设置为输出， 使用该gpio的全局gpio编号

**这些接口提供给gpio subsystem， 普通的driver应该调用gpio subsystem提供的接口， 不要使用这些接口**

###5. pinctrl 提供给其它driver的接口

普通的device driver调用pinctrl的api的目的有两个：

1. 设定pin脚的功能服用， 即根据function selector和group elector来选择合适的功能
2. 设定pin脚对应的电气特性

###5.1 pin control api

许多内核模块需要利用pinctrl subsystem提供的接口来设置pin脚

	int pin_config_get(const char *dev_name, const char *name, unsigned long *config);
	int pin_config_set(const char *dev_name, const char *name, unsigned long config);

	int pin_config_group_get(const char *dev_name, const char *pin_group, unsigned long *config);
	int pin_config_group_set(const char *dev_name, const char *pin_group, unsigned long config);

####5.2 pin control state

很多设备通常都具有多种不同的电源管理状态(例如 idle， sleep， active)， 不同的状态下， 设备所连接的pin脚的状态具有不同的组合，为此， pinctrl subsystem定义了 pin control state的 
概念， 一个设备可以处于多种pin control state之中的一种， device driver可以切换设备所处的状态， pinctrl subsystem定义了pin control state holder， 用于管理一个设备的所有pin control state

	// pin control state holder， 用于维护一个device的所有的 pin control
	// driver author 无需分配 pin control state holder， pinctrl_get()/pinctrl_put() 会维护引用计数， 并且自动分配/销毁该结构
	struct pinctrl {
		struct list_head node;	//一个pin controller中的pin所连接的device的pin control state holer形成链表， 由 struct pinctrl_dev.p索引
		struct device *dev;
		struct list_head states;		//该设备所有的pin control state， 形成链表
		struct pinctrl_state *state;	//该设备当前的state
		struct list_head dt_maps;		//从device tree中解析出的mapping chunk块
		struct kref users;			//引用计数
	};

	// 设备的一种 pin control state
	struct pinctrl_state {
		struct list_head node;	//一个设备的所有的 pin control state形成链表， 由 struct pinctrl.states来索引
		const char *name;		//该状态的名称
		struct list_head settings; //该状态下的设定值
	};

	//一个 pin control state包含若干个settings
	struct pinctrl_setting {
		struct list_head node;		//一个pin control state的所有settings形成一个链表， 由struct pinctrl_state.node 来索引
		enum pinctrl_map_type type;		//settings的类型
		struct pinctrl_dev *pctldev;	//处理该状态的pinctrl device， type成员为PIN_MAP_TYPE_DUMMY_STATE时不使用
		const char *dev_name;		//device name
		union {				//settings data
			struct pinctrl_setting_mux mux;
			struct pinctrl_setting_configs configs;
		} data;
	};

enum pinctrl_map_type指明了settings data的类型

	enum pinctrl_map_type {
		PIN_MAP_TYPE_INVALID,
		PIN_MAP_TYPE_DUMMY_STATE,
		PIN_MAP_TYPE_MUX_GROUP,	//功能复用相关的settings， 则struct pinctrl_setting.data的类型为struct pinctrl_setting_mux
		PIN_MAP_TYPE_CONFIGS_PIN,	//设定单一pin的config， 则struct pinctrl_setting.data的类型为struct pinctrl_setting_configs
		PIN_MAP_TYPE_CONFIGS_GROUP,//设定pin group的config， 则struct pinctrl_setting.data的类型为struct pinctrl_setting_configs
	};

两种类型的settings data的定义如下

	struct pinctrl_setting_mux {
		unsigned group;
		unsigned func;
	};

	struct pinctrl_setting_configs {
		unsigned group_or_pin;
		unsigned long *configs;
		unsigned num_configs;
	};

pinctrl subsystem定义了几种状态名称

	#define PINCTRL_STATE_DEFAULT "default"
	#define PINCTRL_STATE_IDLE "idle"
	#define PINCTRL_STATE_SLEEP "sleep"
	
device driver设置设备的pin control state的大致逻辑是

1. 获取pin controler state holder
2. 设置设备的pin control state
3. 释放获取pin controler state

####5.3 pin control state相关的api

	// get/put 某个设备的 pin control state holder (根据引用计数自动分配/销毁该结构)
	struct pinctrl * __must_check pinctrl_get(struct device *dev);
	void pinctrl_put(struct pinctrl *p);
	
	// resource management 版本的 pinctrl_get / pinctrl_put
	struct pinctrl * __must_check devm_pinctrl_get(struct device *dev);
	void devm_pinctrl_put(struct pinctrl *p);

	// 根据state name， 寻找某个device 的state对应的struct  pinctrl_state 
	struct pinctrl_state * __must_check pinctrl_lookup_state struct pinctrl *p, const char *name)

	// 将某一个device 的state 切换到指定的状态
	int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *s);

	// 将某一个device 的state切换到指定的状态
	struct pinctrl * __must_check pinctrl_get_select(struct device *dev, const char *name);
	struct pinctrl * __must_check devm_pinctrl_get_select(struct device *dev, const char *name);

	// 将某一个device的状态切换到名为 “default” 的状态
	struct pinctrl * __must_check pinctrl_get_select_default(struct device *dev);
	struct pinctrl * __must_check devm_pinctrl_get_select_default(struct device *dev);



### dts pin configuration

在dts中, pin controller 节点的字节点， 用于描述pin configuration

例如， gpio 20 连接nfc chip NQ210 的enable pin， 则在pin configuration中可以如下描述该pin

	pmx_nfc_reset {
		qcom,pins = <&gp 20>;
		qcom,pin-func = <0>;
		qcom,num-grp-pins = <1>;
		label = "pmx_nfc_disable";
	};

+ qcom,pins ： 指明所使用的pin脚为gpio 20所对应的pin脚， “&gp” 表示引用了描述gpio的dts节点
+ qcom,pin-func ： 指明该pin 脚的function selector为2
+ qcom,num-grp-pins ： 指明pin 脚的数量为1
+ label ： 该node的label， 可以通过label来引用该节点

**device tree中并未定义标准的属性来描述pin configuration， 这些属性大多是由厂商自行定义， 并且由pin controller driver来负责解析， 因此， 在向device tree中添加pin configuration时， 需要弄清楚厂商使用哪些属性来描述pin configuration， pin controller driver使用“postcore_initcall()”等较早的初始化等级，早于一般driver的module_init()入口等级**

nfc chip NQ210 定义了2中电源状态 ： “active” (gpio 20 输出高电平) 和 “suspend” (gpio 20输出低电平)， 为此， 给gpio20的pin configuration 定义了2中pin control state

	pmx_nfc_reset {
		qcom,pins = <&gp 20>;
		qcom,pin-func = <0>;
		qcom,num-grp-pins = <1>;
		label = "pmx_nfc_disable";
	
		nfc_disable_active: active {
			drive-strength = <6>;
			bias-pull-up;
            	};

		nfc_disable_suspend: suspend {
			drive-strength = <6>;
			bias-disable;
		};

	};

分别为 “active” 和 “suspend” 状态定义了不同的电气特性， pinctrl subsystem为devicer tree预设了一些参数可选:

+ "bias-disable"		: disable 任何偏置
+ "bias-high-impedance"	: 设置pin为高阻态， 即3态 
+ "bias-bus-hold"		: 处于bus holder， 即总线上的其它设备可以将该线路拉高/拉低或者设置为3态
+ "bias-pull-up"		: 设置pin脚为pull-up
+ "bias-pull-down"		: 设置pin脚为pull-down
+ "bias-pull-pin-default"	: 设置为默认的pull状态， up还是down要取决于pin controller
+ "drive-push-pull"	: 设置驱动方式为 push-pull mode， 参数会被忽略
+ "drive-open-drain"	: 设置驱动方式为 open-drain mode， 参数会被忽略
+ "drive-open-source"	: 设置驱动方式为 open-source mode， 参数会被忽略
+ "drive-strength"		: 设置驱动电流, 参数单位为mA， pin 脚 为sink(灌电流)或者source(输出电流)
+ "input-enable"		: enable input
+ "input-disable"		: disable input
+ "input-schmitt-enable"	: enable 施密特 触发器
+ "input-schmitt-disable"	: disable 施密特触发器
+ "input-debounce"		: enable 输入去抖， 参数单位为usec
+ "low-power-enable"	: enable low power operation mode (如果pin脚支持几种操作模式)
+ "low-power-disable"	: disable low power operation mode (如果pin脚支持几种操作模式)
+ "output-low"		: 设置pin脚为output mode， 输出低电平
+ "output-high"		: 设置pin脚为output mode， 输出高电平
+ "slew-rate"		: 设置输出电压的摆动速率(如果支持的话)， 

#### device 引用pin configuration

仍以nfc chip NQ210 为例， 其使用i2c接口， 在device tree中的描述为

	&i2c_6 { /* BLSP2 QUP2 */
    		nq@28 {
        			compatible = "qcom,nq-nci";
        			reg = <0x28>;
        			qcom,nq-irq = <&msm_gpio 21 0x00>;
        			qcom,nq-ven = <&msm_gpio 20 0x00>;
        			qcom,nq-clkreq = <&pm8950_gpios 5 0x00>;
        			qcom,nq-firm = <&msm_gpio 16 0x00>;
        			interrupt-parent = <&msm_gpio>;
        			qcom,clk-src = "BBCLK2";
        			interrupts = <21 0>;
        			interrupt-names = "nfc_irq";
        			pinctrl-names = "nfc_active", "nfc_suspend";
        			pinctrl-0 = <&nfc_int_active &nfc_disable_active>;
        			pinctrl-1 = <&nfc_int_suspend &nfc_disable_suspend>;
        			clocks = <&clock_gcc clk_bb_clk2_pin>;
        			clock-names = "ref_clk";
    		};  
	};

NQ210 连接在 i2c-6 上， 地址为0x28， 其中， pinctrl-names指定了该设备的两种pin control state

+ “nfc_active” ： pinctrl-0 属性描述了该状态下的pin configuration， 即引用了gpio20 和 gpio21的 “active”状态的config
+ "nfc_suspend" ： pinctrl-1 属性描述了该状态下的pin configuration， 即引用了gpio20 和 gpio21的 “suspend”状态的config

