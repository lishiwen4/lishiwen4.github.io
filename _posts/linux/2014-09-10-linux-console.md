---
layout: post
title: "linux console"
description:
category: linux
tags: [linux, debug]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux 控制台  
   
在linux中存在着多种类型的终端， 如tty， pty， console  
  
###2.串行端口终端(/dev/ttySn)  
  
串行端口终端(Serial Port Terminal)是使用计算机串行端口连接的终端设备。计算机把每个串行端口都看作是一个字符设备。有段时间这些串行端口设备通常被称为终端设备，因为那时它的最大用途就是用来连接终端。这些串行端口所对应的设备名称是/dev/tts/0(或/dev/ttyS0), /dev/tts/1(或/dev/ttyS1)等，设备号分别是(4,0), (4,1)等，分别对应于DOS系统下的COM1、COM2等。若要向一个端口发送数据，可以在命令行上把标准输出重定向到这些特殊文件名上即可。例如，在命令行提示符下键入：  

	echo test > /dev/ttyS1

会把单词”test”发送到连接在ttyS1(COM2)端口的设备上。可接串口来实验。  
    
###3.伪终端(/dev/pty/)  
  
伪终端(Pseudo Terminal)是成对的逻辑终端设备(即master和slave设备, 对master的操作会反映到slave上)。
例如/dev/ptyp3和/dev/ttyp3(或者在设备文件系统中分别是/dev/pty/m3和 /dev/pty/s3)。它们与实际物理设备并不直接相关。如果一个程序把ptyp3(master设备)看作是一个串行端口设备，则它对该端口的读/ 写操作会反映在该逻辑终端设备对应的另一个ttyp3(slave设备)上面。而ttyp3则是另一个程序用于读写操作的逻辑设备。  
  
这样，两个程序就可以通过这种逻辑设备进行互相交流，而其中一个使用ttyp3的程序则认为自己正在与一个串行端口进行通信。这很象是逻辑设备对之间的管道操作。对于ttyp3(s3)，任何设计成使用一个串行端口设备的程序都可以使用该逻辑设备。但对于使用ptyp3的程序，则需要专门设计来使用 ptyp3(m3)逻辑设备。  
  
例如，如果某人在网上使用telnet程序连接到你的计算机上，则telnet程序就可能会开始连接到设备 ptyp2(m2)上(一个伪终端端口上)。此时一个getty程序就应该运行在对应的ttyp2(s2)端口上。当telnet从远端获取了一个字符时，该字符就会通过m2、s2传递给 getty程序，而getty程序就会通过s2、m2和telnet程序往网络上返回”login:”字符串信息。这样，登录程序与telnet程序就通过“伪终端”进行通信。通过使用适当的软件，就可以把两个甚至多个伪终端设备连接到同一个物理串行端口上。  
  
在使用设备文件系统 (device filesystem)之前，为了得到大量的伪终端设备特殊文件，使用了比较复杂的文件名命名方式。因为只存在16个ttyp(ttyp0—ttypf) 的设备文件，为了得到更多的逻辑设备对，就使用了象q、r、s等字符来代替p。例如，ttys8和ptys8就是一个伪终端设备对。不过这种命名方式目前仍然在RedHat等Linux系统中使用着。  
  
但Linux系统上的Unix98并不使用上述方法，而使用了”pty master”方式，例如/dev/ptm3。它的对应端则会被自动地创建成/dev/pts/3。这样就可以在需要时提供一个pty伪终端。目录 /dev/pts是一个类型为devpts的文件系统，并且可以在被加载文件系统列表中看到。虽然“文件”/dev/pts/3看上去是设备文件系统中的一项，但其实它完全是一种不同的文件系统。
即: TELNET ---> TTYP3(S3: slave) ---> PTYP3(M3: master) ---> GETTY  
  
在linux上， master端共享同一个 /dev/ptmx 设备节点(打开它内核将自动给出一个未分配的PTY)，所有slave端都位于 /dev/pts 目录下，名为 /dev/pts/# (内核会根据需要自动生成和删除它们)， 一旦master端被打开，相应的slave设备就可以按照与 TTY 设备完全相同的方式使用。master设备与slave设备之间通过内核进行连接，等价于拥有 TTY 功能的双向管道(pipe)  
  
###4.控制终端(/dev/tty)  
  
