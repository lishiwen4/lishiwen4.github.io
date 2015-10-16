---
layout: post
title: "android.mk"
description:
category: android
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. android.mk  
  
Android.mk 用来向编译系统描述你的源代码。具体来说：该文件是GNU Makefile的一小部分，会被编译系统解析一次或多次。你可以在每一个Android.mk 中定义一个或多个模块，你也可以在几个模块中使用同一个源代码文件。编译系统为你处理许多细节问题。  
  
Android.mk  中的内容大致可以分为3种:  
  
1. 预定义变量(例如 LOCAL_MODULE)  
2. 调用预定义函数(例如 $(call my-dir))  
3. include 预定义Android.mk文件(例如include(CLEAR_VARS))  
  
###2. 预定义变量  
  
**1. LOCAL_PATH** : 一个Android.mk file首先必须定义好LOCAL_PATH变量。它用于在开发树中查找源文件， 通常使用
	
    LOCAL_PATH:=$(call my-dir)  
    
来赋值， `my-dir`为android编译系统中预定义的函数， 用于返回当前路径
  
**2. LOCAL_PACKAGE_NAME** : 生成package的名字(例如指定为app， 则生成app.apk)， 编译apk时需要指定
  
**3. LOCAL_MODULE** : 以标识你在Android.mk文件中描述的每个模块。名称必须是唯一的，而且不包含任何空格。注意编译系统会自动产生合适的前缀和后缀，例如，一个被命名为'foo'的共享库模块，将会生成'libfoo.so'文件  
  
**4. LOCAL_IS_HOST_MODULE** ： 标识模块是为android编译还是为编译时使用的主机(例如)编译
  
+ 如果该变量为true， 则编译系统内置的变量"my_prefix"值为“HOST_”， 否则为“TARGET_”   
+ 如果该变量为true， 则编译系统内置的变量“my_host”值为“host-”，否则为“my_host:= ”  
  
**5. LOCAL_MODULE_TAGS** : 可选 “eng”， “userdebug“，“user”， ”optional”4种， 标识在对应的编译类型下才生效， 例如，赋值为“eng”则该模块只有在编译eng image时才会被编译，默认情况下， 编译类型为"eng"  
  
注意， “optional”意思为“don't include this”， 即在最终生成的image中不包含该模块， 使用该选项时，模块虽然会被编译， 但是不会被打包到最终的image中， 如果需要打包进去， 还需要将该模块添加到 “PRODUCT_PACKAGES” 变量中去，例如  
  
	PRODUCT_PACKAGES += \
 		i2c-tools
  
一般在模块需要编译到任何image中时， 将LOCAL_MODULE_TAGS 赋值为“optional", 然后添加到 PRODUCT_PACKAGES 变量中去  
  
**6. LOCAL_MODULE_CLASS** : 标识了所编译模块的类型， 模块的类型决定了编译模块时中间文件存放的位置， 可选的值有
  
+ ETC ： 
+ EXECUTABLES ： 
+ SHARED_LIBRARIES ：   
+ STATIC_LIBRARIES ：
+ FAKE ：
+ JAVA_LIBRARIES
+ APPS ： 

对于recovery模块， 还有：  
  
+ RECOVERY_EXECUTABLES
+ UTILITY_EXECUTABLES
  
**7. LOCAL_MODULE_PATH** : 表示模块生成的目标将最终存放的路径, 如果不指定， 默认的目标路径为  
  
	LOCAL_MODULE_PATH := $($(my_prefix)OUT$(partition_tag)_$(LOCAL_MODULE_CLASS)) 
  
例如， 对于 LOCAL_MODULE_CLASS 为 ETC 的模块， 则最终拷贝到 /system/etc中  

