---
layout: post
title: "VDSO和系统调用"
description:
category: linux
tags: [linux, build-link]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. VDSO

VDSO就是Virtual Dynamic Shared Object，就是内核提供的虚拟的.so,这类.so文件不在磁盘上，而是包含在内核里面。内核把包含某一.so的物理页在程序启动的时候映射入其进程的内存空间，对应的程序就可以当普通的.so来使用里头的函数

linux上目前使用的vdso是“linux-vdso.so.1”(旧版的名称可能是“linux-gate.so.1”)， 通过ldd命令可以发现，基本所有的应用都会依赖这一so

	$ ldd cat 
	linux-vdso.so.1 =>  (0x00007fff47ffe000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f45f6af2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f45f6ecc000)

VDSO在进程中映射的地址可通过如下方式来查看:

	$ cat /proc/xxx/maps | grep  vdso

需要注意的是， 新的内核中提供了的进程地址随机化功能， 因此，linux-vdso.so.1每一次的映射地址都不相同(即使是对于同一个可执行程序), linux中的"/proc/sys/kernel/randomize_va_space"用于控制linux的进程地址随机化, 取值如下：

+ 0 ： 表示关闭内存地址随机化 
+ 1 ： 表示将mmap的基址，stack和vdso地址随机化
+ 2 ： 表示在1的基础上， heap地址随机化

可通过如下命令关闭进程地址随机化

	$ echo "0" > /proc/sys/kernel/randomize_va_space

###2. VDSO和glibc

往往linux中新增加一个系统调用， glibc要花很久才能添加支持， 因为glibc还要支持 BSD，SysV Windows， solari， hurd等， 而linux同样也要兼容不同版本的glibc， 用户可能会经常升级内核而不升级glibc， 这样双方都背负了很大的历史包袱， 为次linux让glibc变成VDSO进驻内核， 这样，随内核发行的libc就唯一的和一个特定版本的内核绑定到一起了， 双方都不需要为了兼容对方的多个版本而额外添加一堆代码， 也减少了bug， 当然， VDSO中的glibc只是对系统调用的包裹，和平台无关的部分还是存放在用户空间

###3. VDSO和系统调用

####3.1 int 0x80

在 Linux 2.4 内核中，用户态 Ring3 代码请求内核态 Ring0 代码完成某些功能是通过系统调用完成的，而系统调用的是通过软中断指令（int 0x80）实现的。在 x86 保护模式中，处理 INT 中断指令时，CPU 首先从中断描述表 IDT 取出对应的门描述符，判断门描述符的种类，然后检查门描述符的级别 DPL 和 INT 指令调用者的级别 CPL，当 CPL<=DPL 也就是说 INT 调用者级别高于描述符指定级别时，才能成功调用，最后再根据描述符的内容，进行压栈、跳转、权限级别提升。内核代码执行完毕之后，调用 IRET 指令返回，IRET 指令恢复用户栈，并跳转会低级别的代码。

在发生系统调用，由 Ring3 进入 Ring0 的这个过程浪费了不少的 CPU 周期，例如，系统调用必然需要由 Ring3 进入 Ring0（由内核调用 INT 指令的方式除外，这多半属于 Hacker 的内核模块所为），权限提升之前和之后的级别是固定的，CPL 肯定是 3，而 INT 80 的 DPL 肯定也是 3，这样 CPU 检查门描述符的 DPL 和调用者的 CPL 就是完全没必要  

####3.2 sysenter/sysexit

Intel x86 CPU 从 PII 300（Family 6，Model 3，Stepping 3）之后，开始支持新的系统调用指令 sysenter/sysexit。sysenter 指令用于由 Ring3 进入 Ring0，SYSEXIT 指令用于由 Ring0 返回 Ring3。由于没有特权级别检查的处理，也没有压栈的操作，所以执行速度比 INT n/IRET 快了不少

在 Intel 的软件开发者手册第二、三卷（Vol.2B,Vol.3）中，4.8.7 节是关于 sysenter/sysexit 指令的详细描述。手册中说明，sysenter 指令可用于特权级 3 的用户代码调用特权级 0 的系统内核代码，而 SYSEXIT 指令则用于特权级 0 的系统代码返回用户空间中。sysenter 指令可以在 3，2，1 这三个特权级别调用（Linux 中只用到了特权级 3），而 SYSEXIT 指令只能从特权级 0 调用

在 Intel 的手册中，还提到了 sysenter/sysexit 和 int n/iret 指令的一个区别，那就是 sysenter/sysexit 指令并不成对，sysenter 指令并不会把 SYSEXIT 所需的返回地址压栈，sysexit 返回的地址并不一定是 sysenter 指令的下一个指令地址。调用 sysenter/sysexit 指令地址的跳转是通过设置一组特殊寄存器实现的。这些寄存器包括：

1. SYSENTER_CS_MSR － 用于指定要执行的 Ring 0 代码的代码段选择符，由它还能得出目标 Ring 0 所用堆栈段的段选择符；
2. SYSENTER_EIP_MSR － 用于指定要执行的 Ring 0 代码的起始地址；
3. SYSENTER_ESP_MSR－用于指定要执行的Ring 0代码所使用的栈指针

这些寄存器可以通过 wrmsr 指令来设置，执行 wrmsr 指令时，通过寄存器 edx、eax 指定设置的值，edx 指定值的高 32 位，eax 指定值的低 32 位，在设置上述寄存器时，edx 都是 0，通过寄存器 ecx 指定填充的 MSR 寄存器，sysenter_CS_MSR、sysenter_ESP_MSR、sysenter_EIP_MSR 寄存器分别对应 0x174、0x175、0x176，需要注意的是，wrmsr 指令只能在 Ring 0 执行

