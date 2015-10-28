---
layout: post
title: "SELinux 安全上下文"
description:
category: android
tags: [android, linux]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. SELinux之安全上下文    
  
SElinux建立在对象的安全上下文的基础上的， 安全上下文用于描述资源， 由 user:role:type:sensitivity 组成， 例如  
  
	# ls -Z /init.rc  
	-rwxr-x--- root     root     u:object_r:rootfs:s0 init.rc 
    
	# ps -Z  
	LABEL                          USER     PID   PPID  NAME  
	u:r:init:s0                    root      1     0     /init
    
表明文件/init.rc的SELinux用户、SELinux角色、类型和安全级别分别为u、object_r、rootfs和s0  
进程init的安全上下文为“u:r:init:s0”，这表明进程init的SELinux用户、SELinux角色、类型和安全级别分别为u、r、init和s0
  
需要注意的是， 对象可以划分为主体(进程)和客体(资源)，**对于客体， role是没有意义的**，但是为了保持安全上下文标签的完整性， SELinux硬编码了一个特殊的role用于客体,所以我们看到的客体的role都是"object_r", 并且不能在外部再声明名为“object”的role  
  
在基于类型强制的安全上下文中， 类型才是最重要的， android中默认只有一个user一个role， 默认安全等级为s0:  
  
	# cat external/sepolicy/users
	user u roles { r } level s0 range s0 - mls_systemhigh
    
	# cat external/sepolicy/roles
	role r;  
    
因此， 在android中我们重点关注type  
  
###2. SELinux的访问规则  
  
在 SELinux 中,所有访问都必须明确授权,SELinux 默认不允许任何访问， 定义访问规则的语法是：  
  
	[ allow | dontaudit | auditallow | neverallow] sourceType targetType ： objectClass permSet  
    
+ sourceType : 即规则的源类型
+ targetType : 即规则的目标类型
+ objectClass ： 客体的资源类别
+ opSet ： 客体类别的操作许可

而规则的动作又可取值如下：

+ allow ： 允许主体对客体执行允许的操作
+ dontaudit ： 不记录违反规则的决策信息,且违反规则不影响运行
+ auditallow ： 允许操作并记录访问决策信息
+ neverallow ： 不允许主体对客体执行指定的操作
  
如果不显式allow， 那么访问是被禁止的， 因此neverallow事实上是用于冲突检查： 如果对某一访问同时指定neverallow和allow， 那么编译时会报错  
  
例如：  
	
	allow init fuse:dir search
  
基于类型的安全上下文中， 类型才是最重要的，尤其是在android中， 安全上下文标识符的组成中， 只有type会变化， 因此， 在SELinux的规则语句中， 使用安全上下文的位置， 只是标出安全上下文的type    
  
###3. SELinux的主体和客体  
  
关于主体和客体，很多地方说： “主体是进程， 客体是资源”， 但是一说法常常令人感到困惑  
  
事实上， 主体和客体只不过是安全上下文出现在SELinux规则语句中的位置不同， 那么那些能扮演主体， 那些能够扮演客体呢? 前面说过， 安全上下文用于描述资源，我个人将其分为两类：  
  
+ 进程安全上下文 ： 描述了进程资源的安全上下文, 是的， 进程也是资源，SELinux定义的资源类别为process, 还有相关的操作许可  
+ 资源安全上下文 ： 除了进程安全上下文之外的其它安全上下文
  
那么他们可以扮演哪些角色呢:  
  
+ 主体 ： 进程安全上下文  
+ 客体 ： 进程安全上下文 和 资源安全上下文, 即所有的安全上下文
  
后面会频繁提及 “进程安全上下文” 和 “资源安全上下文” 这两个名词  
  
###4. SELinux客体类别及操作许可  
  
SELinux的目的是限制主体对客体的访问， 以一条规则为例
  
	allow init fuse:dir search
    
这一条规则定义的是： 允许安全上下文类型为init的主体对安全上下文类型为fuse的客体中类别为dir的资源进行search操作       
客体类别即是对系统资源的类别划分, 下文所说的客体类别， 资源类别以及 object class 所指相同  
  
