---
layout: post
title: "linux内核模块版本检查"
description: 
category: linux-kernel
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux的内核版本号  

linux的内核版本号(KERNELRELEASE)在编译的过程中生成， 存储在“include/config/kernel.release”文件中, 在KERNEL的Makefile中有

	KERNELRELEASE = $(shell cat include/config/kernel.release 2> /dev/null)

先来看“include/config/kernel.release”文件的生成过程， 同样是在kernel的Makefile中

	# Store (new) KERNELRELASE string in include/config/kernel.release
	include/config/kernel.release: include/config/auto.conf FORCE
    		$(Q)rm -f $@
    		$(Q)echo "$(KERNELVERSION)$$($(CONFIG_SHELL) $(srctree)/scripts/setlocalversion $(srctree))" > $@

可以看到， KERNELRELEASE由两部分组成: 

+ KERNELVERSION ： 这一部分是kernel源码的版本信息, 对于某一版本的源码来说， 是固定不变的
+ 由setlocalversion脚本生成的版本信息， 这一部分被称为 LOCALVERSION， 在编译过程中生成  

####1.1 KERNELVERSION

在linux kernel的Makefile文件的最上面可以看到

	VERSION = 3
   	PATCHLEVEL = 10
   	SUBLEVEL = 49
   	EXTRAVERSION =
   	NAME = TOSSUG Baby Fish

而在Makefile中还有  

	KERNELVERSION = $(VERSION)$(if $(PATCHLEVEL),.$(PATCHLEVEL)$(if $(SUBLEVEL),.$(SUBLEVEL)))$(EXTRAVERSION)

在本例中， 生成的 KERNELVERSION 为 “3.10.49”

####1.2 LOCALVERSION

再来看 setlocalersion 脚本生成的LOCALVERSION

在脚本 setlocalersion 中有, 按照顺序来看LOCALVERSION的组成

	# localversion* files in the build and source directory
	res="$(collect_files localversion*)"
	if test ! "$srctree" -ef .; then
    		res="$res$(collect_files "$srctree"/localversion*)"
	fi

上面代码， 执行如下的步骤：

1. 首先，若编译kernel的out目录中存在以“localversion”开头的文件， 则将其内容添加到LOCALVERSION中  
2. 其次，若kernel的src目录中存在以“localversion”开头的文件， 则也将其内容添加到LOCALVERSION中

	# CONFIG_LOCALVERSION and LOCALVERSION (if set)
	res="${res}${CONFIG_LOCALVERSION}${LOCALVERSION}"

如上两行代码， 在定义了CONFIG_LOCALVERSION或者LOCALVERSION变量时， 依次将它们添加到LOCALVERSION中

	# scm version string if not at a tagged commit
	if test "$CONFIG_LOCALVERSION_AUTO" = "y"; then
    	    # full scm version string
    	    res="$res$(scm_version)"
	else
    	    # append a plus sign if the repository is not in a clean
    	    # annotated or signed tagged state (as git describe only
    	    # looks at signed or annotated tags - git tag -a/-s) and
    	    # LOCALVERSION= is not specified
    	    if test "${LOCALVERSION+set}" != "set"; then
        	        scm=$(scm_version --short)
        	        res="$res${scm:++}"
    	    fi  
	fi

上面的代码， 执行如下步骤：

1.  如果 CONFIG_LOCALVERSION_AUTO=y， 则会在最终的LOCALVERSION后追加一段由版本控制工具git或者svn生成的版本信息， 对于git， 如果打了tag，则会使用tag， 否则， 使用字母‘g’再加上最新的commit ID的前7位， 例如 “3.10.49-gde6b4c1”, 若存在未被追踪的修改， 则还会添加" -dirty"字样
2. 如果 CONFIG_LOCALVERSION_AUTO=n， 且未指定LOCALVERSION变量则会在最终的LOCALVERSION后追加‘+’， 例如 “3.10.49+”

最后按照不同的组合情况， 列出LOCALVERSION的取值， 其中"\[\]"表示可选， "&lt;&gt;"表示一定存在 ：

+ CONFIG_LOCALVERSION_AUTO=y  

```
	LOCALVERSION =	[out/localversion*] [src/localversion*] [CONFIG_LOCALVERSION] [LOCALVERSION] <version tool info>
```

+ CONFIG_LOCALVERSION_AUTO=y， 未定义LOCALVERSION   

```
	LOCALVERSION =	[out/localversion*] [src/localversion*]
			[CONFIG_LOCALVERSION] [LOCALVERSION]
			<'+'>	
```

+ CONFIG_LOCALVERSION_AUTO=y， 定义LOCALVERSION    

```
	LOCALVERSION =	[out/localversion*] [src/localversion*]
			[CONFIG_LOCALVERSION] [LOCALVERSION]
```

