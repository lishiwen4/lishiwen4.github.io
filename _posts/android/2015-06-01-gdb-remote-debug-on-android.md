---
layout: post
title: "gdb 远程调试在android上的应用"
description:
category: android
tags: [android, debug]
mathjax: 
chart:
comments: false
---

###1. gdb 远程调试  

如果要调试运行于一个不能运行GDB的机器上的程序时，使用远程调试就很有帮助了。例如，可以用远程调试操作系统内核，或者在小型的，没有足够的操作系统能力来支持运行完全功能的调试器的系统里调试, gdbd的远程调试采用gdb+gdbserver的远程调试方法：gdbserver在目标板中运行，而gdb则在主机上运行(主机上的gdb是特殊版本， 可在主机上运行， 但是针对目标平台编译， )

GDB可以配置特殊串口和TCP/IP接口来支持远程调试特殊的调试目标。另外，GDB还有通用串口协议（特指GDB，不是指任何特殊目标系统），如果你可以用它实现远程代理–远程代理的代码运行于远程系统，用来和GDB通讯

Android 上通常都自带了gdbserver的源码并且编译出可执行文件，需要注意的是， 64位系统上一般带有2个版本: gdbserver, gdbserver64, 需要跟据要调试的程式选用正确的版本  

而gdb则可以使用编译android的源码所使用的交叉编译器中自带的gdb， 其路径可使用如下的方式

	$ source build/envsetup.sh
	$ lunch
	$ env | grep ANDROID_TOOLCHAIN

使用获取的路径中的 *gdb 即可

宿主机上的gdb使用“target remote”命令来连接gdbserver， 对于串行线和tcp/udp方式的usage 分别如下

	target remote serial-device
	target remote host:port

例如
	target remote /dev/ttyb
	target remote tcp::5039
	target remote udp::5039

gdb和gdbserver需使用相同的方式， 通常都是使用TCP的方式， 在使用KGDB调试内核时， 会用到串行线
在使用串行口的时候， 还需要设置波特率， 流控等

###2. 设置端口转发

gdb的远程调试支持 tcp/串行口等通信方式，android上通常并不方便使用串行线，android设备和PC之间使用usb来连接， 因此， 可以要使用adb 的端口转发功能在PC和android设备之间通过adb来转发端口数据，通过tcp来连接gdbserver，adb转发tcp端口的cmd如下：

	$  adb forward tcp:pc-port tcp:android-port

例如， 若想要将pc的tcp端口1234转发到android设备的tcp端口4321， 则可以使用

	$ adb forward tcp:1234 tcp:4321

使用如下命令还可以查看已存在的转发 / 移除存在的转发

	$ adb forward --list
	$ adb forward --remove <local> 
	$ adb forward --remove-all

事实上， “adb forward” 命令还可以建立 unix socket等其它类型的转发， 详细信息可以查看 adb 的 help 

例如， 要在android设备上， gdbserver监听tcp端口5555， 在pc上gdb使用tcp端口4444与其通信

	$ adb forward tcp:4444 tcp:5555

###3. 启动adbserver  

在 android 设备上， gdbserver的usage如下

	Usage:	gdbserver [OPTIONS] COMM PROG [ARGS ...]
		gdbserver [OPTIONS] --attach COMM PID
		gdbserver [OPTIONS] --multi COMM

	COMM may either be a tty device (for serial debugging), or 
	HOST:PORT to listen for a TCP connection.

+ 对于已经启动运行的程序，例如android中的一些service， 可以使用“--attach“选项附着上去
+ 对于应用程序， 可以直接使用gdbserve来启动

例如在android上以dgbserver来启动一个”test“应用， 并且在端口tcp端口5555监听， 则可以使用

	$ gdb :5555 /system/bin/test 

可以看到如下输出

	Listening on port 5555

###4. 在pc端启动gdb

以MSM8916平台为例， 附带的gdb路径为

	prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android-gdb

usage为

	aarch64-linux-android-gdb
	aarch64-linux-android-gdb  program

可以使用两种方式指定需要调试的可执行文件：

+ 在启动gdb时，直接指定需要调试的program
+ 在启动gdb时不指定， 在启动后通过“file”命令来指定

**gdb 调试时， 最好使用unstripe的可执行文件， android 源码中build出的可执行文件和lib可以在在out/target/product/xxxx/symbols/ 下找到对应的unstripe版**

