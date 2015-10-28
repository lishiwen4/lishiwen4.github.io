---
layout: post
title: "linux gpio 驱动模型"
description:
category: linux-driver
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux gpio driver model  
  
为了给不同GPIO控制器提供一个统一的编程接口，内核提供了一个可选择的实现架构。这个架构被称作"gpiolib"，在这个架构下，每个GPIO控制器被抽象成“struct gpio_chip"，这里GPIO控制器的所有常规信息：
  
+ 设定传输方向（输入/输出）的函数
+ 读写GPIO值的函数
+ 是否调用可睡眠的函数的flag
+ 可选择的用来调试的输出（dump method)  
  
通常gpio_chip是被包含在一个体系相关的结构体内，这个结构体里有一些与gpio状态相关的成员，比如如何寻址，电源管理等等。  
  
###2. 如何启用gpio  
  
Kconfig里应该选择ARCH_REQUIRE_GPIOLIB 或 ARCH_WANT_OPTIONAL_GPIOLIB，并且让<asm/gpio.h>包含<asm-generic/gpio.h>（只要包含就可以，直接间接无所谓）并且定义三个函数：gpio_get_value(), gpio_set_value(), and gpio_cansleep().通常还需要提供ARCH_NR_GPIOS的值（代表有几个GPIO分组）。  
  
上面3个函数实际上要调用底层函数，自己实现后就可以用gpio的架构了。  
  
	#define gpio_get_value        __gpio_get_value
	#define gpio_set_value        __gpio_set_value
 	#define gpio_cansleep         __gpio_cansleep
    
###3. struct gpio_chip  
  
gpio_chip 用于描述一个gpio controller：  
  
  	struct gpio_chip {
		const char		*label;
		struct device		*dev;
		struct module		*owner;
		struct list_head        list;

		int			(*request)(struct gpio_chip *chip,
						unsigned offset);
		void			(*free)(struct gpio_chip *chip,
						unsigned offset);
		int			(*get_direction)(struct gpio_chip *chip,
						unsigned offset);
		int			(*direction_input)(struct gpio_chip *chip,
						unsigned offset);
		int			(*get)(struct gpio_chip *chip,
						unsigned offset);
		int			(*direction_output)(struct gpio_chip *chip,
						unsigned offset, int value);
		int			(*set_debounce)(struct gpio_chip *chip,
						unsigned offset, unsigned debounce);

		void			(*set_pinmux)(int gpio, int alt);
		int			(*get_pinmux)(int gpio);

		void			(*set)(struct gpio_chip *chip,
						unsigned offset, int value);

		int			(*to_irq)(struct gpio_chip *chip,
						unsigned offset);

		void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *chip);
		int			base;
		u16			ngpio;
		struct gpio_desc	*desc;
		const char		*const *names;
		unsigned		can_sleep:1;
		unsigned		exported:1;

		#if defined(CONFIG_OF_GPIO)
			/*
	 		* If CONFIG_OF is enabled, then all GPIO controllers described in the
	 		* device tree automatically may have an OF translation
	 		*/
			struct device_node *of_node;
			int of_gpio_n_cells;
			int (*of_xlate)(struct gpio_chip *gc,
		        const struct of_phandle_args *gpiospec, u32 *flags);
		#endif
		#ifdef CONFIG_PINCTRL
			/*
	 		* If CONFIG_PINCTRL is enabled, then gpio controllers can optionally
	 		* describe the actual pin range which they serve in an SoC. This
	 		* information would be used by pinctrl subsystem to configure
	 		* corresponding pins for gpio usage.
	 		*/
			struct list_head pin_ranges;
		#endif
	};

在gpio chip的成员中  
  
+ label:	bank名  
+ dev：	设备文件
+ owner:	模块所有者  
+ list：	用于将自己保存在链表中  
+ request: 申请gpio， 用于确保gpio只被一人使用   
+ free： 释放request的gpio， 之后其它使用者可以request  
+ get_direction：	获取gpio的方向
+ direction_input： 配置gpio为输入  
+ get： 获取gpio的值  
+ direction_output: 配置gpio为输出  
+ set_debounce: 去抖时间  
+ set_pinmux: 设置引脚复用功能
+ get_pinmux: 获取引脚复用状态  
+ set： 设置引脚输出值  
+ to_irq: 获取引脚的irq号  
+ dbg_show: 打印debug信息  
+ base： gpio的起始编号  
+ ngpio: gpio的个数  
+ desc： 
+ names： 
+ can_sleep: 能否睡眠  
+ exported: 能否导出到用户空间  
  
###4. struct gpio_desc  
  
gpio_desc 用于描述一个gpio口 :  
  
	struct gpio_desc {
		struct gpio_chip	*chip;
		unsigned long		flags;
			/* flag symbols are bit numbers */
			#define FLAG_REQUESTED	0
			#define FLAG_IS_OUT	1
			#define FLAG_EXPORT	2	/* protected by sysfs_lock */
			#define FLAG_SYSFS	3	/* exported via /sys/class/gpio/control */
			#define FLAG_TRIG_FALL	4	/* trigger on falling edge */
			#define FLAG_TRIG_RISE	5	/* trigger on rising edge */
			#define FLAG_ACTIVE_LOW	6	/* sysfs value has active low */
			#define FLAG_OPEN_DRAIN	7	/* Gpio is open drain type */
			#define FLAG_OPEN_SOURCE 8	/* Gpio is open source type */

			#define ID_SHIFT	16	/* add new flags before this one */

			#define GPIO_FLAGS_MASK		((1 << ID_SHIFT) - 1)
			#define GPIO_TRIGGER_MASK	(BIT(FLAG_TRIG_FALL) | BIT(FLAG_TRIG_RISE))

			#ifdef CONFIG_DEBUG_FS
				const char		*label;
			#endif
	};

