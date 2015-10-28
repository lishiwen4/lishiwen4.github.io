---
layout: post
title: "gcc attribute 扩展"
description:
category: linux
tags: [linux, build-link]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. gcc attribute 机制  
  
atrribute 机制是gcc 的一大特色， 可以为函数， 变量和类型设定不同的属性  
  
###2. attribute的使用  
  
gcc attribute的语法为  
  
	__attribute__ ((attribute-list))
例如

	int var __attribute__ ((__unused__))

注意：  
  
+ 放置在声明结尾的 “;” 之前，最少两对外括号，不能省    
+ 对于属性名， 可以在前后都加上双下划线（如 unused和 \__unused\__ 相同）， 防止和源码中定义的宏名冲突  
+ 注意， attribute应用c/c++的声明语句里面， 而不是定义语句里面  
+ 一次使用多个属性可重复使用 \__attribute\__ 宏
+ 有些属性会在编译时给出警告，但这需要打开 –Wall 选项
  
按照 attribute 语句作用的对象不同， 可以分为3类

+ 函数属性
+ 变量属性
+ 类型属性

###3. 函数属性  

####3.1 \__attribute\__ (( alias("target") ))

alias 属性用于定义一个函数的弱别名  ， 例如
  
	void __f () { /* Do something. */; }
	void f() __attribute__ ((weak, alias ("__f")));
        
f() 是 _f()的一个弱别名， 弱别名意味着， 如果有一个外部的强符号f(), 将会覆盖这个符号， 有些机器上的实现不支持此属性。  
  
####3.2 \__attribute\__ (( always_inline ))

always_inline 属性用于指定函数内联, 一般情况下， 只有当开启了编译的优化选项时， 函数才可能内连，这一选项可以指定函数总是内连  
  
####3.3 \__attribute\__ (( cdecl ))

cdecl 属性用于在i386上指定使用 cdecl调用协定（调用者负责清理堆栈）  
  
####3.4 \__attribute\__ (( const ))

const 参数用于修饰那些输出值只依赖于输入值(即只要输入参数相同则输出相同)的函数传值参数，当开启编译优化选项后， 若使用相同参数调用该函数多次，则会被优化为调用一次， 这类函数通常使用数值型参， 因此使用指针参数， 全局变量或者静态变量的函数就不能使用该属性修饰

####3.5 \__attribute\__ (( constructor ))

constructor 属性指定该函数在 main()运行前自动被调用， 类似于 c++的全局对象的构造函数在main()前运行 
  
####3.6 \__attribute\__ (( destructor ))	

destructor 属性与constructor 属性相反， 指定函数在main()返回后后自动被调用
  
####3.7 \__attribute\__ (( deprecated ))	

deprecated 属性声明该函数已经被废弃， 源码中有该属性的函数若被使用，则在编译时都会给出警告， 例如
  
	int old_fn () __attribute__ ((deprecated));
	int old_fn ();
	int (*fn_ptr)() = old_fn;
        
对第三行会给出警告  
  
####3.8 \__attribute\__ (( dllexport ))

dllexport 属性用于在microsoft windows 和 symbian OS上用于导出函数到dll
  
####3.9 \__attribute\__ (( dllimport ))

dllimport 属性用于在microsoft windows 和 symbian OS上用于从dll导入函数
  
####3.10 \__attribute\__ (( fastcall	))

fastcall属性用于在i386上指定使用fastcall调用协定  
  
前两个参数放入 ECX 和 EDX， 其它参数压栈，如果是可变参数则全部压栈。被调用者负责清理堆栈。  
  
####3.11 \__attribute\__ (( format )) 

format 属性用于指定当像printf scanf 等一样使用可变参数时， 对参数进行检查  
  
	__attribute__((format(printf,m,n)))
	__attribute__((format(scanf,m,n)))
	__attribute__((format(strftime,m,n)))
	__attribute__((format(strfmon,m,n)))
       
其中， m指定格式化字符串是第几个参数， n指定...中地一个参数对应第几个型参（注意在c++的成员函数中要计算隐含的this）  
  