例如， 在pc上启动gdb连接gdbserver来调试test

	$ aarch64-linux-android-gdb ~/android/out/target/product/msm8916/symbols/system/bin/test

启动后， 在pc端输出：

	GNU gdb (GDB) 7.7
	Copyright (C) 2014 Free Software Foundation, Inc.
	License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
	This is free software: you are free to change and redistribute it.
	There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
	and "show warranty" for details.
	This GDB was configured as "--host=x86_64-linux-gnu --target=aarch64-elf-linux".
	Type "show configuration" for configuration details.
	For bug reporting instructions, please see:
	<http://source.android.com/source/report-bugs.html>.
	Find the GDB manual and other documentation resources online at:
	<http://www.gnu.org/software/gdb/documentation/>.
	For help, type "help".
	Type "apropos word" to search for commands related to "word"...
	Reading symbols from /home/sven/src/ze550kl-dev/out/target/product/Z00L/symbols/system/bin/gdbtest...done.
	(gdb)

如果之前未设定好端口转发， 则可在gdb中设置

	(gdb) shell adb forward tcp:4444 tcp:5555

连接gdbserverl， 对于使用tty或者tcp连接方式， usage分别如下

	target remote host:port
	target remote /dev/ttyx

在 android 调试时， 一般使用pc的本机端口， 因此， cmd如下

	(gdb) target remote :4444

连接成功后， 可见android端输出

	Remote debugging from host 127.0.0.1

设置动态库的路径，通常，编译时连接的库都会使用绝对路径， 例如 "/lib/libc.so.6"、"/lib/libdl.so.2"， 而在运行时使用dlopen()打开的库则可能会使用相对路径， 例如“"./libddd.so"，指定动态库的搜索路径涉及到两条cmd

+ set sysroot : 别名为“set solib-absolute-prefix”， 从名字上看， 就是设置根目录， linux中的可执行文件， 动态库都有标准的搜索路径(比如 /bin/ usr/bin /lib 等)， 设置一个根目录后， 将从该目录下的标准目录搜索可执行文件以及动态库， 在远程调试android应用时， 需要指定根目录为“xxx/out/target/product/xxxx/symbols”， 根目录只能设定一个
+  set solib-search-path ： 设定动态库的搜索路径， 对于没有放置在标准目录中的库， 需要添加搜索路径才能被找到，类似于linux下没有放置在标准目录中的可执行文件必须要添加到PATH变量中才能被搜索到一样， android中的动态库放置在 system/lib 下， 因此需要添加搜索路径“xxx/out/target/product/xxxx/symbols/system/lib”(64位库需要使用lib64), 和只能设置一个根目录不同， 搜索路径可以设置多个， 中间使用“:”分割

然后， 就可以开始进行调试了

若需要给要调试的执行程序指定参数， 有两种方式：

+ 在使用 "gdb cmd" 方式指定执行程序时， 后跟参数
+ 在启动gdb后， 使用“set args”来指定参数

**另外， Android上提供了gdbclient来简化设置， 只需要启动gdbserver， 在pc端执行“source, lunch”后， 设置好端口转发(gdbclient默认使用5039号端口)， 然后执行“gdbclient test”即可**

###5. GDB cmd

####5.1 获取gdb的帮助信息

help 命令列出所有的cmd， 然后使用“help cmd”可以列出各cmd的详细帮助信息， 例如 “help breakpoint” 即可列出“breakpoint” 的详细帮助信息

####5.2 show

用来获取GDB本身设置相关的一些信息

####5.3 查看被调试程序的相关信息

用来获取和被调试的应用程序相关的信息, 常用的有

+ info address : 查看符号的地址， 例如 “info address main” 可产看main()的地址
+ info all-registers : 打印所有整数寄存器的内容
+ info args ： 查看可执行程序的启动参数
+ info breakpoints  ： 显示所有的断点的信息
+ info frame ： 列出所有选中的栈帧的信息
+ info functions ： 列出所有函数名
+ info line ： 列出指定行的代码地址， 例如“info line 10” 列出了源码中第10行的代码的地址
+ info locals ： 列出当前栈帧中的局部变量的值， 包括静态变量
+ info macro ： 显示指定的宏(需要在编译时指定“-ggdb3”选项)
+ info macros ： 显示所有的宏(需要在编译时指定“-ggdb3”选项)
+ info os : 显示目标系统的相关信息， 例如sockt， 线程， 进程， 打开的文件等相关信息， 具体的信息视目标设备可能有所不同
+ info registers ： 显示所有寄存器和其中的内容
+ info set ： 显示GDB的相关设定
+ info sharedlibrary ： 显示所有加载的动态库
+ info signals ： 显示GDB对各信号的处理方式
+ info source ： 显示当前的source文件的信息
+ info sources ： 显示所有的source文件的信息
+ info stack ： 显示调用栈
+ info symbol ： 显示当前地址的符号
+ info starget ： 显示调试的程序的信息
+ info threads ： 列出所有的线程
+ info vtbl ： 显示c++的虚函数表
+ info watchpoints ： 显示所有的watchpoints