不同种类的资源上许可的操作方式就不一样, linux内核支持多少种资源类别, 根据版本的不同可能有所差异， 具体情况可以参照内核源码中的selinux部分 “security/selinux/include/classmap.h”
	
	struct security_class_mapping secclass_map[] = { 
		......
    		{ "process",
			{ "fork", "transition", "sigchld", "sigkill",
        			"sigstop", "signull", "signal", "ptrace", "getsched", "setsched",
        			"getsession", "getpgid", "setpgid", "getcap", "setcap", "share",
        			"getattr", "setexec", "setfscreate", "noatsecure", "siginh",
        			"setrlimit", "rlimitinh", "dyntransition", "setcurrent",
        			"execmem", "execstack", "execheap", "setkeycreate",
        			"setsockcreate", NULL } },

			......

		{ "binder", { "impersonate", "call", "set_context_mgr", "transfer", NULL } },
		{ NULL }
  	};
    
可以看到，进程资源"process"支持的操作有“fork”等等， “binder”资源支持的操作有“call”等等  
  
上面提到的只是内核所支持的所有的资源类别和对应的操作， 在实际使用时，还需要在策略文件中声明会使用到的资源类型和对应的操作集合， 声明资源类别和操作可以只是secclass_map的子集  
  
####4.1 申明资源类别  
  
声明资源类别通常放到security_classes文件中， 使用关键字class， 语法为  
  
	class name  
    
android中资源类别的声明在 “external/sepolicy/security_classes” 文件中, 截取部分如下    
	
	class process
	class filesystem
	class file
	class dir
	class fd
  
**需要注意的是， 大部分的资源需要通过系统调用才能访问， 因此在内核中能够完成求权限许可的检查， 但是有一些资源完全是在用户空间， 比如 android 的 property， 因此， 在这里面可以看到security_classes定义的资源类别比classmap.h中多**， android中，用户空间的资源是由Security Server来保护的    
  
在编译时， linux会根据security_class生成对应的头文件， 放置在kernel out 目录下的 security/selinux/flask.h，截取一段  

	#define SECCLASS_FILE                                     6
	#define SECCLASS_DIR                                      7
	#define SECCLASS_FD                                       8
     
####4.2 申明资资源的可用操作  
  
同样的， 申明资源可用的操作语法如下  
  
	class className {perm1, perm2, .... permn }
    
例如  
  
	class property_service { set }
  
还可以定义一些公共的操作集合， 然后直接继承， 减少文字量， 例如：  
  
	common ipc 
	{
		create
		destroy
		getattr
		setattr
		read
		write
		associate
		unix_read
		unix_write
	}
    
	class sem inherits ipc

	class msgq inherits ipc
	{
		enqueue
	}

common部分定义了 信号量(sem)和消息队列(msgq)都具有的操作集合， 然后msgq还有一个独有的操作"enqueue"  
  
**注意， 声明可用操作集合时， 只能继承common定义的操作集合， 不能继承别的资源类别(class)**， 但是common和class可以同名  
  
android中为各种资源类别声明的可用操作在 external/sepolicy/access_vectors， 截取如下  
  
	class lnk_file
	inherits file
	{
		open
		audit_access
		execmod
	}
   
在编译时， linux会根据access_vectors生成对应的头文件， 放置在kernel out 目录下的 security/selinux/av_permissions.h, 截取一段如下    
  
	......
	#define FILE__IOCTL                               0x00000001UL
	#define FILE__READ                                0x00000002UL
	#define FILE__WRITE                               0x00000004UL
	......
  
###5. SELinux中的type和attribute  
  
SELinux并未预置任何类型，可以使用type关键字定义一个类型， 例如：  
  
	type init_shell；
  
另外， 我们还会经常看到attribute语句，type和attribute经常让人混淆，事实上 **attribute不过是语法糖** ，用于减轻te文件的编写负担  
  
可以使用attribute关键字定义一组规则的集合， 然后在定义type的时候继承该attribute， 即可包含attributre中的规则，在编译生成策略文件时会将attribute拓展为其包含的type  
  
