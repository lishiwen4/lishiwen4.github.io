---
layout: post
title: "android native crash"
description:
category: android
tags: [android, debug]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. deadbaad

android中有时候遇到native crash， 在log中显示“fault addr deadbaad”， 例如

	19:15:37.209   565     0 I Kernel  : <6>[ 1389.481323] system_server[920]: unhandled level 1 translation fault (11) at 0xdeadbaad, esr 0x92000045
	19:15:37.214   565     0 I Kernel  : <1>[ 1389.481335] pgd = ffffffc06462a000
	19:15:37.214   565     0 I Kernel  : <1>[ 1389.483694] [deadbaad] *pgd=0000000000000000
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487944] 
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487955] CPU: 2 PID: 920 Comm: system_server Tainted: P           O 3.10.49-g94a866e-00766-gf33a0a5 #1
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487962] task: ffffffc062391800 ti: ffffffc060824000 task.ti: ffffffc060824000
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487971] PC is at 0x7f7a74f108
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487976] LR is at 0x7f7a74f100
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487982] pc : [<0000007f7a74f108>] lr : [<0000007f7a74f100>] pstate: 60000000
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487987] sp : 0000007fdfee1730
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487991] x29: 0000007fdfee1730 x28: 0000000000000001 
	19:15:37.214   565     0 I Kernel  : <6>[ 1389.487999] x27: 0000000000000001 x26: 0000000000000002 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488006] x25: 0000005581a29ed0 x24: 0000000000000002 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488013] x23: 0000000000000000 x22: 0000007f7a7afe90 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488021] x21: 0000005581cc59b0 x20: 0000007f7a7af000 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488028] x19: 0000005581cc59a0 x18: 0000000000000000 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488035] x17: 0000000000000001 x16: 0000007fdfee1730 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488043] x15: 000c4dd53dd5b24d x14: 0000000000000000 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488050] x13: 0000000000000000 x12: 0000000000000001 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488057] x11: a1eeb98161ff9c24 x10: 7f7f7f7f7f7f7f7f 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488064] x9 : 6471656b631f6e73 x8 : 7f7f7f7f7f7f7f7f 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488072] x7 : 0000000000000010 x6 : 0000000000000000 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488079] x5 : 00000000deadbaad x4 : 00000000139f0460 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488086] x3 : 0000000000000000 x2 : 0000000000000000 
	19:15:37.215   565     0 I Kernel  : <6>[ 1389.488093] x1 : 0000007f7a7ed8f0 x0 : a1eeb98161ff9c24 

事实上， 给出的出错地址“deadbaad”是一个特殊的地址， 在android的bionic c库中， 当free或者realloc存在问题时(一般是发生了HEAP CORRUPTION)， 会触发USAGE_ERROR_ACTION

	#define USAGE_ERROR_ACTION(m,p) __bionic_heap_usage_error(__FUNCTION__, p)

	static void __bionic_heap_usage_error(const char* function, void* address) {
  		__libc_fatal_no_abort("invalid address or address of corrupt block %p passed to %s", address, function);

  		// So that debuggerd gives us a memory dump around the specific address.
  		// TODO: improve the debuggerd protocol so we can tell it to dump an address when we abort.
  		*((int**) 0xdeadbaad) = (int*) address;
	}

由于虚拟地址 0xdeadbaad 通常并未被映射， 因此会触发一个页表转换错误， 打印出如上log， 这时，0xdeadbaad这一地址并无意义， 需要根据其它的信息来debug
