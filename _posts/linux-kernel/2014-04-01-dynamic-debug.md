---
layout: post
title: "linux dynamic debug"
description: 
category: linux-kernel
tags: [linux, debug, driver]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. dynamic debug  
  
linux kerne 的 dynamic debug 特性使得你能在系统运行时动态 打开/关闭 使用如下的内核消息打印接口
  
+ pr_debug()
+ dev_dbg()
+ print_hex_dump_debug()
+ print_hex_dump_bytes()  
+ dev_dbg_ratelimited()
+ pr_debug_ratelimited()

**若编译的是debug版的kernel， 则pr_debug() dev_dbg() 都会被打开， 不能dynamic control**

dynamic debug 需要debugfs的支持， 在debugfs挂载后才能工作, 在编译内核时还需要打开"CONFIG_DYNAMIC_DEBUG".  

###2. 如何开关dynamic debug  
  
dynamic debug 使用"&lt;debugfs&gt;/dynamic_debug/control"属性（debugfs一般挂载在"/sys/kernel/debug"， 即为"/sys/kernel/debug/dynamic_debug/control"）来控制开关。  
  
	cat /sys/kernel/debug/dynamic_debug/control
    
能看到所有已经打开的dynamic debug消息, 例如:  

	/usr/src/packages/BUILD/sgi-enhancednfs-1.4/default/net/sunrpc/svc_rdma.c:341 [svcxprt_rdma]svc_rdma_init =_ "\011max_inline       : %d\012"  
        
以上的输出信息详细记录了每一条dynamic debug消息的详细信息

dynamic debug最细可以控制每一条打印语句的开关， 其格式为

	"file:line [module]function flags format"  

我们可以使用这些规则来匹配要打开的dynamic debug消息:  
  
例如：

**1. 打开内核源码中“kxtj2.c”中的dynamic debug**
  
	echo 'file kxtj2 +p' > /sys/kernel/debug/dynamic_debug/control  
    
    + 为加上某个flag
    - 为减去某个flag
    = 为设置为指定的标志
    p 为打开dynamic debug  
    f 为在dynamic debug消息中包含函数名
    l 为在dynamic debug消息中包含行号  
    m 为在dynamic debug消息中包含模块名称  
    t 为在dynamic debug消息中包含线程ID（正对非中断上下文中的消息）
    — 代表无标志（即清除上述所有标志） 
    
**2. 打开内核模块“akm”中的dynamic debug**
  
	echo 'module akm +p' > /sys/kernel/debug/dynamic_debug/control  
    
**3. 可以使用‘line number’ 或者 'line number - number‘来选择指定的行**  
  
	/* 打开第12行的dynamic debug消息 */
	echo 'file kxtj2 line 12 +p' > /sys/kernel/debug/dynamic_debug/control
	echo 'module akm line 12 +p' > /sys/kernel/debug/dynamic_debug/control
    	
	/* 打开第12行到230行的dynamic debug消息 */
	echo 'file kxtj2 line 12-230 +p' > /sys/kernel/debug/dynamic_debug/control
	echo 'module akm line 12-230 +p' > /sys/kernel/debug/dynamic_debug/control
	
    	/* 打开从第12行开始到末尾的dynamic debug消息 */
    	echo 'file kxtj2 line 12- +p' > /sys/kernel/debug/dynamic_debug/control
    	echo 'module akm line 12- +p' > /sys/kernel/debug/dynamic_debug/control
	
	/* 打开开始到第230行的dynamic debug消息 */
	echo 'file kxtj2 line -230 +p' > /sys/kernel/debug/dynamic_debug/control
	echo 'module akm line -230 +p' > /sys/kernel/debug/dynamic_debug/control
    
    
**4. 使用'func function name'来打开某个函数中的所有dynamic debug  **
  
	echo 'file kxtj2 func kxtj2_probe +p' > /sys/kernel/debug/dynamic_debug/control
	echo 'module akm func init +p' > /sys/kernel/debug/dynamic_debug/control
    
###3. 如何打开boot阶段的dynamic debug  
  
dynamic debug 需要debugfs中的属性来控制开关， 那么打开系统boot时的dynamic debug消息呢：  
  
####3.1 对于built-in module

可以向内核传递启动参数“dyndbg”来控制dynamic debug消息的开关, 如：  

	dyndbg="file ec.c +p"

在使用grub引导的linux系统上， 通常可以修改“/boot/grtub/grub.cfg”, "/boot/efi/EFI/fedora/grub.cfg"之类的文件来添加内核启动参数， 如：  

	menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-e9d9a772-1fd1-4900-b6ef-ec5280dca270' {
	recordfail
        gfxmode $linux_gfx_mode
        insmod gzio
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos6'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos6 --hint-efi=hd0,msdos6 --hint-baremetal=ahci0,msdos6  e9d9a772-1fd1-4900-b6ef-ec5280dca270
        else
          search --no-floppy --fs-uuid --set=root e9d9a772-1fd1-4900-b6ef-ec5280dca270
        fi
        **linux   /boot/vmlinuz-3.5.0-17-generic root=UUID=e9d9a772-1fd1-4900-b6ef-ec5280dca270 ro   quiet splash $vt_handoff**
        initrd  /boot/initrd.img-3.5.0-17-generic
	}
  
在以 linux 开头的那一行后以“name=value”的形式添加内核启动参数。  
  
修该“grub.cfg”文件的方式添加的内核启动参数的方式， 在执行`update-grub`和`grub2-mkconfig`之类的命令更新grub配置文件后就会被打回原型， 针对这种情况， 可以修该grub配置文件的模板“/etc/default/grub”  

	GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
	GRUB_CMDLINE_LINUX=""
    
将需要添加的内核启动参数以“name=val”的形式添加到“GRUB_CMDLINE_LINUX=”的后面

使用uboot引导的系统上, 以android为例， 它的启动参数放在cmdline中（和ramdis kernel一起打包成boot.img），cmdline文件如下  

	console=ttyS0,115200 console=logk0 earlyprintk=nologger loglevel=8 drm.debug=0x3e kmemleak=off ptrace.ptrace_can_access=1 androidboot.bootmedia=sdcard androidboot.hardware=K013 androidboot.spid=xxxx:xxxx:xxxx:xxxx:xxxx:xxxx androidboot.serialno=01234567890123456789012345678901 oops=panic panic=40 nmi_watchdog=panic softlockup_panic=1 vmalloc=172M emmc_ipanic.ipanic_part_number=3 vga=current i915.modeset=1 drm.vblankoffdelay=1 acpi_backlight=vendor i915.i915_rotation=0 i915.mipi_panel_id=3

可以直接boot.img, 修改cmdline文件来添加需要的启动参数。  
cmdline文件的内容其实是android源码中的device 下的 BoardConfig.mk中的BOARD_KERNEL_CMDLINE变量中,可以在改变量中添加内核启动参数， 然后重新build.img  
  
####3.2 可加载的内核模块:  
  
可加载的模块有几种方式打开dynamic debug按照其判定顺序如下：  

**1. 使用modprobe 加载模块时 解析 “/etc/modprobe.d/*.conf” 中的参数， 如：**

	options foo dyndbg=+pt  
    
**2. 执行modeprobe时， 解析“/proc/cmdline”中的启动参数,如**

	foo.dyndbg=" func bar +p; func buz +mp"  
    
**3. 执行modeprobe时传递的参数， 如**

	modprobe foo dyndbg==pmf