**按照上面的规则, 如果不想要使用任何localversion信息， 则可将CONFIG_LOCALVERSION_AUTO=n， 并且指定“LOCALVERSION”变量为空即可**  
  
通常， 可以在执行Makefile的参数里面指定“LOCALVERSION”或者将其添加到环境变量中去

Android中，LOCALVERSION 通常只包含git的版本信息

在build linux kernel的过程中， 如下的几个version 信息会被导出到环境变量  

	export VERSION PATCHLEVEL SUBLEVEL KERNELRELEASE KERNELVERSION

使用uname或者 cat /proc/version 可在目标系统上查看内核版本号

###2. vermagic  
  
linux的vermagic 用于内核模块的版本检查， 其值由kernel Makefile 中的 “VERMAGIC_STRING”来表示， 在编译kernel的过程中生成  

	file "include/generated/utsrelease.h"

	#define UTS_RELEASE "3.10.49-gc7c11ec-01090-g82b47c9"


	file "include/linux/vermagic.h"

	#include <generated/utsrelease.h>

	/* Simply sanity version stamp for modules. */
	#ifdef CONFIG_SMP
	#define MODULE_VERMAGIC_SMP "SMP "
	#else
	#define MODULE_VERMAGIC_SMP ""
	#endif
	#ifdef CONFIG_PREEMPT
	#define MODULE_VERMAGIC_PREEMPT "preempt "
	#else
	#define MODULE_VERMAGIC_PREEMPT ""
	#endif
	#ifdef CONFIG_MODULE_UNLOAD
	#define MODULE_VERMAGIC_MODULE_UNLOAD "mod_unload "
	#else
	#define MODULE_VERMAGIC_MODULE_UNLOAD ""
	#endif
	#ifdef CONFIG_MODVERSIONS
	#define MODULE_VERMAGIC_MODVERSIONS "modversions "
	#else
	#define MODULE_VERMAGIC_MODVERSIONS ""
	#endif
	#ifndef MODULE_ARCH_VERMAGIC
	#define MODULE_ARCH_VERMAGIC ""
	#endif

	#define VERMAGIC_STRING                         \
    		UTS_RELEASE " "                         \
    		MODULE_VERMAGIC_SMP MODULE_VERMAGIC_PREEMPT             \
    		MODULE_VERMAGIC_MODULE_UNLOAD MODULE_VERMAGIC_MODVERSIONS   \
    		MODULE_ARCH_VERMAGIC

概括起来， 在vermagic中， 只有内核版本是一定会被包含的， 而其它的要看具体的kernel配置  

	VERMAGIC_STRING = 	<UTS_RELEASE> ["SMP"] ["preempt"] ["mod_unload"]["modversions"]