例如 定义一个属性sysfs_type， 有一条属性  
  
	attribute file_type;
	allow file_type tmpfs:filesystem associate;
	allow file_type rootfs:filesystem associate;
    
那么如果定义一个type， 继承了sysfs_type属性  

	type system_file, file_type;
	type system_data_file, file_type;
    
那么这一条在编译期间， 会被替换为  
  
	allow system_file tmpfs:filesystem associate;
	allow system_file rootfs:filesystem associate;
    
	allow system_data_file tmpfs:filesystem associate;
	allow system_data_file rootfs:filesystem associate;
    
一个type可以关联(继承)多个属性， 用“，”隔开， 例如:  
  
	type sysfs_bluetooth_writable, fs_type, sysfs_type, mlstrustedobject;
  
如果把类型定义和关联属性分开进行， 可以使用typeattribute关键字
  	
	type sysfs_writable；
	typeattribute sysfs_writable sysfs_type; 

因此， type和attribute都是一组规则的集合    
需要注意的时， type和attribute使用同一命名空间， 因此，不能有重名  
  
###6.SELinux的进程安全上下文  
  
很多地方都说“主体(进程安全上下文)的类型是domain”, 而且， 很多进程安全上下文的定义时， 基本上都会关联到"domain", 这是为什么呢？  
  
我们先来看一看domain到底是什么呢， 在 external/sepolicy/attribute 中可以找到定义：  
  
	# All types used for processes.
	attribute domain;
    
前面已经讲过， attribute是语法糖衣， 是一堆规则的集合， 看一看 attribute domain 的定义， 在 external/sepolicy/domain.te 中  这里仅贴出一段  
  	
	allow domain init:process sigchld;

	# Intra-domain accesses.
	allow domain self:process {
		fork
		sigchld
		sigkill
		sigstop
		signull
		signal
		getsched
		setsched
		getsession
		getpgid
		setpgid
		getcap
		setcap
		getattr
		setrlimit
	};
    
	# Connect to adbd and use a socket transferred from it.
	# This is used for e.g. adb backup/restore.
	allow domain adbd:unix_stream_socket connectto;
	allow domain adbd:fd use;
	allow domain adbd:unix_stream_socket { getattr getopt read write shutdown };
  
里面基本上就是各种资源许可，满足一个进程最基本的需求，否则， 如果自行定义一个type(进程安全上下文)且不继承doamin属性， 当从其它的进程安全上下文转移到这个上下文中之后， 就会存步难行， 连基本的文件描述符使用，打印log都无法做到， 因此每一个进程安全上下文都需要继承domain属性(除非自己手动添加所需的所有属性)，并且android里面也限定了进程安全下文必须继承doamin属性  
  
	$ cat external/sepolicy/roles
	role r;  
	role r types domain;
    
前面说过， android中只定义了一个role r， 而最后一行则限定了只有继承了domain属性的type才能和role r在一起组合成安全上下文， 由于r只会出现那在主体中， 也就是说进程安全上下文的type必须继承domain属性， 而客体使用硬编码的role object_r，因此不受此限制，所以， 我们可以看到，资源安全上下文都没有继承domain属性  
  
doamin中只定义了基本的资源访问许可， 因此除了继承doamin属性，定义的type基本上都会在声明所需的许可， android中还定义了其它的一些属性  
  
+ unconfineddomain ： 无限权限， 供deamon和私有组件使用;
+ appdomain : 供zygote fork出的app使用;
+ netdomain ： 网络相关的许可;
+ bluetoothdomain ： 使用蓝牙相关的许可;
+ binderservicedomain :  使用binder相关的许可;
+ mlstrustedsubject ： 允许进程绕过mls检查  

在自定义进程安全上下文时，可以根据需要继承这些domain属性  
  
因此, 将不同的主体(进程安全上下文)称作不同的domain，进程安全上下文的转移称作domain的转移也是可以理解  

解释“主体”和”客体“的部分说道过， 进程作为一种资源， 进程安全上下问可以作为客体出现 例如：    
  
	allow zygote system_server:process { getpgid setpgid };  
    
这一条， 许可了安全上下文为“zygote”的进程 set/get 安全上下文为 “system_server”的进程的的pid  
    
