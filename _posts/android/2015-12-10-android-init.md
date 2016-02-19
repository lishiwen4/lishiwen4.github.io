---
layout: post
title: "android init 启动流程"
description:
category: android
tags: [android, linux]
mathjax: 
chart:
comments: false
---

###1. android init

android上， init的源码位于 “system/core/init” 目录， 可以编译出 init， ueventd， watchdogd， 本章只关注init

###2. init 启动流程

init 进程为用户空间的第一个进程， 其入口为 main(), 在android上， 启动参数为 “--second-stage”

    //清除umask
    umask(0);
    
    //添加环境变量
    add_environment("PATH", _PATH_DEFPATH);
    
init 的两个阶段:

+ first statge ： init第一次运行时， 处于SELinux的kernel domain
+ second stage ： 初始化SELinux之后， re-exec init， 切换到SELinux的 init domain

####2.1 first stage 的工作

    //挂载虚拟文件系统 /dev /dev/pts /proc /sys
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    mount("proc", "/proc", "proc", 0, NULL);
    mount("sysfs", "/sys", "sysfs", 0, NULL);
    
    //初始化init的标准流
    open_devnull_stdio();
    
    //初始化log系统和log等级， 从现在开始可以打印log了
    klog_init();
    klog_set_level(KLOG_NOTICE_LEVEL);
    
    //初始化SELinux， 包括load SELinux 的policy
    selinux_initialize(is_first_stage);
    
    //切换init的domain为file_contexts文件中指定的上下文， re-exec init, 进入 second stage
    if (is_first_stage) {
        if (restorecon("/init") == -1) {
            ERROR("restorecon failed: %s\n", strerror(errno));
            security_failure();
        }
        char* path = argv[0];
        char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
        if (execv(path, args) == -1) {
            ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
            security_failure();
        }
    }
    
####2.3 second stage

    //建立 /dev/.booting 文件， 标示kernel boot完成
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
    
    //初始化property 系统
    property_init();
    
    //处理通过 devictree 传递的kernerl启动参数
    process_kernel_dt();
    
    //处理通过 cmdline 传递的kernel启动参数
    
    //设置boot属性， 即名为“ro.boot.*”的属性
    
    //完成SELinux在second stage的初始化工作
    selinux_initialize(is_first_stage);
    
    //这些目录在SELinux初始化之前创建， 因此需要根据 file_contexts 文件的内容来重新设置上下文
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon_recursive("/sys");
    
    //
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    
    //注册SIGCHLD信号的处理例程，init.rc中定义的service都是init的子进程
     signal_handler_init();

    //读取 /default.prop 文件中记录的属性
    property_load_boot_defaults();
    
    //启动 property service
    start_property_service();
    
    //解析 init.rc 脚本
    init_parse_config_file("/init.rc");
    
    //执行action队列中的command
    action_for_each_trigger("early-init", action_add_queue_tail);
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    queue_builtin_action(keychord_init_action, "keychord_init");
    queue_builtin_action(console_init_action, "console_init");

    action_for_each_trigger("init", action_add_queue_tail);
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    
    if (property_get("ro.bootmode", bootmode) > 0 && strcmp(bootmode, "charger") == 0) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }
    
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");


    //进入循环， 处理事件
    while (true) {
        if (!waiting_for_exec) {
            execute_one_command();
            restart_processes();
        }
        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }
        if (!action_queue_empty() || cur_action) {
            timeout = 0;
        }
        bootchart_sample(&timeout);
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

后续将会详细解释 init.rc 脚本的处理， 以及init进入循环后的处理过程

###3. init.rc 脚本语法

Android init.rc 中可以分为3种section

    import xxxx.rc          //包含其他的rc文件
    
    on trigger                 //定义action
        command
        command
        ......
        
    service name path args       //定义service
        options
        options
        ......

init.rc 脚本中的语句可以细分为4种语句

1. commands ： command是init支持的的内建命令， 例如 chmod， copy等
2. actions ： action是命名的command序列， 以关键字"on" 开头， 通过不同的条件来触发
3. services ： service是由init负责启动或者重新启动的程序
4. options ： options是service的修饰符

Android init.rc 脚本的语法风格为

1. 每一个语句占据一行，所有关键字通过空格来分割
2. c语言风格的反斜杠(/)将被转义为插入一个空格
3. 如果一个关键字含有一个或多个空格，那么怎么保证关键字完整呢？可以使用双引号来确定关键字的范围
4. 用于行尾的反斜杠表示续行符
5. Actions和Services声明一个字段(section)，紧随其后的Commands和Options均属于这个字段，在第一个字段之前的Commands和Options的没有意义
6. Actions和Services有独一无二的名字，如果Actions和Services的名字有重名，那么将被视作错误