####3.12 \__attribute\__ (( format_arg ))

format_arg 属性用于指出该函数可以返回一个 format string (类似于printf， scanf 等的format string)

如果打开了编译选项 “-Wformat-nonliteral”， 则编译器会检查 printf， scanf等的 format string， 如果不是一个const字符串(例如， 调用一个函数返回字符串指针), 则会给出警告， 使用 format_arg 属性声明的函数， 可以被调用产生format setring， 且不会被编译器警告

####3.13 \__attribute\__ (( interrupt ))

interrupt 属性用于在 ARM, AVR, C4x, M32R/D and Xstormy16指定函数为中断处理例程，编译器会产生合适的进入退出代码  

####3.14 \__attribute\__ (( long_call/short_call ))

long_call 属性指明在ARM平台上调用函数使用long_call, 即首先将将函数的地址装载到寄存器中， 然后使用寄存器的内容进行调用

short_call 属性指明在ARM平台上调用函数使用short_call， 即将目标函数和调用点的地址偏移量作为 `bl` 指令的参数

####3.15 \__attribute\__ (( noinline ))

noinline 属性声明该函数不能被作为内联函数使用

####3.16 \__attribute\__ (( nonnull(arg-index, ...) ))

nonull 属性用于标明指定的参数不能为空指针， 例如

	extern void * my_memcpy (void *dest, const void *src, size_t len) __attribute__((nonnull (1, 2))); 

则编译器会检查 dest 和 src 参数， 如果不指定index， 则会检查所有的指针参数

如果打开了编译选项 "-Wnonnull", 且编译器判定某一个声明了 nonull 属性的函数的某个指针参数为空， 则编译器会给出警告

####3.17 \__attribute\__ (( noreturn	 )) 

noreturn属性 指示该函数没有返回值， 不产生警告  
  
一些标准的C库函数， 例如 abort 和 exit， 当在你自己的函数里使用这些函数时， 可以使用这一属性提醒编译器  
  
	void fatal () __attribute__ ((noreturn));
          
	void fatal (/* ... */)
	{
		/* ... */ /* Print error message. */ /* ... */
		exit (1);
	}
    
####3.18 \__attribute\__ (( nothrow	)) 

nothrow 属性提示编译器在函数里不能抛出异常  
  
####3.19 \__attribute\__ (( pure ))	

pure 属性告诉编译器，该函数除了返回值，不会对外部产生影响（如全局变量，指针），可用于优化  
  
####3.20 \__attribute\__ (( section ("section-name") ))

section 属性指定该函数被放置到目标文件中的哪一个section中  
  
通常函数会被放置到 .text 节中，但是你可以使用section属性来指定放到一个新的section中， 有些文件格式不支持section属性， 你需要借助连接器来实现目的。  
  
####3.21 \__attribute\__ (( stdcall	))

stdcall 属性用于指定在i386上使用stdcall调用协定  
  
####3.22 \__attribute\__ (( unused ))

unused 属性用于指定在该函数未被使用时， 编译器不要给出警告  
  
####3.23 \__attribute\__ (( used ))

used 属性用于告诉编译器， 这一段代码有用， 即使是在没有调用它时也不要给出警告， 因为有可能是在内联汇编中调用它

####3.24 \__attribute\__ (( warn_unused_result ))	

warn_unused_result 属性用于指定在该函数返回值未被使用时， 编译器不要给出警告  
  
####3.25 \__attribute\__ (( weak ))	

weak 属性用于将函数定义为弱符号
  
通常情况下，对于外部函数或者变量， 连接时作为强引用符号，当不能被决议时（比如未被定义），连接将会报错， 而如果弱符号不能被决议， 则其将被当作值为0或者一个特殊值。 若符号能够被强符号所覆盖， 这在提供库函数时相当有用， 库函数可以对某些扩展功能模块的引用声明为弱引用，当我们将扩展模块与程序链接在一起时，功能模块就可以正常使用；如果我们去掉某些功能模块，
那么程序也可以正常链接，只是缺少了相应的功能，这使得程序的功能更加容易裁剪和组合。  
  