####5.4 设置断点

可以使用如下的方式指定断点的位置

+ break function ： 在某个函数的入口处设置断点， 例如 “break main”
+ break +offset ： 在当前位置的向下offset行处设置断点， 例如 “break +10”
+ braek –offset : 在当前位置的向上offset行处设置断点， 例如 “break -10”
+ break linenum : 在当前源文件的指定行设置断点， 例如 “break 10”
+ break filename:linenum ： 在指定文件的指定行设置断点， 例如 “break test.c:10”
+ break filename:function : 在指定文件的指定程序入口处设置断点， 例如“break test.c:hello”
+ break : 在当前位置设置断点

还可以使用rbreak cmd，在所有符合正则表达式的的地方设置断点

+  rbreak regex

默认指定的断点， 在enable时每次都会触发，暂停程序的运行， 可以设置条件断点， 在为真的时候才触发

+ break xxxx if exp 
+ condition bnum expression ： 这个需要在已经设定号断点后， 给断点设置条件
+ condition n ： 取消第n个断点的条件， 将会一致触发

也可以使用 tbreak 指定只触发一次的断点

+ tbreak xxx

有些平台上还可以支持硬件断点， 可以使用“hbreak”来设置硬件断点，通常支持的硬件断点数量受限， ARM7/9最多支持2个， ARM11最多支持8个， 软件断点通常是替换目标位置的指令， 插入“INT 3”， 在中断后暂停执行

查看断点 ：

+ info break n ： 查看编号为n的断点， 每设置一个断点，都会分配一个ID， 依次递增， 即使是旧的断点被取消掉
+ info break ： 查看所有的断点

删除断点

+ clear : 清除当前栈帧下的下一条指令之后的所有断点
+ clear function ： 清除函数function入口处的断点
+ clear filename:function ： 清除函数文件filename中function入口处的断点
+ clear linenum ： 清除当前文件的指定行的断点
+ delete [breakpoints] [range…] ： 删除指定范围的断点， range可以是单个断点的编号， 也可以是编号的范围

禁用/启用断点

+ disable [breakpoints] [range…] : 禁用指定范围的断点， range可以是单个断点的编号， 也可以是编号的范围
+ enable [breakpoints] [range…] ： 启用指定范围的断点， range可以是单个断点的编号， 也可以是编号的范围
+ enable [breakpoints]once [range…] : 只启用一次指定范围的断点，程序停止下来后，自动禁用这些断点， range可以是单个断点的编号， 也可以是编号的范围
+ enable [breakpoints]delete [range…] : 只启用一次指定范围的断点，程序停止下来后，自动删除这些断点， range可以是单个断点的编号， 也可以是编号的范围

忽略断点

+  ignore n count ： 忽略第n个断点count次

指定在某个断点处停下来后执行的动作：

	commands [bnum]
	… command-list …
	end

若不指定num，则缺省为最后一个断点

例如，指定编号为10的断点处的动作

	commands 10

	silent
	printf "x is %d\n",x
	continue

	end

可以取消指定断点处的动作

	commands [bnum]
	end

####5.5 设置观察点

在某个表达式的值发生变化的时候停止运行，达到‘监视’该表达式的目的

表达式可以是一个符号， 若添加“-l”选项， 则watch表达式所指的地址  

+ watch expr : 设置写watchpoint，当应用程序写expr,修改其值时，程序停止运行
+ rwatch expr: 设置读watchpoint，当应用程序读表达式expr时，程序停止运行
+ awatch expr: 设置读写watchpoint, 当应用程序读或者写表达式expr时，程序都会停止运行

watchpoint和breakpoint共用一套ID空间， 使用相同的语法来 enable/disable/delete