每一个可执行程序在运行时， 都有一个进程安全上下文， 这两者是怎么关联起来的呢， 这就涉及到了进程安全上下文(domain)的继承和转移  
  
####6.1 进程安全上下文的继承  
  
int进程是系统中的第一个进程，所有的进程都是从init或者其子进程派生  
  
init进程在完成selinux的初始化后， 将自己的安全上下文设置为“u:r:init:s0”， 见init.rc：  
  
	setcon u:r:init:s0
        
每一个子进程在被fork出来后， 继承了父进程的安全上下文，除非自己声明进行直到进行安全上下文的转移  

####6.2 进程安全上下文的静态转移  
  
若可执行程式共享一个进程安全上下文， 其中某一个可执行程序需要某一资源许可时， 就需要要给这一进程安全上下文添加该资源许可，那么该进程安全上下文中的其它的可执行程序也拥有了这一资源的许可， 这样一来， 就达不到安全隔离的目的， 为此， 我们可以为这一有特殊资源需求的可执行程序定义一个新的进程安全上下文(doamin)  
  
**注意，进程安全上下文的转移只能通过exec()系统调用**, 即父进程在fork()后， 调用exec()执行一个可执行文件的过程中就发生了进程安全上下文(domain)的转移  
  
那么，一个可执行程序怎么和一个进程安全上下文联系起来的呢, 在SELinux中概括起来就是“允许某个资源安全上下文的可执行程序在执行时， 可以从一个指定的进程安全上下文转移到一个指定的新的进程安全上下文”， 下面我们举例说明：  
  
假设我们在init.rc中定义了一个service， 其可执行文件为/system/bin/test.sh，为其资源安全上下文的type为test_exec， 我们希望为其运行时的进程转移到类型为test_domain的进程安全上下文(domain)， 需要修改或者添加下列文件：  
  
	file : file_contexts
    	+ /system/bin/test.sh u:object_r:test_exec:s0
  
	file test.te
	+ type test_domain, domain;
	+ type test_exec, exec_type, file_type;
	+ allow init_shell test_exec : file {getattr execute}  
	+ allow test_domain test_exec : file {getattr execute}  
	+ allow test_domain test_exec : file entrypoint
	+ allow init_shell test_doamin  : process transition;
	+ type_transition init_shell test_exec : process test_domain;
    
file_contexts文件中指定了可执行文件的安全上下文, 然后， 添加一个test.te文件， 下面解释一下test.te文件内容  
  
1. 第一行， 定义一个进程安全上下文(domain) test_doamin  
2. 第三行， service在被fork出来后， 继承的安全上下文是init_shell，其需要有test.sh文件的执行许可
3. 第四行， exec()中， 在转移到新的进程安全上下文test_domain时， 其还需要有test.sh的执行许可 
4. 第5行， 允许test.sh文件执行时，进程安全上下文转移到test_domain  
5. 第6行， 允许从进程安全上下文init_shell转移到test_domain  
6. 第6行, 以上规则只是允许了两个进程安全上下文的转换， 但是只有这些规则的话，是不会进行转换的，因为， 父进程可以执行多种type的可执行文件，分别转移到不同的进程安全上下文， 而一种type的可执行文件， 也可以被多种进程安全上下文的父进程exec， 分别转移到不同的进程安全上下文， 因此， 我们还需要指定在何种条件下进行哪一种转移， 这一行定义了在进程安全上下文init_shell中运行运行资源安全上下文为 test_exec 的可执行文件时， 默认会尝试将进程安全上下文转移到test_doamin  
   
可以看到， 进程安全上下文的转移需要指定许多的规则  
  
####6.3 domain_auto_trans  
  