如果不希望使用默认的路径， 则可使用 LOCAL_MODULE_PATH 来指定位置, 在指定目标位置时，"build/core/envsetup.mk"中预定义了一些位置, 其中部分如下：

	TARGET_OUT_ROOT : out/target
	TARGET_PRODUCT_OUT_ROOT := $(TARGET_OUT_ROOT)/product			#即 out/target/product/
	PRODUCT_OUT := $(TARGET_PRODUCT_OUT_ROOT)/$(TARGET_DEVICE)		#即 out/target/product/xxx/
	TARGET_OUT_INTERMEDIATES := $(PRODUCT_OUT)/obj			#即 out/target/product/xxx/obj/
	TARGET_OUT_GEN := $(PRODUCT_OUT)/gen					#即 out/target/product/xxx/gen/
	TARGET_OUT := $(PRODUCT_OUT)/$(TARGET_COPY_OUT_SYSTEM)			#即 out/target/product/xxx/system/
	TARGET_OUT_FAKE := $(PRODUCT_OUT)/fake_packages			#即 out/target/product/xxx/fake_packeges/
	TARGET_OUT_DATA := $(PRODUCT_OUT)/$(TARGET_COPY_OUT_DATA)		#即 out/target/product/xxx/fake_packeges/
	TARGET_OUT_CACHE := $(PRODUCT_OUT)/cache				#即 out/target/product/xxx/cache/
	TARGET_OUT_VENDOR := $(PRODUCT_OUT)/$(TARGET_COPY_OUT_VENDOR)		#即 out/target/product/xxx/system/vendor/
	TARGET_OUT_OEM := $(PRODUCT_OUT)/$(TARGET_COPY_OUT_OEM)		#即 out/target/product/xxx/oem/
	TARGET_OUT_UNSTRIPPED := $(PRODUCT_OUT)/symbols			#即 out/target/product/xxx/symbols/
	TARGET_ROOT_OUT := $(PRODUCT_OUT)/$(TARGET_COPY_OUT_ROOT)		#即 out/target/product/xxx/root/
	TARGET_RECOVERY_OUT := $(PRODUCT_OUT)/$(TARGET_COPY_OUT_RECOVERY)	#即 out/target/product/xxx/recovery/
	TARGET_OUT_EXECUTABLES := $(TARGET_OUT)/bin				#即 out/target/product/xxx/bin/
	TARGET_OUT_OPTIONAL_EXECUTABLES := $(TARGET_OUT)/xbin			#即 out/target/product/xxx/xbin/
	TARGET_OUT_JAVA_LIBRARIES := $(TARGET_OUT)/framework			#即 out/target/product/xxx/framework/
	TARGET_OUT_APPS := $(TARGET_OUT)/app					#即 out/target/product/xxx/app/
	TARGET_OUT_APPS_PRIVILEGED := $(TARGET_OUT)/priv-app			#即 out/target/product/xxx/priv-app/
	TARGET_OUT_ETC := $(TARGET_OUT)/etc					#即 out/target/product/xxx/etc/
	TARGET_OUT_DATA_APPS := $(TARGET_OUT_DATA)/app			#即 out/target/product/xxx/data/app/
	TARGET_OUT_VENDOR_OPTIONAL_EXECUTABLES := $(TARGET_OUT_VENDOR)/xbin	#即 out/target/product/xxx/system/vendor/xbin/
	TARGET_OUT_VENDOR_APPS := $(TARGET_OUT_VENDOR)/app			#即 out/target/product/xxx/system/vendor/app/
	TARGET_OUT_VENDOR_ETC := $(TARGET_OUT_VENDOR)/etc			#即 out/target/product/xxx/syatem/vendor/etc/
	TARGET_OUT_OEM_APPS := $(TARGET_OUT_OEM)/app				#即 out/target/product/xxx/oem/app
	TARGET_OUT_OEM_ETC := $(TARGET_OUT_OEM)/etc				#即 out/target/product/xxx/oem/etc	
	TARGET_ROOT_OUT_BIN := $(TARGET_ROOT_OUT)/bin				#即 out/target/product/xxx/root/bin
	TARGET_ROOT_OUT_SBIN := $(TARGET_ROOT_OUT)/sbin			#即 out/target/product/xxx/sbin
	TARGET_ROOT_OUT_ETC := $(TARGET_ROOT_OUT)/etc				#即 out/target/product/xxx/etc
	TARGET_ROOT_OUT_USR := $(TARGET_ROOT_OUT)/usr				#即 out/target/product/xxx/usr