如果当前进程有控制终端(Controlling Terminal)的话，那么/dev/tty就是当前进程的控制终端的设备特殊文件(就像/proc/self 代表当前进程的目录一样)， 它看起来像一个到实际所使用的终端的一个链接。可以使用命令”ps –ax”来查看进程与哪个控制终端相连。对于你登录的shell，/dev/tty就是你使用的终端，设备号是(5,0)。使用命令”tty”可以查看它具体对应哪个实际终端设备。  
  
例如， 在ubuntu上， 使用 “shift + alt + f1” 切换到tty1登录后：  
  
	$ tty
    $ /dev/tty1
  
可以看到， 我们在tty1登录时， 使用的是虚拟终端tty1
而使用 “shift + alt + f7” 切换到图形终端， 然后使用 “ shift + alt + t” 打开一个终端后
  
	$ tty
    $ /dev/pts/9
    
这时， 使用的是伪终端9  
  
###5.控制台终端(/dev/ttyn, /dev/console)  
  
在Linux 系统中，计算机显示器通常被称为控制台终端 (Console)。它仿真了类型为Linux的一种终端(TERM=Linux)，并且有一些设备特殊文件与之相关联：tty0、tty1、tty2 等。当你在控制台上登录时，使用的是tty1。使用Alt+[F1—F6]组合键时，我们就可以切换到tty2、tty3等上面去。tty1–tty6等称为虚拟终端，而tty0则是当前所使用虚拟终端的一个别名，系统所产生的信息会发送到该终端上。因此不管当前正在使用哪个虚拟终端，系统信息都会发送到控制台终端上。你可以登录到不同的虚拟终端上去，因而可以让系统同时有几个不同的会话期存在。只有系统或超级用户root可以向 /dev/tty0进行写操作。  
  
终端控制台要是串口控制台， 要么是虚拟控制台（vga + 键盘）。
  
###6. /dev/tty 和 /dev/tty0的差别  
  
这两个都是特殊的终端设备文件， /dev/tty 代表当前所使用的终端， 实际上使用可能的是串行终端(/dev/ttySn), 伪终端(/dev/pts/n), 或者虚拟终端（/dev/ttyn）。  
  
而 /dev/tty0, 指代的是当前所使用的虚拟终端, 例如在tty1 上登录， /dev/tty0 就代表 tty1， 而通常在图形模式下登录时， 对应的是/dev/tty7
  
###7. /dev/ttyn 与 /dev/console的关系  
  
根据内核文档，在2.1.71之前，/dev/console根据不同系统的设定可以链接到/dev/tty0或者其他tty＊上，在2.1.71版本之后则完全由内核控制。目前，只有在单用户模式下可以登录/dev/console（可以在单用户模式下输入tty命令进行确认）。  
  
在ubuntu 14.04开机到grub时，用上下键移到第二行的恢复模式，按e（注意不是回车），把ro recovery nomodeset 改成rw single init=/bin/bash， 然后按f10引导进入单用户模式后  
  
	root@(none):/tty
    /dev/console
    
历史上，console指主机本身的屏幕键盘，而tty指用电缆链接的其它位置的控制台(仅包含屏幕和键盘)。  
当前被激活的tty会输出到console上。  
  
console和tty有很大区别：console是个只输出的设备，功能很简单，只能在内核中访问；tty是char设备，可以被用户程序访问。  
  
实际的驱动比如串口对一个物理设备会注册两次，一个是tty，一个是console，并通过在console的结构中记录tty的主次设备号建立了联系。 
  
在内核中，tty和console都可以注册多个。当内核命令行上指定console=ttyS0之类的参数时，首先确定了printk实际使用那个console作为输出，其次由于console和tty之间的对应关系，打开/dev/console时，就会映射到相应的tty上。用一句话说：/dev/console将映射到默认console对应的tty上。   
  
###8. boot console 与 real console  
  
在系统启动过程中，在正真的console driver初始化好之前，就需要打印内核消息， 这时可注册 boot console， 它可能直接操作裸机来输出， 并且可以在任何时候注册 boot console， 等到真正的console driver ready 后， 会注册real console， 一旦real console者测成功， 所有的boot console将被移除， 并且后续注册boot console都会失败。
  
###9. 内核参数cosnole项的处理  
  
在cosnole_setup（）中调用_ _add_preferred_console 把其添加到全局变量console_cmdline数组中。    
  
串口驱动在注册tty后， 调用register_console(), 在这里会将匹配console_cmdline数组。
