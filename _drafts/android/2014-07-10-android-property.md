---
layout: post
title: "android 属性系统"
description:
category: android
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. android属性系统  
  
每个属性都有一个名称和值，他们都是字符串格式。属性被大量使用在Android系统中，用来记录系统设置或进程之间的信息交换。属性是在整个系统中全局可见的。每个进程可以get/set属性。  
  
###2. 属性系统API  

####2.1 native API  
  
使用libcutils中（包含cutils/properties.h）的API来 GET/SET 属性  
  
	int property_get(const char *key, char *value, const char *default_value);
	int property_set(const char *key, const char *value);  
    
libcutils其实调用libc中的 _ _system_property_xxx 函数获得共享内存中的属性。libc的源代码位于：device/system/bionic。   
  
####2.2 java API  
  
Android在Java库中提供了两个API来 GET/SET java属性（不是系统属性）：  
  
	System.getProperty()
	System.setProperty()
    
但是请注意！虽然从语法上面看Java的代码和Native代码非常相近，但是Java版本存储把属性存在其他地方，而不是我们上面提到的属性系统中。在 JVM中有一个hash表来维护Java的属性。所以Java属性和Android属性是不同的，不能用Java API（System.getProperty和System.setProperty）来设置系统属性。也不能通过Native的方法 （property_get和property_set）设置Java的属性。  
  
如果你需要通过Java API来访问系统属性， 可以使用“/frameworks/base/core/java/android/os/SystemProperties.java”中提供的方法。  
  
####2.3 shell 命令  
  
toolbox中提供了 `setprop` 和 `getprop` 命令用于 GET/SET 属性。  
  
###3. 属性初始值  
  
属性服务调用libc中的 _ _system_property_init函数来初始化属性系统的共享内存。当启动属性服务时，将从以下文件中加载默认属性：  
  
+ /default.prop
+ /system/build.prop
+ /system/default.prop
+ /data/local.prop  
     
属性将会以上述顺序加载。后加载的属性将覆盖原先的值。这些属性加载之后，最后加载的属性会被保持在/data/property中。    
还有些属性值通过内核的启动参数来传递， 例如， 通过 androidboot.hardware=xxx 来设置 ro.hardware的属性值。  

###4. 属性的访问限制  
  
在 system/core/init/property_service.c 的 property_perms[] 中定义了可选的属性前缀以及对应的访问UID，GID， 如：  
  
	struct {
		const char *prefix;
		unsigned int uid;
		unsigned int gid;
	} property_perms[] = {
		{ "net.rmnet0.",      AID_RADIO,    0 },
		{ "net.gprs.",        AID_RADIO,    0 },
		{ "net.ppp",          AID_RADIO,    0 },
		{ "net.qmi",          AID_RADIO,    0 },
		{ "net.lte",          AID_RADIO,    0 },
		{ "net.cdma",         AID_RADIO,    0 },
		{ "ril.",             AID_RADIO,    0 },
		{ "gsm.",             AID_RADIO,    0 },
		{ "persist.radio",    AID_RADIO,    0 },
		{ "net.dns",          AID_RADIO,    0 },
		{ "sys.usb.config",   AID_RADIO,    0 },
		{ "net.",             AID_SYSTEM,   0 },
		{ "dev.",             AID_SYSTEM,   0 },
		{ "runtime.",         AID_SYSTEM,   0 },
		{ "hw.",              AID_SYSTEM,   0 },
		{ "sys.",             AID_SYSTEM,   0 },
		{ "sys.powerctl",     AID_SHELL,    0 },
		......
	}
    
+ PA_COUNT_MAX 指定了系统（共享内存区域中）最多能存储多少个属性.    
+ PROP_NAME_MAX 指定了一个属性的key最大允许长度.  
+ PROP_VALUE_MAX 则指定了value的最大允许长度.
  
如果属性名称以“ro.”开头，那么这个属性被视为只读属性。一旦设置，属性值不能改变。  
如果属性名称以“persist.”开头，当设置这个属性时，其值也将写入/data/property。  
如果属性名称以“net.”开头，当设置这个属性时，“net.change”属性将会自动设置，以加入到最后修改的属性名。  

selinux 也限定了属性的访问权限， 在 “external/sepolicy/property_context.te” 中描述了各种前缀的属性的客体类型， 例如

	......
	net.                    u:object_r:system_prop:s0
	dev.                    u:object_r:system_prop:s0
	runtime.                u:object_r:system_prop:s0
	hw.                     u:object_r:system_prop:s0
	sys.                    u:object_r:system_prop:s0
	sys.powerctl            u:object_r:powerctl_prop:s0
	service.                u:object_r:system_prop:s0
	wlan.                   u:object_r:system_prop:s0
	dhcp.                   u:object_r:dhcp_prop:s0
	......

在应用程序要， 若要 set/get 这些属性， 则需要声明相应的selinux权限

###5. ctl.start/ctl.stop 

属性“ ctrl.start ”和“ ctrl.stop ”是用来启动和停止服务。每一项服务必须在/init.rc中定义.系统启动时，与init守护进程将解析init.rc和启动属性服务。一旦收到设置“ ctrl.start ”属性的请求，属性服务将使用该属性值作为服务名找到该服务，启动该服务。这项服务的启动结果将会放入“ init.svc.&;服务名&gt;“属性中 。客户端应用程序可以轮询那个属性值，以确定结果。  
  
###6. 属性action  
  
默认情况下，设置属性只会使"init"守护程序写入共享内存，它不会执行任何脚本或二进制程序。但是，您可以将您的想要的实现的操作与init.rc中某个属性的变化相关联.例如，在默认的init.rc中有：  
  
	# adbd on at boot in emulator
	on property:ro.kernel.qemu=1
		start adbd
	on property:persist.service.adb.enable=1
		start adbd
	on property:persist.service.adb.enable=0
		stop adbd
  
这样，如果你设置persist.service.adb.enable为1 ，"init"守护程序就知道需要采取行动：开启adbd服务。  
  
###7. 系统属性的实现  
  
属性系统的结构如下：  

![属性系统的结构](/images/android/property-system.png)
                          
android的 init 进程实现了property service， 其使用一块共享内存来存储系统中的属性值, 所有进程都可以读取这一块共享内存， 获取属性值， 但是只有property service可写以写这一块共享内存, 因此， 需要设置属性值的进程需要委托property service来改写属性值， poperty settter 通过 unix domain socket 与property service通信
                  
init进程启动property service  
  
	int main(int argc, char **argv){
		queue_builtin_action(property_init_action, "property_init");
	}

	static int property_init_action(int nargs, char **args)
	{
		property_init(load_defaults);
	}
        
property_init中初始化共享内存空间， 丛属性文件中加载属性值  
  
	void property_init(bool load_defaults)
	{
		 init_property_area();
		load_properties_from_file(PROP_PATH_RAMDISK_DEFAULT);
	}
        
http://bbs.pediy.com/showthread.php?p=1249691