###3.1 Actions 语句

Action 是一组被命名的command序列，“on“ 是action的关键字， 表示下面的序列时Action 其格式如下， 当trigger条件触发时，init讲顺序执行所有的command 

    on <trigger>
        <command>
        <command>
        <command>

###3.2 Service

Service 是由init负责启动或者重新启动的程序， 格式如下

    service <name> <pathname> [ <argument> ]*
        <option>
        <option>
        ...
        
其中 name 为service name， pathname 为对应的可执行程序的路径， argument为可执行程序的参数

android中开可以通过如下的方法来start/stop service， 以名为test的service为例

+ 执行"start/stop  test"
+ 向ctl.start/ctl.stop 属性写入 ”int.svc.test“ 

####3.3 Option

Options是Services的修饰符

Option 的关键字在 init 源码的 keywords.h 中以类似于如下的语句定义

    KEYWORD(class,       OPTION,  0, 0)
    
所有的option 关键字如下

+ class     : 设置service的class， 相同class的service会被同时启动或者停止， 若不定义则被视为”default“class
   + 语法为 ”class <name>“
+ console   ： 
+ critical  ： 设备相关的关键服务， 若4分钟内该service重启了4次， 则设备将进入recovery模式 
+ disabled  ： 服务不会自启动， 必须显示地启动
+ group     ： 在 exce 该进程前， 先切换到指定的group
   + 语法为 ”group <groupname>“
+ ioprio
+ keycodes  ： 指定收到某些组合键后启动service
+ oneshot   ： 当service退出后， 不重启， init进程默认在service每次退出后都重启
+ onrestart ： 当service重启时， 执行指定的comamnd
   + 语法为 ”onrestart <command>“
+ seclabel  ： 在exec之前设置进程的SELinux安全上下文， 通常是rootfs中的执行程序使用
   + 语法为 ”seclabel <seclabel>“
+ setenv    ： 设置环境变量
   + 语法为”setenv <name> <value>”
+ socket    ： 在“dev/sockets/“ 下创建一个unix domain socket， 并且传递fd给启动的进程
   + 语法为 ”socket <name> <type> <perm> [ <user> [ <group> [ <seclabel> ] ] ]“
   + type 必须是 "dgram", "stream"，"seqpacket" 中的一个， user和group默认为0， seclabel是该socket的SELinux上下文
+ user ： 在执行该进程之前先切换user
   + 语法为 ”user <username>“
+ writepid    : 当service中执行fork时， 讲children的pid写入指定的文件
   + 语法为 ”writepid <file...>“
   
####3.4 commands

Commands即是在满足triger条件后，Actions中执行的内容， init 支持的command 使用类似如下的语法定义在 ”keywords.h“ 中

    KEYWORD(bootchart_init,        COMMAND, 0, do_bootchart_init)
    
其中 ”bootchart_init“ 为command的关键字， ”do_bootchart_init“ 为该command的处理例程， init支持的command如下：

+ bootchart_init
+ chmod
+ chown
+ class_reset
+ class_restart
+ class_stop
+ copy
+ domainname
+ enable
+ exec
+ export
+ hostname
+ ifup
+ insmod
+ installkey
+ load_all_props
+ load_persist_props
+ loglevel
+ mkdir
+ mount_all
+ mount
+ powerctl
+ restart
+ restorecon
+ restorecon_recursive
+ rm
+ rmdir
+ setprop
+ setrlimit
+ start
+ stop
+ swapon_all
+ symlink
+ sysclktz
+ trigger
+ verity_load_state
+ verity_update_stae
+ wait
+ write

####3.5 Trigger

触发器用来描述一个触发条件， 当条件满足时， 开始执行指定的action

android init原生支持的 trigger有如下几种

+ early-init
+ init 
+ late-init 
+ charger  

"late-init" 和 "charger" 模式互斥， "late-init" 存在于正常模式， charger存在于充电模式

+ boot ： 这是init启动后， 触发的第一个trigger
+ property:&lt;name&gt;=&lt;value&gt; ： 当名为name的属性值变化为value时， 触发
   + 可以使用”&&“进行与操作， 例如 ”on property:test.a=1 && property:test.b=1“
   
虽然init只支持这2种trigger， 但是配合init的"trigger" command, 却可以自定义trigger， 例如， 在init.rc 中有：

    on early-fs
    ......
    on fs
    ......
    on post-fs
    ......
    on post-fs-data
    ......
    on boot
    ......
    
    on late-init
        trigger early-fs
        trigger fs
        trigger post-fs
        trigger post-fs-data
        trigger early-boot
        trigger boot
        
"early-fs", "fs", "post-fs" "boot" 等， 都是在init.rc中自定义的trigger， 依靠 init 的 trigger 命令来触发