####5.6 设置异常点

catchpoints的作用是让程序在发生某种事件的时候停止运行，比如C++中发生异常事件，加载动态库事件

+ catch event : 当事件发生时， 程序停止运行

支持的event有：

1. throw: C++抛出异常
2. catch: C++捕捉到异常
3. exec: exec被调用
4. fork: fork被调用
5. vfork: vfork被调用
6. load:加载动态库
7. loadlibname: 加载名为libname的动态库
8. unload:卸载动态库
9. unloadlibname: 卸载名为libname的动态库
10. syscall [args]:调用系统调用，args可以指定系统调用号，或者系统名称

catchpoints同watchpoint一样， 使用相同的语法来 enable/disable/delete

####5.7 附加到进程上 

+ attach pid ： 附着到一个已经运行的进程上， 进行调试
+ detach ： 取消attach

####5.8 多线程相关

在多线程程序时， 需要用到thread相关命令

+ thread tid  ： 切换当前线程到由tid指定的线程
+ info threads：查看GDB当前调试的程序的各个线程的相关信息
+ thread apply < threadno | all > args：对指定（或所有）的线程执行由args指定的命令

在使用step或者continue命令调试当前被调试线程的时候，其他线程也是同时执行的，有时候需要只让被调试程序执行， 可以通过锁定其它的线程来实现

+ set scheduler-locking off : 不锁定任何线程，也就是所有线程都执行，这是默认值
+ set scheduler-locking on : 有当前被调试的线程会执行
+ set scheduler-locking step，在单步的时候，除了next过一个函数的情况以外，只有当前线程会执行

####5.9 多进程相关

默认在fork/vfork之后，GDB仍然调试父进程，与子进程不相关

+ set follow-fork-mode mode ： mode为"parent"时，与缺省情况一样；mode为"child"时，fork/vfork之后，GDB进入子进程调试，与父进程不再相关
+ show follow-fork-mode：查看当前GDB多进程跟踪模式的设置

####5.10 单步执行

+ step count : 如果没有指定count, 则继续执行程序，直到到达与当前源文件不同的源文件中时停止；如果指定了count,则重复行上面的过程count次
+ stepi count :如果没有指定count, 继续执行下一条机器指令，然后停止；如果指定了count，则重复上面的过程count次

step比较常见的应用场景：在函数func被调用的某行代码处设置断点，等程序在断点处停下来后，可以用step命令进入该函数的实现中，但前提是该函数编译的时候把调试信息也编译进去了，负责step会跳过该函数

+ next [count]:如果没有指定count, 单步执行下一行程序；如果指定了count，单步执行接下来的count行程序
+ nexti [count]:如果没有指定count, 单步执行下一条指令；如果指定了count,单步执行接下来的count条执行

**stepi和nexti的区别：nexti在执行某机器指令时，如果该指令是函数调用，那么程序执行直到该函数调用结束时才停止， 而stepi则会进入被调用的函数的内部**

####5.11 继续执行程序**

+ continue [count] : 唤醒程序，继续运行，至到遇到下一个断点，或者程序结束。如果指定count，那么程序在接下来的运行中，count次断点
+ finish : 继续执行程序，直到当前被调用的函数结束，如果该函数有返回值，把返回值也打印到控制台
+ return [expression] : 中止当前函数的调用，如果指定了expression，把expresson值当做当前函数的返回值；如果没有，直接结束当前函数调用

####5.12 信号的处理

+ info signals ： 打印所有的信号相关的信息，以及GDB缺省的处理方式 
+ info handle ： 同上
+ handle signal keywords : 设置GDB对具体某个信号的处理方式。signal可以为信号整数值，也可以为SIGSEGV这样的符号keywords的取值如下

1. stop ：当GDB收到指定的信号，停止应用程序的执行
2。nostop ： 当GDB收到指定的信号，不会应用停止程序的执行，只会打印出一条收到信号的消息
3. print ：表示如果收到指定的信号，打印出一条信息
4. noprint：与print表示相反
5. pass ：表示如果收到指定的信号，把该信号通知给应用程序
6. nopass ：nopass表示与pass相反的意思
7. ignore ：ignore与nopass同义
8. noignore ： noignore与pass同义

可以在gdb中向被调试的程序发送信号

+ signal signal: 向程序发送信号signal,signal可以是信号的符号或数字形式，如果signal=0，那么程序将会继续运行，程序不会收到任何信号