**注意在64bit的目标平台上， 在编译动态链接库时， 通常会编译64bit和32bit版本， 若编译动态链接库时，指定了LOCAL_MODULE_PATH， 则会导致打包img时， 只包含32bit版本， 且放置在/system/lib64/中， 此时应该使用LOCAL_MODULE_RELATIVE_PATH来指定同台库的目标路径**
  
**8. LOCAL_SRC_FILES** : 模块的源码文件  
  
**9. LOCAL_MODULE_SUFFIX** : 生成的模块的后缀，例如“.apk” ".so"等，编译系统会自动选择正确的后缀， 一般无需指定  
  
**10. LOCAL_SHARED_LIBRARIES** : 指定模块依赖的动态连接库， 例如  
  
	LOCAL_SHARED_LIBRARIES ：= libc libcutils  
    
**11. LOCAL_STATIC_JAVA_LIBRARIES** : 依赖的静态的jar包  
  
**12. LOCAL_STATIC_LIBRARY** : 指定模块使用的静态库  
  
**13. LOCAL_FORCE_STATIC_EXECUTABLE** : 如果模块依赖的动态库存在对应的静态库， 则会链接静态库  
  
**14. LOCAL_C_INCLUDES** : 额外的C/C++头文件路径  
  
**15. LOCAL_CC** : 指定c编译器，不使用默认的  
  
**16. LOCAL_CXX** : 指定C++编译器，不使用默认的  
  
**17. LOCAL_CFLAGS** : 为C编译器传递额外的参数(如宏定义)如  
  
	LOCAL_CFLAGS += -DLIBUTILS_NATIVE=1
    
**18. LOCAL_CPPFLAGS** : 传递额外的参数给C++编译器，如：  
	
    LOCAL_CPPFLAGS += -ffriend-injection -DLIBUTILS_NATIVE
    
**19. LOCAL_LDLIBS** : 如
  
	LOCAL_LDLIBS += -lm –lz –lc -lcutils –lutils –llog …
  
**20. LOCAL_LDFLAGS** : 传递给链接器额外的参数，比如想传递而外的库和库路径给ld，或者传递给ld linker的一些链接参数  
  
**21. LOCAL_CPP_EXTENSION** : 如果你的C++文件不是以cpp为文件后缀，你可以通过LOCAL_CPP_EXTENSION指定C++文件后缀名     
 
**22. LOCAL_CERTIFICATE** : 用于指定签名时使用的KEY，如果不指定，默认使用testkey，LOCAL_CERTIFICATE可设置的值如下：
	
+ platform
+ shared
+ media
+ testkey
  
即使用 “media.pk8 media.x509.pem” "platform.pk8 platform.x509.pem" "shared.pk8 shared.x509.pem" “testkey.pk8 testkey.x509.pem”
  
正好对应 AndroidManifest.xml文件中的  
  
	android:sharedUserId="android.uid.system"
    android:sharedUserId="android.uid.shared"
    android:sharedUserId="android.media"
    
    
**23. LOCAL_PREBUILT_JNI_LIBS** : 指定模块依赖的jni库  
  
**24. LOCAL_MULTILIB** : 在64位的android上， 如果模块需要使用32位的库， 则需要使用该变量来指定  
	
    LOCAL_MULTILIB:=32  
  
**25. LOCAL_REQUIRED_MODULES** : 指定该模块所依赖的其它模块， 当该模块被安装时， 他所依赖的模块都会被安装

**26. LOCAL_OVERRIDE_PACAGES** : 此变量可以使其他的模块不加入编译， 即覆盖其它的模块， 例如， 有两个闹钟app: AlarmClock 和 DeskClock， 在DeskClock中指定LOCAL_OVERRIDE_PACAGES := AlarmClock, 则可阻止AlarmClock编译

###3. 预定义android.mk文件  
  
例如:

	include $(CLEAR_VARS)  