**其中UTS_RELEASE即为上面说说明的KERNELRELEASE**， 由kernel的Makefile来生成,过程如下

	uts_len := 64
	define filechk_utsrelease.h
    	if [ `echo -n "$(KERNELRELEASE)" | wc -c ` -gt $(uts_len) ]; then \
      		echo '"$(KERNELRELEASE)" exceeds $(uts_len) characters' >&2;    \
      		exit 1;                                                         \
    	fi;                                                               \
    	(echo \#define UTS_RELEASE \"$(KERNELRELEASE)\";)
	endef

	include/generated/utsrelease.h: include/config/kernel.release FORCE
    	$(call filechk,utsrelease.h)

在linux系统上， 可以通过“cat /proc/version” 来查看linux 系统的 vermagic
对于linux kernel module， 可以通过“modinfo filename” 的方式来查看module的 vermagic

###3. export symbol CRC checksum

CRC检查用于保证linux kernrel 和 kernel module的二进制兼容性，即要求两个不变 

+ 语法保持不变 : 遵守这个条件，说明如果模块在新内核下重新编译，那应该没有任何语法问题, 即导出符号的类型名没有变化，如果是函数，则要求参数和返回值类型没有任何变化；如果这些类型是结构体的话，结构体的成员名也没有有任何变化。
+ 语义保持不变 : 这要求符号的类型不能有变化，如果类型本身是结构体(struct)，则它成员的类型不能有变化，成员在结构体内的位置不能有变化，以及成员本身不能增删。

上述两点总结起来就是：导出符号的签名不能有变化

导出符号的CRC计算过程比较复杂，这里不涉及原理， 只需要知道：
	
+ 语法属性：任保一个类型名，或者变量名发生变化，都会造成最终的CRC发生变化
+ 语议属性：任何一个类型变化，或者结构成员出现位置调整，都会造成最终的CRC发生变化

**kernel 导出的每一个symbol都有一个CRC校验值， module中保存了每一个使用到的由kerenel导出的symbol的CRC校验值， 在module加载时， 会检查两者的校验值若不匹配则模块会被拒绝加载**

当内核编译时， 若““CONFIG_MODVERSIONS”选项被打开， 则每个通过宏EXPORT_SYMBOL定义而输出的符号名都会被计算出一个CRC校验值
CRC校验值通过命令genksyms来生成的,它会讲一个符号中的所有定义，包括结构，联合，枚举，类型定义的结构解释开，变成它编译前的最终内容，然后将其内容用CRC算法生成一个整数，这样如果一个符号中的任何定义发生变化，这个整数将会不一样 

在编译kernel完成之后， 生成的 “Module.symvers” 包含了kernel的所有的导出符号的CRC校验值， 例如， 在某一个系统上， printk的CRC校验值为 0x27e1a049

	0x27e1a049	printk	vmlinux	EXPORT_SYMBOL

同样的， 在build module时， 若其导出了任何符号， 都会在生成的“Module.symvers”中有记录， 以一个导出了“hhhhh()”的hello.ko为例

	0x58ac6358	hhhhh	/home/sven/test/hello	EXPORT_SYMBOL

事实上， kernel module的导出符号及其CRC校验值，会被附加到ko文件的特殊的section中， 在模块加载时， 这些信息会被添加到内核的符号表中， 例如， hello.ko中的几个特殊的section

	[ 6] __ksymtab_strings PROGBITS         0000000000000000  00000080  0000000000000006  0000000000000000   A       0     0     1  
	[11] __ksymtab         PROGBITS         0000000000000028  000001b0  0000000000000010  0000000000000000  WA       0     0     8  
	[12] .rela__ksymtab    RELA             0000000000000000  00010320  0000000000000030  0000000000000018          34    11     8  
	[13] __kcrctab         PROGBITS         0000000000000038  000001c0  0000000000000008  0000000000000000  WA       0     0     8  
	[14] .rela__kcrctab    RELA             0000000000000000  00010350  0000000000000018  0000000000000018          34    13     8  

而当module 引用了由kernel或者其它的模块导出的符号时， 在编译的过程中， 需要读取kernel或者其它的module的“Module.symvers”， 依赖的其它的module的“Module.symvers”需要放置到当前driver的source目录下， 或者在Makefile中使用“KBUILD_EXTRA_SYMBOLS=”来指定所依赖的模块的Module.symvers文件的内容

在生成的ko文件中， 会有一个 “__versions” 节, 里面保存了module所引用的符号和对应CRC校验值， 在build module过程中的 xxx.mod.c中可以看到完整的内容, 还可以使用modeprobe的“--dump-modversions”来查看  

如果你的模块在一个“CONFIG_MODVERSIONS”被打开的内核环境中编译， 你的每一个.c源码文件中都必须包含“linux/modversions.h”文件， 可以通过gcc的“-include”选项减轻负担

	ifdef CONFIG_MODULES
	ifdef CONFIG_MODVERSIONS
	MODFLAGS += -DMODVERSIONS -include $(HPATH)/linux/modversions.h
	endif 

###4. 模块加载时的检查步骤

首先， 会进行模块签名验证，然后才是版本检查

版本检查又分为 vermagic 检查和 CRC校验检查， 这又和内核的编译选项"CONFIG_MODVERSIONS"相关

+ **kernel 打开 CONFIG_MODVERSIONS， module 不打开 CONFIG_MODVERSIONS** ： 将不会检查CRC， 但是会完整匹配vermagic, 但是因为两者的vermagic中“modversions”的差别, 模块将不能加载成功
+ **kernel 打开 CONFIG_MODVERSIONS， module 打开 CONFIG_MODVERSIONS**， 将会检查CRC， 然后匹配vermagic的第一个空格后的部分， 例如， 若vermagic为“3.10.49-gc7c11ec SMP preempt mod_unload modversions aarch64”， 则只会使用"SMP preempt mod_unload modversions aarch64来匹配", 即SMP， 模块卸载等关键特性还是需要进行检查  
+ **kernel 不打开 CONFIG_MODVERSIONS， module 打开 CONFIG_MODVERSIONS**， 将会完整匹配vermagic, 但是因为两者的vermagic中“modversions”的差别, 模块将不能加载成功
+ **kernel 不打开 CONFIG_MODVERSIONS， module 不打开 CONFIG_MODVERSIONS**， 完整匹配vermagic

###5. 忽略版本检查  

modprobe 还提供了两个选项， 用于在加载模块时忽略vermagic或者CRC检查， 强制加载一个模块

+ --force-vermagic ： 忽略vermagic检查
+ --force-modversion ： 忽略CRC检查
+ -f : 相当于同时使用 --force-vermagic 和 --force-modversion

但是若存在兼容性问题， 将会导致kernel crash