####5.13 查看调用栈

+ backtrace ： 查看调用栈信息
+ backtrace n ： 查看调用栈信息, 只显示栈顶n帧
+ backtrace -n ： 查看调用栈信息, 只显示栈底部n帧
+ backtrace limit n ： 设置 backtrace 最大显示层数为n

另外 “where”， “info stack” 都是backtrace的别名， 可用于查看调用栈以及当前的位置， backtrace输出的内容格式如下：

	(gdb) backtrace 
	#0  add (a=0, b=1) at test.c:11
	#1  0x0000000000400654 in main () at test.c:26

包含了栈桢号， 函数名， 形参名和值， 源码文件名和行号

backtrace可以查看调用栈的大致信息， 而如果想要查看单个栈桢的信息， 则需要用到“frame”命令

+ framen: 查看第n桢的信息
+ frameaddr: 查看pc地址为addr的桢的相关信息
+ upn: 查看当前桢上面第n桢的信息
+ downn: 查看当前桢下面第n桢的信息

其输出形式如下

	(gdb) frame 0
	#0  add (a=0, b=1) at test.c:11
	11		return c;

若需要输出最详细的信息， 应使用“info frame”命令

+ info frame、： 
+ info framen ： 
+ info frameaddr ： 

输出的内容如下：

	(gdb) info frame 0
	Stack frame at 0x7fffffffdfa0:
	 rip = 0x4005d0 in add (test.c:11); saved rip 0x400654
	 called by frame at 0x7fffffffdff0
	 source language c.
	 Arglist at 0x7fffffffdf90, args: a=0, b=1
	 Locals at 0x7fffffffdf90, Previous frame's sp is 0x7fffffffdfa0
	 Saved registers:
	  rbp at 0x7fffffffdf90, rip at 0x7fffffffdf98

包含的内容有：

1. 当前桢的地址
2. 下一桢的地址
3. 上一桢的地址
4. 源代码所用的程序的语言(c/c++)
5. 当前桢的参数的地址
6. 当前桢中局部变量的地址
7. PC（program counter）
8. 当前桢中存储的寄存器

####5.14 被调试的程序源码目录

+ directory dirname : 设置当前调试的程序的源代码目录为dirname
+ directory : 将当前调试的程序的源代码目录清空
+ show directories : 打印当前调试的程序的源代码目录

####5.15 查看程序的源码

+ list linenum: 打印当前文件中第linenum行周围的源代码
+ list function: 打印function函数周围的源代码
+ list : 在上一次使用list命令的基础上，再多打印一些源代码
+ list -: 打印和上一次使用list命令一样的源代码
+ set listsizecount: 设置list命令显示源代码的行数
+ show listsize : 查看list命令显示

####5.16 查看变量

+ print expr: 打印表达式expr的值
+ print /f expr:以f指定的格式打印表达式expr的值
+ print : 打印上一次打印的表达式的值
+ print /f: 以f指定的形式打印上一次打印的表达式的值

print支持的格式有：

1. x-16进制整数
2. d-10进制整数
3. u-10进制无符号整数
4. o-8进制整数
5. t-2进制整数
6. a-地址形式整数
7. c-字符常量整数
8. f-浮点数

例如， 对于一个局部变量 “int start=1”

	(gdb) print start
	$6 = 1
	(gdb) print /x start
	$7 = 0x1

再来看表达式， gdb支持的表达式有

+ 寄存器 ： 使用“info register”查看到的所有的寄存器均可使用， 例如，使用“$rip”, "$cs", "$ds"分别引用了pc寄存器， cs寄存器， ds寄存器 
+ 由所调试的语言中的任何合法的常量， 变量和操作符所组成的表达式, 例如c语言中的加减乘除， 移位， 取地址等操作
+ array@len ： 将arry所在的内存数组来打印，打印n个元素，每一个元素的内存大小由符号arry来确定， 若为常量指针， 则大小为4byte， 例如（“print start@2”, "print *0x7fffffffdfcc@4", "print *array@1"）
+ file::variable ： 指定文件中的变量
+ function::varable : 指定函数中的变量
+ {type}address: 把address指定的内存解释为type类型（类似于强制转型，更加强）例如“ p /x {short}0x7fffffffdfcc”

查看变量的类型

+ whatis var

####5.17 打印内存

使用命令 “x"

	x /nfu addr