实际上是 include build/core/clear_vars.mk， 该Android.mk文件中会清除LOCAL_xxx 变量，但不会清除LOCAL_PATH 变量， 因此， 通过该文件可以看到编译系统中预定义了多少种 LOCAL_XXX 变量    
  
这些预定义的Android.mk文件定义了大量的规则， 常用的几个如下：  
  
+ include $(BUILD_EXECUTABLE)  : build/core/executable.mk
+ include $(BUILD_PREBUILT)  : build/core/prebuilt.mk
+ include $(BUILD_PACKAGE) : build/core/package.mk
+ include $(BUILD_STATIC_LIBRARY) : build/core/static_library.mk
+ include $(BUILD_SHARED_LIBRARY) : build/core/shared_library.mk
+ include $(BUILD_JAVA_LIBRARY) : build/core/java_library.mk
+ include $(BUILD_PREBUILT) : build/core/prebuilt.mk  
+ include $(BUILD_MULTI_PREBUILT) : build/core/multi_prebuilt.mk  
+ include $(BUILD_PHONY_PACKAGE) ： build/core/config.mk  

在编译模块时， 只需要根据需要 include 对应的规则文件即可  
  
###4. 预定义的函数  
  
在 build/core/definitions.mk中定义大量的Makefile 函数， 可以在Android.mk中调用  
 
+ print-vars
+ my-dir
+ all-makefiles-under
+ first-makefiles-under
+ all-subdir-makefiles
+ all-named-subdir-makefiles
+ all-java-files-under
+ all-c-files-under
+ all-subdir-c-files
+ find-subdir-files
+ find-subdir-subdir-files
+ find-parent-file
+ add-dependency
  
###5. Android.mk 模板  
  
####5.1 编译可执行文件的模板  
  
	LOCAL_PATH := $(call my-dir)
    
	include $(CLEAR_VARS)
    
	LOCAL_SRC_FILES:= main.c
	LOCAL_MODULE:= test
    
	include $(BUILD_EXECUTABLE)
  
####5.2 编译动态库模板  
  
	LOCAL_PATH := $(call my-dir)
    
	include $(CLEAR_VARS)
    
	LOCAL_SRC_FILES:= /
               helloworld.c
	LOCAL_MODULE:= libtest_shared
	TARGET_PRELINK_MODULES := fals
    
	include $(BUILD_SHARED_LIBRARY)
    
####5.3 编译静态库模板  
  
	LOCAL_PATH := $(call my-dir)
	include $(CLEAR_VARS)
	LOCAL_SRC_FILES:= /
               helloworld.c
	LOCAL_MODULE:= libtest_static
  
	include $(BUILD_STATIC_LIBRARY)
    
####5.4 编译apk模板  
	
	LOCAL_PATH := $(call my-dir)
    
	include $(CLEAR_VARS)
  
	LOCAL_SRC_FILES := $(call all-subdir-java-files)
	LOCAL_PACKAGE_NAME := LocalPackage
  	
	include $(BUILD_PACKAGE)

####5.5 拷贝预编译文件模板  
  
	LOCAL_PATH := $(call my-dir)

	include $(CLEAR_VARS)

	LOCAL_MODULE := cfg.ini
	LOCAL_MODULE_TAGS := eng
	LOCAL_MODULE_CLASS := ETC 
	LOCAL_SRC_FILES := cfg.ini
    
	include $(BUILD_PREBUILT)  
    
####5.6 预编译带jni lib的apk  
  
假设在64位的android上， 需要预置一个test.apk到img中，且需要给apk打sign， apk依赖一个32位的jin lib "libtest.so" 
  
	LOCAL_PATH := $(call my-dir)

	include $(CLEAR_VARS)

	LOCAL_MODULE := test.apk
  	LOCAL_SRC_FILES := test.apk
  	LOCAL_PREBUILT_JNI_LIBS := libtest.so
  	LOCAL_MULTILIB := 32

	include $(BUILD_PREBUILT)


预编译的apk需要的jni lib需要拷贝到 /system/lib 中