###4. init 处理 init.rc 脚本

####4.1 解析 init.rc 脚本

"init.rc" 是 init直接读取的文件， 而init.rc 中可以使用 “import” 关键字来包含另一个init.rc脚本， 类似于clang中的“#include”

在 init 中， init.rc 中的 command 使用如下结构来表示

    struct command
    {
        struct listnode clist;                  //保存一个Action中所有的command的链表
        int (*func)(int nargs, char **args);    //command的处理例程
        int line;                               //command在 *.rc文件中的行号
        const char *filename;                   //command所在的rc文件名
        int nargs;                              //command的参数数目
        char *args[1];                          //command的参数
    };
    
在 init 中， init.rc 中的trigger 使用如下的数据结构来表示

    struct trigger {
        struct listnode nlist;      //用于被添加到 struct action.triggers 链表中
        const char *name;           //trigger的名称
    };
    
在 init 中， init.rc 中的action 使用如下的数据结构来表示

    struct action {        
        struct listnode alist;      //保存init中所有action的链表
        struct listnode qlist;      //保存所有待处理的action的链表        
        struct listnode tlist;      //保存某一种trigger下的所有的action的链表， 暂时未被使用
        unsigned hash;        
        struct listnode triggers;   //该action的所有trigger， 一个action可以有多个trigger与操作
        struct listnode commands;   //该action的所有的command
        struct command *current;    //
    };
    
**init 中使用一个全局的链表 “action_list” 来保存所有的action的信息， 同时还有一个全局链表 “action_queue”， 所有要执行的action都添加到该链表中， 依次执行**

struct action， struct trigger, struct command 这3种数据结构的关系如下图

![action trigger command](/images/android/init-actions.png)

在 init 中， init.rc 中的service 使用如下的数据结构来表示

    struct service {
        void NotifyStateChange(const char* new_state);  //当service状态改变时修改 init.svc.xxx 属性
        struct listnode slist;                          //所有service形成链表
        char *name;                                     //service name
        const char *classname;                          //service class
        unsigned flags;                                 
        pid_t pid;                              
                                                        //下面这3个成员和critical option相关
        time_t time_started;                            //service的最后一次启动时间    
        time_t time_crashed;                            //在计时窗口内service的第一次crash的时间
        int nr_crashed;                                 //在计时窗口内service的crash次数， 
        
        uid_t uid;                                      
        gid_t gid;
        gid_t supp_gids[NR_SVC_SUPP_GIDS];              //rc脚本中group option追加的group
        size_t nr_supp_gids;                            //rc脚本中group option追加的group的数目
        const char* seclabel;                           //rc脚本中的seclabel option
        struct socketinfo *sockets;                     //rc脚本中的socket option
        struct svcenvinfo *envvars;
        struct action onrestart;                        //restart时的action， 由onrestart option指定
        std::vector<std::string>* writepid_files_;      //rc脚本中的writepid option
        
        /* keycodes for triggering this service via /dev/keychord */
        int *keycodes;
        int nkeycodes;
        int keychord_id;
        
        IoSchedClass ioprio_class;
        int ioprio_pri;
        
        int nargs;                                      //service的参数个数
        char *args[1];                                  //service的参数
    }; 

**init 中使用一个全局的链表 “service_list” 来保存所有的service的信息**

####4.2 init.rc脚本的处理

    action_for_each_trigger("early-init", action_add_queue_tail);

将所有由 “early-init” 这一trigger触发的action依次添加到待执行action队列 “action_queue” 中

    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");

添加一个init的内建的action,同时添加到待执行action队列 等待 “/dev/.coldboot_done” 文件出现才继续执行，该文件由ueventd创建， 标示ueventd已经成功启动， queue_buildin_action() 接收一个处理例程和一个trigger名称作为参数， 生成一个action， 添加到 全局的 action_list 链表和 action_queu 列表的末尾

    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    
    queue_builtin_action(keychord_init_action, "keychord_init");
    
添加一个init内建的action， 同时添加到待执行action队列， 该action负责初始化init的keychord功能, init 监听到“/dev/keychord” 接收组合键， 当读取到一个组合键后， 会启动所有在init.rc脚本中使用keycode option声明了该组合键的service， 当然只有 adbd service处于运行状态时， init才会处理keychord才会起作

    queue_builtin_action(console_init_action, "console_init");

添加一个init内建的action， 同时添加到待执行action队列， 若“ro.boot.console”属性指定的console存在， 则该actionl讲打开“/dev/tty0”， 并且在第14行写入“A N D R O I D” 字符， 即android设备开机时在屏幕上看到的字符

    action_for_each_trigger("init", action_add_queue_tail);
    