在 Ring3 的代码调用了 sysenter 指令之后，CPU 会做出如下的操作：

1. 将 SYSENTER_CS_MSR 的值装载到 cs 寄存器.
2. 将 SYSENTER_EIP_MSR 的值装载到 eip 寄存器.
3. 将 SYSENTER_CS_MSR 的值加 8（Ring0 的堆栈段描述符）装载到 ss 寄存器.
4. 将 SYSENTER_ESP_MSR 的值装载到 esp 寄存器.
5. 将特权级切换到 Ring0.
6. 如果 EFLAGS 寄存器的 VM 标志被置位，则清除该标志.
7. 开始执行指定的 Ring0 代码.

在 Ring0 代码执行完毕，调用 SYSEXIT 指令退回 Ring3 时，CPU 会做出如下操作：

1. 将 SYSENTER_CS_MSR 的值加 16（Ring3 的代码段描述符）装载到 cs 寄存器.
2. 将寄存器 edx 的值装载到 eip 寄存器.
3. 将 SYSENTER_CS_MSR 的值加 24（Ring3 的堆栈段描述符）装载到 ss 寄存器.
4. 将寄存器 ecx 的值装载到 esp 寄存器.
5. 将特权级切换到 Ring3.
6. 继续执行 Ring3 的代码.

由此可知，在调用 SYSENTER 进入 Ring0 之前，一定需要通过 wrmsr 指令设置好 Ring0 代码的相关信息，在调用 SYSEXIT 之前，还要保证寄存器edx、ecx 的正确性

**AMD 的 CPU 支持一套与之对应的指令 SYSCALL/SYSRET。在纯 32 位的 AMD CPU 上，还没有支持 sysenter 指令，而在 AMD 推出的 AMD64 系列 CPU 上，处于某些模式的情况下，CPU 能够支持 sysenter/sysexit 指令。在 Linux 内核针对 AMD64 架构的代码中，采用的还是 SYSCALL/SYSRET 指令**

####3.3 检查cpu是否支持 sysenter/sysexit

根据 Intel 的 CPU 手册，我们可以通过 CPUID 指令来查看 CPU 是否支持 sysenter/sysexit 指令，做法是将 EAX 寄存器赋值 1，调用 CPUID 指令，寄存器 edx 中第 11 位（这一位名称为 SEP）就表示是否支持。在调用 CPUID 指令之后，还需要查看 CPU 的 Family、Model、Stepping 属性来确认，因为据称 Pentium Pro 处理器会报告 SEP 但是却不支持 sysenter/sysexit 指令。只有 Family 大于等于 6，Model 大于等于 3，Stepping 大于等于 3 的时候，才能确认 CPU 支持 sysenter/sysexit 指令

####3.4 linux对2.6的两种调用方式的支持

在 2.6 内核中，内核代码同时包含了对 int 0x80 中断方式和 sysenter 指令方式调用的支持，因此内核会给用户空间提供一段入口代码，内核启动时根据 CPU 类型，决定这段代码采取哪种系统调用方式。对于 glibc 来说，无需考虑系统调用方式，直接调用这段入口代码，即可完成系统调用。

支持 sysenter 指令的代码包含在文件 arch/i386/kernel/vsyscall-sysenter.S 中，支持int中断的代码包含在arch/i386/kernel/vsyscall-int80.S中，入口名都是__kernel_vsyscall，这两个文件编译出的二进制代码由arch/i386/kernel/vsyscall.S所包含，并导出起始地址和结束地址

2.6内核在启动的时候，调用了新增的函数sysenter_setup（参见arch/i386/kernel/sysenter.c），在这个函数中，内核将虚拟内存空间的顶端一个固定地址页面（从0xffffe000开始到0xffffeffff的4k大小）映射到一个空闲的物理内存页面。然后通过之前执行CPUID的指令得到的数据，检测CPU是否支持sysenter/sysexit指令。如果CPU不支持，那么将采用INT调用方式的入口代码拷贝到这个页面中，然后返回。相反，如果CPU支持SYSETER/SYSEXIT指令，则将采用SYSENTER调用方式的入口代码拷贝到这个页面中

通过内核在启动时的设置，在每个进程的进程空间中，都能访问到内核所映射的这个代码页面， 这一部分包含在VDSO中， 即所看到的“linux-vdso.so.1”

####3.5 快速系统调用指令  

我们将 Intel 的 sysenter/sysexit 指令，AMD 的 SYSCALL/SYSRET 指令统称为"快速系统调用指令"。"快速系统调用指令"比起中断指令来说，其消耗时间必然会少一些，但是随着 CPU 设计的发展，将来应该不会再出现类似 Intel Pentium4 这样悬殊的差距。而"快速系统调用指令"比起中断方式的系统调用方式，还存在一定局限，例如无法在一个系统调用处理过程中再通过"快速系统调用指令"调用别的系统调用。因此，并不一定每个系统调用都需要通过"快速系统调用指令"来实现。比如，对于复杂的系统调用例如 fork，两种系统调用方式的时间差和系统调用本身运行消耗的时间来比，可以忽略不计，此处采取"快速系统调用指令"方式没有什么必要。而真正应该使用"快速系统调用指令"方式的，是那些本身运行时间很短，对时间精确性要求高的系统调用，例如 getuid、gettimeofday 等等。因此，采取灵活的手段，针对不同的系统调用采取不同的方式，才能得到最优化的性能和实现最完美的功能