n为次数， f为打印格式同printf， 但是还新增“i”用于打印机器指令， u为单位的大小,b-byte，h-halfwords（2字节），w-words（4字节），g-Giant words（8字节）

可以使用“x”命令来打印汇编指令， 相当于反汇编， 例如打印当前位置的2条汇编指令

	(gdb) x/2i $rip
	=> 0x40064c <main+119>:	mov    0x2009ee(%rip),%eax        # 0x601040 <count.2653>
	   0x400652 <main+125>:	mov    %eax,%esi

例如打印当前位置向后的2条汇编指令

	(gdb) x/2i $rip-2
	   0x40064a <main+117>:	jmp    0x400682 <main+173>
	=> 0x40064c <main+119>:	mov    0x2009ee(%rip),%eax        # 0x601040 <count.2653>



很多时候“print” 和 “x”都能完成相同的工作， “print”使用更为友好，能够直接打印变量的值和类型， 直接打印指针和字符串, 而“x”则更为强大

####5.18 自动显示

“display“命令可以在程序停止时， 自动打印变量或者内存值

+ display /f expr\|addr：以f为格式(具体格式同print)，打印expr或者addr
+ undisplaydnums ： 删除第dnums个display点
+ delete display dnums：同上
+ disable displaydnums：禁用第dnums个display点
+ enable displaydnums：启用第dnums个display点
+ info display：查看所有的display点

####5.19 打印选项  

gdb中有一些打印选项， 可以使用 “set print xxx [on \| off]” 来打开或者关闭， 使用“show print xxx”来查看  
  
+ array：以一种比较好看的方式打印数组，缺省是关闭的
+ elementsnum-of-elements：设置GDB打印数据时显示元素的个数，缺省为200，设为0表示不限制(unlimited)
+ null-stop：设置GDB打印字符数组的时候，遇到NULL时停止，缺省是关闭的
+ pretty：设置GDB打印结构的时候，每行一个成员，并且有相应的缩进，缺省是关闭的
+ object：设置GDB打印多态类型的时候，打印实际的类型，缺省为关闭
+ static-members：设置GDB打印结构的时候，是否打印static成员，缺省是打开的
+ vtbl：以漂亮的方式打印C++的虚函数表，缺省是关闭的

####5.20 修改变量的值

有两种方式

1. printv=value:　修改变量值的同时，把修改后的值显示出来
2. set [var]v=value:　修改变量值，需要注意如果变量名与GDB中某个set命令中的关键字一样的话，前面加上var关键字

####5.20 在gdb中调用被调试程序的函数

+ call exp

例如， 被调试的文件中有函数

	int add(int a, int b)

则在gdb中可以调用add

	(gdb) call add(1, 5)

####5.21 被调试的程序的启动参数

可以在启动gdb时使用“--argss选项”指定参数， 例如

	$ gdb --args ./test 1 2 3 4 5

也可以在进入adb后， 手动设置参数， 例如

	(gdb) set args 1 2 3 4 5

要查看启动参数， 可以使用
	
	(gdb) show args
	Argument list to give program being debugged when it is started is "1 2 3 4 5".

####5.21 改变程序的执行

如果认为在程序中发现了错误，也许你想知道，纠正当前的错误是否能让剩余的运算得到正确的结果， 使用GDB的执行中改变的功能，通过实践就可以找到答案

可以改变条件变量来改变程序的执行路径， 使用“print”或者set命令来完成

可以通过修改函数的返回值来改变程序的执行流程，使用 “return” 命令 可以让函数立即以指定的返回值来返回

可以直接在gdb中跳转到指定地址上执行

+ jump linespec ： 跳转到指定的行
+ jump location ： 跳转到指定的地址

如果在跳转到的位置上有断点的话，执行会立即中断，jump命令惯常会和tbreak命令联合使用， jump命令除了改变程序计数器之外，不会改变当前堆栈帧，堆栈指针，也不改变任何内存位置和任何寄存器。如果行linespec是在当前执行的函数之外的函数里，如果这两个函数的参数模式或者本地变量不一样的话，那么结果就可能很诡异。由于这个原因，如果指定的行不在当前执行的函数里的话，jump命令需要用户确认

**最常使用jump命令的场景在于回退已经执行过的程序–中间可能有几个断点，可以更详细的检查程序的执行**

很多系统里面， 手动修改“$pc”寄存器可以达到和 jimp 相同的目的