将所有由 “init” 这一trigger触发的action添加到带执行action队列 “action_queue” 尾部

    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    
    
    if (property_get("ro.bootmode", bootmode) > 0 && strcmp(bootmode, "charger") == 0) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }

若是充电模式， 则将将所有由 “charge” 这一trigger触发的action添加到带执行action队列 “action_queue” 尾部, 若是正常开机模式， 则将将所有由 “late-init” 这一trigger触发的action添加到带执行action队列 “action_queue” 尾部

    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");
    
添加所有能够被当前的property值trigger的action到待执行action队列

**所有的待执行的action都只是被添加到待执行队列中， 并未开始执行， 同样， 所有的service也为被启动， 只有等到init进入循环之后， 才开始执行待执行的action， 启动service**

####4.3　trigger的触发顺序

在开机过程中，　init.rc中，　各trigger的触发顺序如下：

    early-init
    init
    late-init
    
    early-fs
    fs
    post-fs
    post-fs-data
    load_all_props_action
    firmware_mounts_complete
    early-boot
    boot
    
其中，　前３个trigger由init触发，　后续的trigger则由late-init触发

**需要注意的是，　若是从charge mode boot时，　也会在init.rc　中触发late-init**

    on property:sys.boot_from_charger_mode=1
        class_stop charger
        trigger late-init
        
####4.4 service的启动顺序

对于声明了 option disable　的service, init 不会主动启动它们，　需要显式地通过　service　管理命令或者 trigger 来启动它们，　而对于需要自启动的service, 其启动顺序和service的class相关        

init.rc中定义的service可以使用　option class 来指定service的class, init.rc 中可以使用`class_start` `class_stop` `class_reset`　命令来控制同一class的service的启动/停止／重启, 共有如下几种class

+ 正常模式下：
   + core : 例如　ueventd, logd, healthd, console, vold, servicemanager, surfaceflinger, bootanim等service
   + main : 例如　netd, ril-daemo, keystore, mdnsd, zygote等service
   + late_start : logcatd, audiod等service
   + default : 这一类service不会被init自启动，　通常被声明为　disabled
+ 充电模式下:
   + charger

在正常模式下，　”core“，　“main”, "late_start"　这３种class的service同过如下的条件触发启动

    on boot
        ......
        class_start core

    on nonencrypted
        class_start main
        class_start late_start
        
    on property:vold.decrypt=trigger_restart_min_framework
        class_start main
        
    on property:vold.decrypt=trigger_restart_framework
        class_start main
        class_start late_start
        
需要注意的是上述的后３种trigger，　它们与android的设备加密相关，设备加密是为了保护android中的　／data　分区和 sdcard　中的用户数据，在 init.target.rc 中有　
        
        on fs
            wait /dev/block/bootdevice
            mount_all fstab.qcom
            
init 中由　do_mount_all()处理 init.rc　中的　mount_all　命令，它需要考虑设备加密的各种情况

+ FS_MGR_MNTALL_DEV_NEEDS_ENCRYPTION : 系统第一次开机且系统设置为设备必须加密
+ FS_MGR_MNTALL_DEV_MIGHT_BE_ENCRYPTED : 非系统第一次开机，　且设备已经加密
+ FS_MGR_MNTALL_DEV_NOT_ENCRYPTED : 非系统第一次开机，　且设备没有加密

设备加密后，系统的启动流程与设备未加密时由很大的不同，　仅仅考虑设备未加密时的情况，　init会在处理mount_all”　命令时，　触发　“nonencrypted”，　因此　“on nonencrypted”这一　action　会被添加到待执行action队列的末尾，　因此会在在　"on boot"　action　之后执行

**因此，　不同class的service的启动顺序为：　core, main, late_start，　且都位于“on boot“ trigger后**

#####5. init　循环处理事件

    while (true) {
        if (!waiting_for_exec) {
            execute_one_command();
            restart_processes();
        }
        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }
        if (!action_queue_empty() || cur_action) {
            timeout = 0;
        }
        bootchart_sample(&timeout);
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

进入　while　循环之后执行如下的步骤

+ 若当前没有正在执行的　”exec“　命令，　则
   1. 执行待执行action队列中的队列头的action的一条command　（每一轮循环只执行一条command，　处理完一个action后，　再处理下一个action）
   2. 重启所有需要重启的service
+ 计算监听事件的epoll　的timeout值，　如果还有待处理的action, 则timeout值为0，　否则可以放宽，　最长为5ms
+ 通过 epoll  监听事件

init通过　register_eapol_handler()　注册了３个文件的事件监听

1. /dev/keychord ：　监听key chord事件
2. /dev/spckets/property_service : 监听属性变化事件
3. 一个socket　pair, 当接收到SIGCHLD信号后，　init的信号处理函数从一端socket中写入”１“, init　poll另一端的socket以检查是否有子进程退出