有一个宏用于定义安全上下文的转移， android上可以在external/sepolicy/te_macros 中找到  
  
	define(`domain_trans', `
		# Old domain may exec the file and transition to the new domain.
		allow $1 $2:file { getattr open read execute };
		allow $1 $3:process transition;
		# New domain is entered by executing the file.
		allow $3 $2:file { entrypoint open read execute getattr };
		# New domain can send SIGCHLD to its caller.
		allow $3 $1:process sigchld;
		# Enable AT_SECURE, i.e. libc secure mode.
		dontaudit $1 $3:process noatsecure;
		# XXX dontaudit candidate but requires further study.
		allow $1 $3:process { siginh rlimitinh };
	')
    
    	define(`domain_auto_trans', `
		# Allow the necessary permissions.
		domain_trans($1,$2,$3)
		# Make the transition occur by default.
		type_transition $1 $2:process $3;
	')
    
依次以 init_shell test_exec test_domain 代入 domain_auto_trans， 可得  
  
	allow init_shell test_exec:file { getattr open read execute };
	allow init_shell test_exec:process transition;
	allow test_domain test_exec:file { entrypoint open read execute getattr };
	allow test_domain init_shell:process sigchld;
	dontaudit init_shell test_domain:process noatsecure;
	allow init_shell test_domain:process { siginh rlimitinh };
	type_transition init_shell test_exec:process test_domain;
    
基本上就是声明了上节所述的进程安全上下文转移的许可  
  
####6.4 进程安全上下文的动态转移  
  
上面讲过进程安全上下文的静态转移， 设为好转移规则后， 父进程fork， exec执行文件后，转移到新的进程安全上下文  
  
在android中要面临另一种情况， android的应用程序都是由zygote进程fork出来， 但是不会像传统Linux的应用程序进程一样，会通过exec系统调用将对应的可执行文件加载起来执行, 因此， 无法使用进程安全上下文静态转移的方式  
  
android中的进程是由ActivityManagerService请求Zygote进程创建的，ActivityManagerService在请求Zygote进程创建应用程序进程的时候，会传递很多参数， 其中包括一个seinfo用于设置应用程序的安全上下文， 对于应用程序， 会调用setSELinuxContext()来设置其进程安全上下文(是指上是调用libselinux的selinux_android_setcontext())  
  
###7. SELINUX的资源安全上下文  
  
在linux系统中， 有许多的资源，接触最多的就是文件了， 那么他们的安全上下文是如何确定的呢？  下面以android系统为例进行说明 
 
####7.1 资源安全上下文的转移  
  
首先， 一个进程安全上下文需要权限才能修改资源的安全上下文， 大多数客体类别在策略中都使用 relabelfrom 和 relabelto 改变与文件有关的客体的类型,relabelfrom 许可控制客体的开始类型,relabelto 许可控制结果类型,一个域必须要同时具有这两个许可才能成功地重新标记一个客体的安全上下文， 例如  
  
	allow user_t user_home_t : file { relabelfrom };
	allow user_t test_t : file { relabelto };
  
上述规则允许进程安全上下文能够将安全上下文为“ user_home_t”的文件的安全上下文转换为“test_t”  
  
改变客体的安全上下文需要用到三个库程序调用:setfilecon(3),lsetfilecon(3)和 fsetfilecon(3)。只需要适当的 relabelfrom 和relabelto 许可就可以明确地改变一个客体的安全上下文  
  
另外， 还可以使用“type_transition”规则定义客体的安全上下文的转移, 例如
	
	type_transition system_server wpa_socket:sock_file system_wpa_socket
    
许可 进程安全上下文为“system_server”在安全上下文为“wpa_socket”的目录创建socket文件时， 文件的安全上下文为“system_wpa_socket”  
  
####7.2 文件的安全上下文  
  
linux中大部分资源都可以使用文件来描述， 那么一个文件系统以及其中文件的安全上下文是如何关联(这一过程被称为 lableing)到一起的呢？   
  
**创建文件时， 若无特殊指定， 会继承其父目录的安全上下文**  
  
SELinux中所有的文件系统的初始安全上下文是在挂载时标记的的，根据文件系统的不同，其labling的规则有所差异, SElinux支持4中文件相关的安全上下文标记类型  
  
#####7.2.1 fs_use_xattr(基于扩展属性)  

适用于： 支持扩展属性的常规文件系统， 如  yaffs2、jffs2、ext2、ext3、ext4、xfs、btrfs等
  
这一类的文件系统的安全上下文保存在文件的inode的属性中，系统可通过getattr()函数读取inode中的安全上下文信息, 这类文件系统使用同一个文件系统安全上下文"u:object_r:labeledfs:s0"来统一标记, ， 例如， 在android上， external/sepolicy/fs_use 中:  
  
	# Label inodes via getxattr.
	fs_use_xattr yaffs2 u:object_r:labeledfs:s0;
	fs_use_xattr jffs2 u:object_r:labeledfs:s0;
	fs_use_xattr ext2 u:object_r:labeledfs:s0;
	fs_use_xattr ext3 u:object_r:labeledfs:s0;
	fs_use_xattr ext4 u:object_r:labeledfs:s0;
	fs_use_xattr xfs u:object_r:labeledfs:s0;
	fs_use_xattr btrfs u:object_r:labeledfs:s0;
	fs_use_xattr f2fs u:object_r:labeledfs:s0;
    
这一类文件系统中的文件，在创建时就标记好了安全上下文并存储在文件系统中，例如， android 中的 system.img 中的文件在生成的时候就已经标记好了安全上下文   
  
#####7.2.2 fs_use_task(基于任务)  
  
适用于 ： 不真实存储用户数据但支持某种类型的内核资源的伪文件系统上(这类文件系统中的文件与用户空间可见的文件没有任何关联， 例如匿名管道， socket， 都使用文件描述符， 带是没有对应的可见文件)，目前只有sockfs，pipefs这两种  
  
针对这一类文件系统， 使用 fs_use_task 规则声明新的与文件有关的客体继承创建它们的进程的安全上下文， 这一类文件系统不支持来自程序请求的安全上下文(意味着其安全上下文无法改变)， 在android中， external/sepolicy/fs_use 文件中有定义  
  
	# Label inodes from task label.
	fs_use_task pipefs u:object_r:pipefs:s0;
	fs_use_task sockfs u:object_r:sockfs:s0;
  
这两个文件系统的文件系统的安全上下文分别是“u:object_r:pipefs:s0”和“u:object_r:sockfs:s0”，其中创建的文件的安全上下文继承自创建他们的进程  
  
#####7.2.3 fs_use_trans(基于转换的文件系统安全上下文)  
  
适用于 ： 虚拟文件系统，现有devpfs、tmpfs、devtmpfs、shm、mqueue使用  
  
这样的文件系统上创建的文件都需要有一套相关的type_transition规则来完成。如果没有找到相应的type_transition规则，文件就会使用文件系统的安全上下文,android上 external/sepolicy/fs_use 文件中有定义  
  
	fs_use_trans devpts u:object_r:devpts:s0;
	fs_use_trans tmpfs u:object_r:tmpfs:s0;
	fs_use_trans devtmpfs u:object_r:device:s0;
	fs_use_trans shm u:object_r:shm:s0;
	fs_use_trans mqueue u:object_r:mqueue:s0;
    
如果在另外定义了一条安全上下文转移规则， 例如：  
  
	type_transition sysadm_t devpts : chr_file sysadm_devpts_t:s0
    
这样， 进程安全上下文为 sysadm_t 的执行程序在 devpts 文件系统中创建的字符设备文件安全上下文为 sysadm_devpts_t,但是若没有定义这一转换规则， 则文件的安全上下文就是“devpts” 
    
#####7.2.4. genfscon 
  
适用于 ： 不支持扩展属性的文件系统(vfat， ntfs等)，虚拟的文件系统(现有rootfs、proc、selinuxfs、cgroup、sysfs、inotifyfs、vfat、debugfs、fuse 使用)  
    
android上 external/sepolicy/fs_use 文件中有定义  
  
	# Label inodes with the fs label.
	genfscon rootfs / u:object_r:rootfs:s0
	# proc labeling can be further refined (longest matching prefix).
	genfscon proc / u:object_r:proc:s0
	genfscon proc /net u:object_r:proc_net:s0
	......
	genfscon proc /sys/kernel/usermodehelper u:object_r:usermodehelper:s0
	genfscon proc /sys/net u:object_r:proc_net:s0
	genfscon proc /sys/vm/mmap_min_addr u:object_r:proc_security:s0
	# selinuxfs booleans can be individually labeled.
	genfscon selinuxfs / u:object_r:selinuxfs:s0
	genfscon cgroup / u:object_r:cgroup:s0
	# sysfs labels can be set by userspace.
	genfscon sysfs / u:object_r:sysfs:s0
	genfscon inotifyfs / u:object_r:inotify:s0
	genfscon vfat / u:object_r:vfat:s0
	genfscon tntfs / u:object_r:sdcard_external:s0
	genfscon texfat / u:object_r:sdcard_external:s0
	genfscon debugfs / u:object_r:debugfs:s0
	genfscon fuse / u:object_r:fuse:s0
	genfscon pstore / u:object_r:pstorefs:s0
	genfscon functionfs / u:object_r:functionfs:s0
	genfscon usbfs / u:object_r:usbfs:s0  
      
genfscon指定的安全上下文作为文件系统的安全上下文， 以及里面文件的默认安全上下文  
   
事实上， 对于虚拟文件系统， 上述规则只是定义了文件系统的初始安全上下文， 如何自己指定文件或者目录的安全上下文呢？ 现在， 需要用到file_contexts 文件， 在android上就是 /file_contex文件(由external/sepolicy/file_contex 和 device目录下的file_contex文件生成):  
  
	/           u:object_r:rootfs:s0
	/init           u:object_r:rootfs:s0
	/sbin(/.*)?     u:object_r:rootfs:s0
    
	/dev(/.*)?      u:object_r:device:s0
	/dev/console        u:object_r:console_device:s0
    
	/system(/.*)?       u:object_r:system_file:s0
	/system/bin/sh      --  u:object_r:shell_exec:s0
    
	/data(/.*)?     u:object_r:system_data_file:s0
  
file_contexts文件会在以下几个地方被使用到：  
  
1. 生成 system.img 的时候， 为里面的文件设置安全上下文  
2. init进程中为目录和文件设置文件安全上下文 （例如 restorecon(/dev), restorecon persist ）    
  
file_contex使用正则表达式的语法， 被匹配到的目录或文件都将被标记上相应的安全上下文，这个过程发生在文件系统安全上下文设置完成之后执行, 在android中， 我们自己添加的文件可以在file_contex中设定安全上下文    
     
####7.3 属性的安全上下文  
  
在 external/sepolicy/property_contexts 和  device/xxx/sepolicy/property_contexts 中定义了property的安全上下文， 最终被汇总到 $OUT/root/property_contexts文件中， 截取如下：  
  
	net.                    u:object_r:system_prop:s0
	dev.                    u:object_r:system_prop:s0
	runtime.                u:object_r:system_prop:s0
	hw.                     u:object_r:system_prop:s0
	sys.                    u:object_r:system_prop:s0
	sys.powerctl            u:object_r:powerctl_prop:s0
	service.                u:object_r:system_prop:s0
	wlan.                   u:object_r:system_prop:s0
	dhcp.                   u:object_r:dhcp_prop:s0
  
里面定义了以某种前缀开头的属性的安全上下文  
  
####7.4 android为资源安全上下文定义的属性  
  
1. dev_type ： 与设备文件相关  
2. fs_type: ： 文件系统相关  
3. file_type ： 文件系统相关
4. exec_type ： 可执行文件相关  
5. data_file_type ： /data 目录下文件相关
6. sysfs_type ： sysfs 文件系统下所有文件相关
7. node_type ： 
8. netif_type : 网络接口相关  
9. port_type ： 网络端口相关
10. mlstrustedsubject ： 继承这一属性的type，在作为主体时， 能绕过MLS检查相关
11. mlstrustedobject ： 继承这一属性的type，在作为客体时， 能绕过MLS检查相关
  
我们在自己定义新的资源安全上下文的时候， 根据需要， 继承这些属性, 例如， 要在 /syetm/bin下添加有一个可执行文件test.sh, 为其定义一个安全上下文type test_type  
  
	type test_type, exec_type， file_type  
      
###8. android中定义的宏  
  
external/sepolicy/te_macros   
  
+ domain_trans(olddomain, type, newdomain) : 定义从olddomain到newdomain的转移许可
+ domain_auto_trans(olddomain, type, newdomain) ： 用于定义从olddomain到newdomain的转移许可并且自动转移    
+ file_type_trans(domain, dir_type, file_type) ： 定义domain在安全上下文为type的目录创建文件时， 文件的安全上下文转移规则    
+ file_type_trans(domain, dir_type, file_type) ： 定义domain在安全上下文为type的目录创建文件时， 文件的安全上下文转移规则并且自动转移    
+ r_dir_file(domain, type) ： 允许domain访问安全上下文为type的目录下的文件和符号连接文件  
+ unconfined_domain(domain) ： 使指定的doamin获得更多的许可(这一宏只是让指定的进程安全上下文继承mlstrustedsubject和unconfineddomain属性)  
+ tmpfs_domain(domain) ： 定义domain访问tmpfs / shmem / ashmem 的许可  
+ init_daemon_domain(domain) : 定义从init doamin到指定的domain的转移(适用于init.rc中定义的service)    
+ app_domain(domain) ： 为domain定义所有android app所需的基本许可(实际上是继承appoamin属性)   
+ net_domain(domain) ： 为doamin定义访问网络所需的基本许可(实际上是继承netdoamin属性)  
+ bluetooth_domain(domain) : 为doamin定义访问蓝牙所需的基本许可(实际上是继承blutoothdoamin属性)  
+ unix_socket_connect(clientdomain, socket, serverdomain) ： 定义了clientdomain通过给定安全上下文的本地socket连接serverdomain所需的许可  
+ unix_socket_send(clientdomain, socket, serverdomain) ： 定义了clientdomain通过给定安全上下文的本地socket向serverdomain发送(write, sendto)消息的许可  
+ binder_use(domain) : 允许domain使用binder通信  
+ binder_call(clientdomain, serverdomain) ： 允许clientdomain通过binder调用serverdomain  
+ binder_service(domain) ： 定义domain作为binder service的相关许可  
+ wakelock_use(domain) ： 定义doamin使用wakelock所需的许可  
+ selinux_check_access(domain) ： 定义domain访问selinuxfs来检查权限的许可  
+ selinux_check_context(domain) ： 定义domain访问selinuxfs来检查安全上下文的许可  
+ selinux_setenforce(domain) ： 定义domain enable/disable selinux的许可  
+ selinux_setbool(domain) : 定义了doamin设置selinux布尔值的许可  
+ security_access_policy(domain) ： 定义doamin访问所有policy文件的只读许可  
+ selinux_manage_policy(domain) ： 定义doamin管理所有policy文件并且reload policy文件的许可  
+ mmac_manage_policy(domain) ： 定义doamin管理所有mmac的policy文件并且reload policy文件的许可  
+ access_kmsg(domain) ： 定义doamin能够访问kernel log及执行klogctl系统调用的许可  
+ write_klog(domain) ： 定义doamin写kernel log(使用klog_write)的许可  
+ create_pty(domain) ： 定义doamin创建并且使用pty的许可  
+ non_system_app_set ： 代表非系统app的权限许可的集合(实际上是从appdomain中去掉system_app的权限许可)   
+ permissive_or_unconfined ： 
+ write_logd(domain)： 定义domain打印android log的许可  
+ read_logd(domain) ： 定义domain通过socket从android logd读取log的许可  
+ control_logd(domain) ： 定义domain通过socket来控制logd  
+ use_keystore(domain) ： 定义domain访问keystore的权限  
  
在android中， 根据build的image的不同， 还定义了一些宏， 用于控制selinux规则在某些条件下才生效  
  
**recovery_only()** : 定义的selinux规则在recovery img 中才生效， 例如  
  
	recovery_only(`
		domain_trans(init, rootfs, recovery)
	')

即在build recovery img中才包含“domain_trans(init, rootfs, recovery)”规则  
  
**userdebug_or_eng()** : 给定的selinux规则在build userdebug或者eng时才生效， 例如：  
  
	userdebug_or_eng(`
		unix_socket_send(wpa, wpa, su) 
	')
  
在build userdebug和eng img时， “unix_socket_send(wpa, wpa, su)”规则才生效