系统中使用一个 gpio_desc数组来保存所有的gpio_desc结构， 数组的大小为ARCH_NR_GPIOS， 在不同的平台上有所不同
  
	static struct gpio_desc gpio_desc[ARCH_NR_GPIOS];
    
###5. 注册gpio chip  
  
上一节讲过， 所有的gpio都有一个全局唯一的编号， 对应的描述结构为 gpio_desc[idx]， 当操作某一个gpio的时候， 从全局数组 gpio_desc[] 找到对应的 gpio_desc 结构， 然后就可以确定对应的的 gpio_chip, 再调用对应chip相关的的操作函数， 而绑定对应的 gpio_desc 和 gpio_chip 依靠的是gpiochip_add  

	int gpiochip_add(struct gpio_chip *chip)  
    	{  
        		unsigned long   flags;  
        		int status = 0;  
        		unsigned    id;  
        		int base = chip->base;   //获取gpio_chip基数  
        		//验证gpio的基数,gpio的最后一个io的编号正确性  
        		if ((!gpio_is_valid(base) || !gpio_is_valid(base + chip->ngpio - 1))&& base >= 0) {     
            		status = -EINVAL;  
            		goto fail;  
        		}  

        		spin_lock_irqsave(&gpio_lock, flags);   //上自旋锁  
        		if (base < 0) {  //若gpio的基数小于0  
            		base = gpiochip_find_base(chip->ngpio);  //根据gpio个数分配新的基数  
            		if (base < 0) {  
                			status = base;  
                			goto unlock;  
            		}  
            		chip->base = base;   //设置新的基数  
        		}  
        		for (id = base; id < base + chip->ngpio; id++) {  
            		if (gpio_desc[id].chip != NULL) {   //判断gpio_desc是否给其他gpio_chip管理  
                		status = -EBUSY;  
                		break;  
            		}  
        		}  
        		if (status == 0) {  
            		for (id = base; id < base + chip->ngpio; id++) {  //填充对应的全局gpio_desc数组项  
                			gpio_desc[id].chip = chip;  //gpio_chip  
                			gpio_desc[id].flags = !chip->direction_input?(1 << FLAG_IS_OUT):0; //设置输入输出标志位  
            		}  
        		}  
        		of_gpiochip_add(chip);  
    	unlock:  
        		spin_unlock_irqrestore(&gpio_lock, flags);  //解自旋锁  
        		if (status)  
            		goto fail;  
        		status = gpiochip_export(chip); //gpio_chip创建用户接口  
        		if (status)  
            		goto fail;  
        		return 0;  
    	fail:  
        		pr_err("gpiochip_add: gpios %d..%d (%s) failed to register\n",chip->base,chip->base+chip->ngpio-1,chip->label?:"generic");  
        		return status;  
    	}  
    	EXPORT_SYMBOL_GPL(gpiochip_add);  
        
在 gpiochip_add() 中根据参数chip的 gpio_chip.base, 来确定这一个chip上的gpio的起始编号， 根据gpio_chip.ngpio来确定这一chip上的gpio数量， 从而将 全局数组 gpio_desc[] 的 gpio_desc[gpio_chip.base] 到 gpio_desc[gpio_chip.base + gpio_chip.ngpio -1] 的项的 chip 指向 gpiochip_add 中传递进来的 chip参数。  
  
	
**使用 gpiochip_remove() 能够移除在全局数组 gpio_desc 中添加的项, 即注销一个 gpio_chip**
  
###6. 内核中的gpio操作接口  
  
linux kernel中提供了一组api供device driver来操控gpio

+ int gpio_set_debounce(unsigned gpio, unsigned debounce)
+ void __gpio_set_value(unsigned gpio, int value)
+ int __gpio_cansleep(unsigned gpio)
+ int __gpio_get_value(unsigned gpio)
+ int gpio_request(unsigned gpio, const char *label)
+ void gpio_free(unsigned gpio)
+ int __gpio_to_irq(unsigned gpio)
+ const char *gpiochip_is_requested(struct gpio_chip *chip, unsigned offset)
+ void gpiochip_remove_pin_ranges(struct gpio_chip *chip)
+ int gpiochip_add_pin_range(struct gpio_chip *chip, const char *pinctl_name,
			   unsigned int gpio_offset, unsigned int pin_offset,
			   unsigned int npins)
+ struct gpio_chip *gpiochip_find(void *data, int (*match)(struct gpio_chip *chip, void *data))
+ int gpio_set_debounce(unsigned gpio, unsigned debounce)
+ int gpio_direction_output(unsigned gpio, int value)
+ int gpio_request_one(unsigned gpio, unsigned long flags, const char *label)
+ int gpio_get_value_cansleep(unsigned gpio)
+ void gpio_free_array(const struct gpio *array, size_t num)
+ int gpio_request_array(const struct gpio *array, size_t num)
+ int gpio_direction_input(unsigned gpio)
+ void gpio_set_value_cansleep(unsigned gpio, int value)

###7. gpio与pinctrl subsystem

linux kernel在3.14之后， 为GPIO subsystem引入了pinctrl subsystem， 统一了pin脚的操作