linux内核里面定义了一些宏来使用函数属性  
  
	# define __inline__          __attribute__((always_inline))
	# define __deprecated           __attribute__((deprecated))
	# define __attribute_used__     __attribute__((__used__))
	# define __attribute_const__     __attribute__((__const__))
	# define __must_check            __attribute__((warn_unused_result))
  
###4. 变量属性  
  
####4.1 \__attribute\__ (( aligned (alignment) ))

alignment 属性用于指定对齐  
  
	int x __attribute__ ((aligned (16))) = 0;
	struct foo { 
		int x[2] __attribute__ ((aligned (8))); 
	};

若不指定对齐的字节数， 则编译器使用目标平台上的最大字节对齐

	short array[3] __attribute__ ((aligned));
        
####4.2 \__attribute\__ (( deprecated ))

deprecated 属性用于指出该变量已经被废弃， 当变量被使用时， 给出警告， 例如
  
	extern int old_var __attribute__ ((deprecated));
	extern int old_var;
	int new_fn () { return old_var; }
  
第三行将给处警告  
  
####4.3 \__attribute\__ (( packed ))	

packed 属性要求变量或者结构体域使用最小的对齐（变量为字节对齐，位域为bit对齐）  
  
	struct foo
	{
		char a;
		int x[2] __attribute__ ((packed));
	};

则 a 和 x 之间不需要padding 
  
####4.4 \__attribute\__ ((section ("section-name") )

section 属性指定全局变量放在某一个section
  
	struct duart a __attribute__ ((section ("DUART_A"))) = { 0 };
	struct duart b __attribute__ ((section ("DUART_B"))) = { 0 };
	char stack[10000] __attribute__ ((section ("STACK"))) = { 0 };
	int init_data __attribute__ ((section ("INITDATA"))) = 0;
          
	main()
	{
		/* Initialize stack pointer */
		init_sp (stack + sizeof (stack));
          
		/* Initialize initialized data */
		memcpy (&init_data, &data, &edata - &data);
          
		/* Turn on the serial ports */
		init_duart (&a);
		init_duart (&b);
	}

####4.5 \__attribute\__ (( unused ))

unused 属性用于声明在变量未被使用时， 不给出警告  
  
####4.6 \__attribute\__ (( weak ))	

weak 属性用于将变量定义为弱符号， 详见函数属性中的weak属性  
  
####4.7 \__attribute\__ (( dllexport ))

dllexport 属性用于在microsoft windows 和 symbian OS上用于导出变量到dll
  
####4.8 \__attribute\__ (( dllimport ))

dllimport 属性用于在microsoft windows 和 symbian OS上用于从dll导入变量

###5. 类型属性  
  
####5.1 \__attribute\__ (( aligned (alignment) ))

aligned 属性指定该类型的最小对齐  
  
	struct S { short f[3]; } __attribute__ ((aligned (8)));
	typedef int more_aligned_int __attribute__ ((aligned (8)));
          
即指定 S 和 more_aligned_int这两种数据类型8byte对齐

####5.2 \__attribute\__ (( packed ))

packed 属性 指定该类型（struct， union）内部使用最小对齐  
  
	struct my_unpacked_struct
	{
		char c;
		int i;
	};
          
	struct my_packed_struct __attribute__ ((__packed__))
	{
		char c;
		int  i;
		struct my_unpacked_struct s;
	};
        
注意， my_packed_struct的内部的 c，i，s 使用最小对齐， 但是s的内部没有使用最小对齐  
  
####5.3 \__attribute\__ (( unused ))

unused 属性用于声明当该类型的变量未被使用时， 不给出警告  
  
####5.4 \__attribute\__ (( deprecated )) 

deprecated 属性用于声明该变量类型已经被废弃， 当该类型的变量在源码中任何地方被使用时，编译器将给出警告  
    
###6. 更多的说明  
  
详细文档见 [gcc atrribute][http://www.unixwiz.net/techtips/gnu-c-attributes.html]
