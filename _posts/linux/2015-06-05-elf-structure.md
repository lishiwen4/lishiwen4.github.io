---
layout: post
title: "ELF 文件结构"
description:
category: linux
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. elf

ELF(Executable and Linking Format)是一种对象文件的格式，用于定义不同类型的对象文件(Object files)中都放了什么东西、以及都以什么样的格式去放这些东西。它自最早在 System V 系统上出现后，被 *NIX 世界所广泛接受，作为缺省的二进制文件格式来使用。可以说，ELF是构成众多*NIX系统的基础之一

TISC(Tool Interface Standard Committee)委员会定义了一套ELF标准, 前后共有两个标准：v1.1和v1.2， 两个版本的内容差别不大

ELF定义的是对象文件的格式， 所谓对象文件(Object files)有三个种类：

**1. 可重定位的对象文件(Relocatable file)** ： 这是由汇编器汇编生成的 .o 文件，链接器会使用一个或一些 Relocatable object files 作为输入，经链接处理后，生成一个可执行的对象文件 (Executable file) 或者一个可被共享的对象文件(Shared object file)。我们可以使用 ar 工具将众多的 Relocatable object files 归档(archive)成 .a 静态库文件. 另外,linux内核可加载模块 .ko 文件也是 Relocatable object file。
**2. 可执行的对象文件(Executable file)** ： vim, gdb 等都是Executable object file
**3. 可被共享的对象文件(Shared object file)** ： 即所谓的动态库文件，也即 .so 文件。如果拿前面的静态库来生成可执行程序，那每个生成的可执行程序中都会有一份库代码的拷贝。如果在磁盘中存储这些可执行程序，那就会占用额外的磁盘空间；另外如果拿它们放到Linux系统上一起运行，也会浪费掉宝贵的物理内存， 使用动态库可以在各进程间共享，节省内存空间

可以使用“file”命令来查看一个ELF对象文件的类别, 例如

	$ file hello.o:     
	ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped

	$ file hello.out
	ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.2.5, dynamically linked (uses shared libs), not stripped

	$file libm.so
	libsub.so: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), not stripped
 
**file命令其实是依靠ELF文件头中的信息来识别ELF对象文件的类别**

###2. ELF对象文件的结构

先来看图， ELF文件结构的两种视图：
  
![elf file view](/images/linux/elf-view.png)

+ ELF header ：  ELF header在文件开始处描述了整个文件的组织, 在每一种ELF对象文件中都有ELF header
+ Section Header Table ： 存放了各个section的信息， Section Header Table是可选的， ELF header中存放了Section Header Table的起始位置， 每一项的大小， 多少项等信息
+ Program Header Table ： 存放了各个segment的信息， Program Header Table是可选的， ELF header中存放了program Header Table的起始位置， 每一项的大小， 多少项等信息
+ section ： 描述的是链接器所需要的信息，连接器需要.text, .rel.text, .data, .rodata等信息做地址的重定位， 因此，可重定位和可共享文件中都有section信息， 可执行文件中可以没有section信息
+ segment ： 描述的是loader所需要的的信息， 装载器只需要知道Read/Write/Execute的属性，可重定位的对象文件通常没有segment信息， 而可执行的对象文件和可被共享的对象文件一定有segment信息， 并且在这两种ELF对象文件中， 一个segment通常对应一到多个section

binutils中提供了一系列的工具用于解读ELF文件， 其中最常用的是readelf工具， 后面将会使用这一tool辅助解读ELF文件

####2.1 ELF定义的数据结构

1. Elf32_Addr
   + size : 4byte
   + align : 4byte
   + purpose : Unsigned program address
2. Elf32_Half 
   + size : 2byte 
   + align : 2byte 
   + purpose : Unsigned medium integer
3. Elf32_Off 
   + size : 4byte 
   + align : 4byte 
   + purpose : Unsigned file offset
4. Elf32_Sword 
   + size : 4byte 
   + align : 4byte 
   + purpose : Signed large integer
5. Elf32_Word 
   + size : 4byte 
   + size : 4byte 
   + purpose : Unsigned large integer
6. unsigned char 
   + size : 1byte 
   + align : 1byte 
   + purpose : Unsigned small integer



####2.2 ELF header

ELF header的定义

	#define EI_NIDENT 16

	typedef struct {
		unsigned char e_ident[EI_NIDENT];
		Elf32_Half e_e_type;
		Elf32_Half e_machine;
		Elf32_Word e_version;
		Elf32_Addr e_entry;
		Elf32_Off e_phoff;
		Elf32_Off e_shoff;
		Elf32_Word e_flags;
		Elf32_Half e_ehsize;
		Elf32_Half e_phentsize;
		Elf32_Half e_phnum;
		Elf32_Half e_shentsize;
		Elf32_Half e_shnum;
		Elf32_Half e_shstrndx;
	};

+ e_ident ： ELF的magic number
   1. 4byte 固定为“_ELF”
   2. 1byte 标识class(EI_CLASS)， 1代表32bit， 2代表64bit， 0代表非法
   3. 1byte 标识大小端(EI_DATA)， 1代表小端字节序， 2代表大端字节序
   4. 1byte 标识当前版本(EI_VERSION)， 当前为1
   5. 9byte 0字节填充(EI_PAD )
+ e_type ： 文件类型 
   1. 0代表invalid(ET_NONE )
   2. 1代表Relocatable file(ET_REL)
   3. 2代表Executable file(ET_EXEC )
   4. 3代表Shared object file(ET_DYN)
   5. 4代表Core file(ET_CORE)
   6. 0xff00代表Processor-specific(ET_LOPROC)
   7. 0xffff代表Processor-specific(ET_HIPROC)
+ e_machine : 目标体系结构
   1. 0 代表No machine(ET_NONE)
   2. 7 代表intel 8086(EM_860)
+ e_version : 目标文件版本， 当前为1
+ e_entry ： 程序入口地址
+ e_phoff ： Program Header Table的文件内偏移
+ e_shoff ： Section Header Table的文件内偏移
+ e_flags ： 未使用
+ e_ehsize ： ELF头大小
+ e_phentsize ： Program Header Table中每条目的大小
+ e_phnum ： Program Header Table中条目个数
+ e_shentsize ： Section Header Table中每条目的大小
+ e_shnum ： Section Header Table中条目个数
+ e_shstrndx ： 指存放各section名称的表(也是一个section， 通常名为.shstrtab)在section header table中的条目的索引

使用”readelf“来读取一个x86_64平台上的debug版程序“test”的ELF header， 输出如下：

	$ readelf -h test
	ELF Header:
	  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
	  Class:                             ELF64
	  Data:                              2's complement, little endian
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - System V
	  ABI Version:                       0
	  Type:                              EXEC (Executable file)
	  Machine:                           Advanced Micro Devices X86-64
	  Version:                           0x1
	  Entry point address:               0x400410
	  Start of program headers:          64 (bytes into file)
	  Start of section headers:          5208 (bytes into file)
	  Flags:                             0x0
	  Size of this header:               64 (bytes)
	  Size of program headers:           56 (bytes)
	  Number of program headers:         9
	  Size of section headers:           64 (bytes)
	  Number of section headers:         36
	  Section header string table index: 33

####2.3 Section Header

Section包含目标文件除了ELF文件头、程序头表、section头表的所有信息，而且目标文件section满足几个条件：

+ 目标文件中的每个section都只有一个section头项描述，可以存在不指示任何section的section头项。
+ 每个section在文件中占据一块连续的空间
+ Section之间不可重叠

section header的结构定义如下：

	typedef struct {
		Elf32_Word sh_name;
		Elf32_Word sh_type;
		Elf32_Word sh_flags;
		Elf32_Addr sh_addr;
		Elf32_Off sh_offset;
		Elf32_Word sh_size;
		Elf32_Word sh_link;
		Elf32_Word sh_info;
		Elf32_Word sh_addralign;
		Elf32_Word sh_entsize;
	} Elf32_Shdr;

+ sh_name : 指明section名;存储的是section header string表(通常为.shstrtab)中的索引， 通过该索引寻找字符串， **事实上，section的名字对编译器和链接器来说是有意义的， 但是操作系统来说是没有意义的， 操作系统根据section的typr和flag来决定如何处理section** 
+ sh_type : section的类型
   + 0 : SHT_NULL， 代表该section头是非活动的它没有相关联的section, 该section头的其他成员的值都是未定义的
   + 1 : SHT_PROGBITS 代表程序定义的数据或代码, 数据的格式和意义由程序自己解释， 一个ELF对象文件中可以有多个该类section
   + 2 : SHT_SYMTAB 代表符号表, 该类section中保留了一个完整的符号表， 可以在静态链接时使用， 也可已被动态连接时使用，一个ELF文件中只能有一个该类型的section， 另外参见SHT_DYNSYM  
   + 3 : SHT_STRTAB 代表字符串表，一个ELF对象文件可以有多个SHT_STRTAB类型的文件以保存多个字符串表
   + 4 : SHT_RELA 代表重定位表
   + 5 : SHT_HASH 代表符号哈希表， 所有参与动态连接的ELF对象文件必须包含一个符号哈希表。当前，一个ELF对象文件只能有一个符号哈希表
   + 6 : SHT_DYNAMIC 代表动态链接信息，一个ELF对象文件只能有一个该类的section
   + 7 : SHT_NOTE 代表文件的note信息
   + 8 : SHT_NOBITS 代表未初始化数据,在文件内不占用空间
   + 9 : SHT_REL 代表
   + 10 : SHT_SHLIB 代表 保留的section ，包含这个类型的section的程序是不符合ABI的
   + 11 : SHT_DYNSYM 在SHT_SYMTAB类型的section中，保存了完整的符号表， 但是很多符号在动态链接时并不是必须的， 因此SHT_DYNSYM类型的section中保存了动态链接所需的符号的最小集合, 一个ELF对象文件中只能有一个此类的section
   + 0x70000000 ～ 0x7fffffff  : SHT_LOPROC ~ SHT_HIPROC 代表范围之间的值为特定处理器自定义保留的
   + 0x80000000 ~ 0xffffffff : SHT_LOUSER ~ SHT_HIUSER  代表范围之间的值为应用程序自定义保留的
+ sh_flags ： section的标记
   + 1 : SHF_WRITE 代表该section包含的数据在进程执行期间可写
   + 2 : SHF_ALLOC 代表该section包含的数据在进程执行期间占据着内存。对于一些控制section没有驻留在目标文件的内存映象中；这些sections的该属性是关闭的
   + 4 : SHF_EXECINSTR 代表该section包含了可执行的机器指令
   + 0xf0000000 : SHF_MASKPROC 这个掩码中包括的所有的位是为特定处理器语意保留的
+ sh_addr : 若section将被载入内存， 则指定了将要被载入到的虚拟地址
+ sh_offset ： Section的raw数据的文件偏移;注意SHT_NOBIT 类型的section没有raw数据,它将由操作系统初始化
+ sh_size ： Section大小， 即使SHT_NOBITS section未占任何文件空间,也可能非零
+ sh_link ： 连接到其他section头的索引， 对于不同类型的section， 这一项意义不同
   + SHT_DYNAMIC ： sh_link值代表该section所使用的字符串表在section header table中的索引
   + SHT_HASH ： sh_link值代表该section所使用的符号表在section header table中的索引
   + SHT_REL ： sh_link值代表该section所关联的符号表在section header table中的索引
   + SHT_RELA  ： 同上
   + SHT_SYMTAB ： 由操作系统来解释
   + SHT_DYNSYM  ： 同上
   + other ： 无意义， 值为 SHN_UNDEF 
+ sh_info ： 
   + SHT_REL ： sh_link值代表重定位应用到的section在section header table中的索引
   + SHT_RELA ： sh_link值代表重定位应用到的section在section header table中的索引
   + SHT_SYMTAB ： 由操作系统来解释
   + SHT_DYNSYM  ： 同上
   + other ： 无意义， 值为 0
+ sh_addralign ： 对齐要求
   + sh_entsize ： 包含固定大小条目的section的条目大小,如符号表

依旧以“test”应用程序为例， 使用readelf， 其中几个sction的信息如下

	$ readelf -S test
	[Nr] Name              Type             Address           Offset	Size              EntSize          Flags  Link  Info  Align
	[ 1] .interp           PROGBITS         0000000000400238  00000238	000000000000001c  0000000000000000   A       0     0     1
	[ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298	000000000000001c  0000000000000000   A       5     0     8
  	[ 5] .dynsym           DYNSYM           00000000004002b8  000002b8	0000000000000060  0000000000000018   A       6     1     8
	[11] .init             PROGBITS         00000000004003c8  000003c8	0000000000000018  0000000000000000  AX       0     0     4
	[12] .plt              PROGBITS         00000000004003e0  000003e0	0000000000000030  0000000000000010  AX       0     0     16
  	[13] .text             PROGBITS         0000000000400410  00000410	00000000000001f8  0000000000000000  AX       0     0     16
  	[14] .fini             PROGBITS         0000000000400608  00000608	000000000000000e  0000000000000000  AX       0     0     4
	[15] .rodata           PROGBITS         0000000000400618  00000618	0000000000000016  0000000000000000   A       0     0     4
	[24] .data             PROGBITS         0000000000601010  00001010	0000000000000010  0000000000000000  WA       0     0     8
	[25] .bss              NOBITS           0000000000601020  00001020	0000000000000018  0000000000000000  WA       0     0     8
	[33] .shstrtab         STRTAB           0000000000000000  0000130d	0000000000000149  0000000000000000           0     0     1
  	[34] .symtab           SYMTAB           0000000000000000  00001d58	00000000000006c0  0000000000000018          35    53     8
	[35] .strtab           STRTAB           0000000000000000  00002418	00000000000001fe  0000000000000000           0     0     1

####2.4 系统保留的section

有一些系统保留的section， 相关信息如下：

	name		sh_type			sh_flag
	.bss		SHT_NOBITS		SHF_ALLOC + SHF_WRITE
	.comment		SHT_PROGBITS		none
	.data		SHT_PROGBITS		SHF_ALLOC + SHF_WRITE
	.data1		SHT_PROGBITS		SHF_ALLOC + SHF_WRITE
	.debug		SHT_PROGBITS		none
	.dynamic		SHT_DYNAMIC		SHF_ALLOC + SHF_WRITE
	.hash		SHT_HASH			SHF_ALLOC
	.line		SHT_PROGBITS		none
	.note		SHT_NOTE			none
	.rodata		SHT_PROGBITS		SHF_ALLOC
	.rodata1		SHT_PROGBITS		SHF_ALLOC
	.shstrtab		SHT_STRTAB		none
	.strtab		SHT_STRTAB		如果有可装载的段必须要用到字符串表，则有SHF_ALLOC标志
	.symtab		SHT_SYMTAB		如果有可装载的段必须要用到字符串表，则有SHF_ALLOC标志
	.text		SHT_PROGBITS		SHF_ALLOC + SHF_EXECINSTR

####2.5 符号表

每一个ELF对象文件都有一个对应的符号表， 一个符号表就是一个section， 通常为“.symtab”和“.dynsym”， 里面的每一个条目都存储了符号的信息

	typedef struct
	{
  		Elf32_Word    st_name;                /* Symbol name (string tbl index) */
  		Elf32_Addr    st_value;               /* Symbol value */
  		Elf32_Word    st_size;                /* Symbol size */
  		unsigned char st_info;                /* Symbol type and binding */
  		unsigned char st_other;               /* Symbol visibility */
  		Elf32_Section st_shndx;               /* Section index */
	} Elf32_Sym;      

+ st_name : 符号名， 存储的是对应的字符串表中的下标
+ st_value : 符号
+ st_size : 符号的大小， 对于包含数据的符号， 保存的是数据类型的大小， 比如 double型大小为8Byte
+ st_info : 低4位为符号的类型： 
   + 符号类型：
      0. STB_LOCAL(局部符号， 对目标文件外部不可见)
      1. STB_GLOBAL(全局符号， 外部可见) 
      2. STB_WEAK(弱引用)
   + 符号的类型
      0. STT_NOTYPE(未知符号类型)
      1. STT_OBJECT(该符号是一个数据对象， 比如变量， 数组)
      2. STT_FUNC(该符号是个函数或者可执行代码)
      3. STT_SECTION(该符号标识一个section， 这种符号的绑定类型必须为STB_LOCAL)
      4. STT_FILE (该符号标识文件名， 一般都是目标文件对应的源文件名， 这种符号的绑定类型必须为STB_LOCAL, 且st_shndx成员必须为SHN_ABS)
+ st_other :为0, 未使用
+ st_shndx : 符号所在的section在section header table 中的index， 对于一些特殊符号， 有一些特殊的值
   + 0xfff1 : SHN_ABS，表示该符号包含了一个绝对的值， 比如表示文件名的符号
   + 0xfff2 : SHN_COMMON 表示该符号是一个"COMMON"块类型的符号
   + 0 : SHN_UNDEF 表示该符号在本目标文件中引用到， 但未被定义， 可能在其它目标文件中定义

使用“readelf -s”可以读取目标文件的符号表

	$ readelf -s test

	Symbol table '.dynsym' contains 4 entries:
   		Num:    Value          Size Type    Bind   Vis      Ndx Name
     		  0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     		  1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     		  2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     		  3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__

	Symbol table '.symtab' contains 74 entries:
   		Num:    Value          Size Type    Bind   Vis      Ndx Name
     		  0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     		  1: 0000000000400238     0 SECTION LOCAL  DEFAULT    1 
     		  2: 0000000000400254     0 SECTION LOCAL  DEFAULT    2 
     		  3: 0000000000400274     0 SECTION LOCAL  DEFAULT    3 
     		  4: 0000000000400298     0 SECTION LOCAL  DEFAULT    4 
     		  5: 00000000004002b8     0 SECTION LOCAL  DEFAULT    5 
     		  6: 0000000000400318     0 SECTION LOCAL  DEFAULT    6 
     		  7: 0000000000400356     0 SECTION LOCAL  DEFAULT    7 
     		  8: 0000000000400360     0 SECTION LOCAL  DEFAULT    8 
     		  9: 0000000000400380     0 SECTION LOCAL  DEFAULT    9 
    		  10: 0000000000400398     0 SECTION LOCAL  DEFAULT   10 
    		  11: 00000000004003c8     0 SECTION LOCAL  DEFAULT   11 
		  ......
    		  33: 000000000040043c     0 FUNC    LOCAL  DEFAULT   13 call_gmon_start
    		  34: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    		  35: 0000000000600e28     0 OBJECT  LOCAL  DEFAULT   18 __CTOR_LIST__
    		  36: 0000000000600e38     0 OBJECT  LOCAL  DEFAULT   19 __DTOR_LIST__
    		  37: 0000000000600e48     0 OBJECT  LOCAL  DEFAULT   20 __JCR_LIST__
    		  38: 0000000000400460     0 FUNC    LOCAL  DEFAULT   13 __do_global_dtors_aux
    		  39: 0000000000601028     1 OBJECT  LOCAL  DEFAULT   25 completed.6531
    		  40: 0000000000601030     8 OBJECT  LOCAL  DEFAULT   25 dtor_idx.6533
    		  41: 00000000004004d0     0 FUNC    LOCAL  DEFAULT   13 frame_dummy
    		  42: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    		  43: 0000000000600e30     0 OBJECT  LOCAL  DEFAULT   18 __CTOR_END__
    		  44: 0000000000400748     0 OBJECT  LOCAL  DEFAULT   17 __FRAME_END__
    		  45: 0000000000600e48     0 OBJECT  LOCAL  DEFAULT   20 __JCR_END__
    		  46: 00000000004005f0     0 FUNC    LOCAL  DEFAULT   13 __do_global_ctors_aux
    		  47: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test1.c
    		  48: 0000000000601038     4 OBJECT  LOCAL  DEFAULT   25 a.2046
    		  49: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test2.c
    		  50: 0000000000600e24     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_end
    		  51: 0000000000600e50     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
    		  52: 0000000000600e24     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_start
    		  53: 0000000000600fe8     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    		  54: 00000000004005e0     2 FUNC    GLOBAL DEFAULT   13 __libc_csu_fini
    		  55: 0000000000601010     0 NOTYPE  WEAK   DEFAULT   24 data_start
    		  56: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
    		  57: 0000000000601024     0 NOTYPE  GLOBAL DEFAULT  ABS _edata
    		  58: 0000000000400628     0 FUNC    GLOBAL DEFAULT   14 _fini
    		  59: 0000000000601020     4 OBJECT  GLOBAL DEFAULT   24 global
    		  60: 0000000000600e40     0 OBJECT  GLOBAL HIDDEN    19 __DTOR_END__
    		  61: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    		  62: 0000000000400538    16 FUNC    GLOBAL DEFAULT   13 hello
    		  63: 0000000000601010     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    		  64: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    		  65: 0000000000601018     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
    		  66: 0000000000400638     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
    		  67: 0000000000400550   137 FUNC    GLOBAL DEFAULT   13 __libc_csu_init
    		  68: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT  ABS _end
    		  69: 0000000000400410     0 FUNC    GLOBAL DEFAULT   13 _start
    		  70: 0000000000601024     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start
    		  71: 00000000004004f4    68 FUNC    GLOBAL DEFAULT   13 main
    		  72: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
    		  73: 00000000004003c8     0 FUNC    GLOBAL DEFAULT   11 _init

在使用连接器链接ELF对象文件的时候，链接器ld会生成一些特殊的符号， 并且可以在自己的程序中声明并使用这些符号， 不同的平台上可能有所差异， 具体要参照ld的链接脚本， 在linux上， ld的链接脚本的默认存储路径为“/user/lib/ldscripts”， 可执行文件的链接脚本的后缀".x"

几个特殊符号如下：

1. __executeable_start : 程序的起始地址， 注意不是入口地址， 通常为“_start”的地址
2. _end ： 程序的结束地址

####2.6 重定位表

在编译时，某一模块中可能会引用外部定义的全局的变量或者函数， 但是此时还不知道这些符号将要被分配的绝对地址，因此在编译阶段将这些符号的地址使用0来代替， 等到链接的阶段(静态链接阶段或者动态链接阶段)再来修正这些符号的地址

例如对于全局变量“global”

	extern int global;
	int main()	
	{
		......
		global = 10;
		......
	}

在重定位前的汇编代码如下

		global = 10;
  	1f:	c7 05 00 00 00 00 0a 	movl   $0xa,0x0(%rip)        # 29 <main+0x29>
  	26:	00 00 00 

而重定位之后， 汇编代码如下

	global = 10;
  	400513:	c7 05 03 0b 20 00 0a 	movl   $0xa,0x200b03(%rip)        # 601020 <global>
  	40051a:	00 00 00 

可以看到，在链接的过程中， 会修改的指令中的地址部分(即全局变量global的地址)， 那么链接器怎么知道哪些位置需要修改呢， 每一个需要重定位的符号都由一条重定位条目，保存在重定位表中
重定位条目的结构如下：

	typedef struct
	{
  		Elf32_Addr    r_offset;
  		Elf32_Word    r_info;
	} Elf32_Rel;

	typedef struct
	{
  		Elf32_Addr    r_offset;
  		Elf32_Word    r_info;
  		Elf32_Sword   r_addend;
	}Elf32_Rela;

+ r_offset ： 需重定位的地址在section中的偏移地址
+ r_info : 高24位是符号表的索引， 低8位表示重定位的类型， 不同体系的机器上， 定义不同(因为重地位需要修改指令的地址， 但是不同体系的机器上， 地址的格式和使用方式都不相同， 例如 Intel x86的jmp指令有11种寻址模式， call指令有10种寻址模式， 而mov指令则有34种寻址模式)， Intel x86上的ELF文件的常见两种类型为： R_386_32(32位绝对寻址修正), R_386_PC32(32位相对寻址修正)
+ r_addend ： 存放一个常量， 在地址修正的过程中使用

**每一个包含需要重定位的符号的section都有一个一个对应的重定位表， 例如“.data”对应“.rela.data”， “。text”对应“.rela.text”**

可以看到， 每一个一个重定位表关联连个section:

1. 符号表 通过section header的sh_info成员来索引
2. 对应的需要重定位的section 通过section header的sh_link成员来索引

"readelf -r" 可用于读取ELF对象文件的重定位表, 以如下代码为例

	extern int global;
	int main()	
	{
		......
		global = 10;
		......
	}

	$ readelf -r test1.o
	  Offset          Info           Type           Sym. Value    Sym. Name + Addend
	......
	000000000021  001100000002 R_X86_64_PC32     0000000000000000 global - 8
	.....

offset说明要修正的偏移量为0x21， 来看重定位前和重定位后的汇编代码

		global = 10;
  	1f:	c7 05 00 00 00 00 0a 	movl   $0xa,0x0(%rip)        # 29 <main+0x29>

		global = 10;
  	400513:	c7 05 03 0b 20 00 0a 	movl   $0xa,0x200b03(%rip)        # 601020 <global>	

可以看到， 重定位后修正了从0x21处开始的4个字节

####2.7 .got表

可执行文件文件可以确定自己在进程的虚拟地址空间中的起始位置， 因为可执行文件往往是第一个被加载的文件， 可以选择一个固定的地址， 例如intel x86 linux上一般为 0x080480000， windows 下一般为 0x0040000， 但是对于共享对象则不一样， 他们可能在不同的进程中被加载到不同的位置， 那么共享对象中的绝对地址访问就会存在问题， 对于数据部分来说， 因为对于每一个使用共享对象的进程， 都会有一个单独的副本， 因此可以在加载时重定位， 但是， 对于代码部分， 因为所有进程都共享同一份副本， 因此， 若在装载时重定位， 势必会影响其它的进程， 若是为每一个进程保留一份共享对象的代码副本， 则会失去节省内存的优势

为了解决这一问题， 必须使用“地址无关”代码(Position-independent code)， 即程序不论被加载到哪一块虚拟地址， 不需要修改指令部分就能使用， 实现的方式是将指令中寻要修改的部分剥离出来， 和数据部分放在一起， 在每一个进程中都有一份副本

GCC提供了一个“-fPIC”选项， 用于生成地址无关的代码

在使用共享库时， 共享库的模块内部的数据访问和函数调用，因为相对位置是固定的， 都可以使用相对寻址， 因此可以不用进行重定位， 需要解决的问题是：

1. 模块间的函数调用
2. 模块间的全局变量

模块间的数据访问和函数调用， 要等到模块被装载时才能决定，ELF在数据段中建立一个指向这些变量的指针数组， 也被称为全局偏移表(Global Offset Table), 即名为“.got”的section
在装载模块时， 链接器会查找每一个符号的地址， 然后填充 .got 中的每一项， 以确保每一个指针的地址正确， .got存放在数据段中， 在每一个进程中都有一个独立的副本，因为在模块编译时， 可以确定 .got 相对于当前指令的偏移地址， 而 .got 中哪一项对应哪一个符号是在编译时就确定的， 在访问时， 通过 .got 中的地址进行访问

ELF共享库在编译时， 默认把所有定义在模块内的全局变量当作定义在其它模块的全局变量， 使用 .got 表访问， 当共享模块被加载时， 如果某个全局变量在可执行文件中存在副本， 则动态连接器会将 .got 中的相应地址指向该副本， 若全局变量在共享模块中被初始化， 则还需要将该初始化值复制到可执行文件中的副本中去

####2.8 .plt表

动态链接时对于全局和静态的数据以及模块间的调用都要进行复杂的got定位， 然后再进行间接跳转， 另外在程序看是执行前， 进行符号的查找和重定位， 这些工作都会减慢程序的启动/运行速度  
  
程序开始执行前， 耗费大量的时间解决模块间的函数引用的符号查找和重定位， 但是在一个程序的运行过程中， 可能有大量的条件分支不被执行到， 这也意味这在程序的运行过程中， 大量的函数可能没有被调用到， 因此， 在开始执行前 就把所有的函数都链接好实际上是一种浪费， ELF采用一种称为“延迟绑定”的做法， 当函数第一次被调用到时才由动态链接器进行符号的查找和重定位， 如果没有用到， 则不进行符号查找和重定位

PLT(Procedulre Linkage Table)即为延迟绑定表，PLT是一个section，通常名为“.plt”， 引入plt技术之后， .got表被分离为 .got 和 .got.plt两个表，其中， .got用于保存全局变量的引用地址， 而.got.plt则保存了函数引用的地址， 并且.got.plt表中前3项有特殊意义：

1. 第一项保存了 .dynamic 的地址
2. 第二项保存了 本模块的ID值
3. 第三项保存了 _dl_runtime_resolve()的地址

上一小节说到过， 模块间的符号引用使用使用 .got 表来间接跳转， 而 .plt 则在.got表的基础上又添加了一次跳转

以puts()函数的调用为例(x64机器上)

	#include <stdio.h>

	int main()
	{
		puts("plt test");
		return 0;
	}

反汇编代码如下：

	Disassembly of section .plt:

	00000000004003e0 <puts@plt-0x10>:
  	4003e0:	ff 35 0a 0c 20 00    	pushq  0x200c0a(%rip)        # 600ff0 <_GLOBAL_OFFSET_TABLE_+0x8>
  	4003e6:	ff 25 0c 0c 20 00    	jmpq   *0x200c0c(%rip)        # 600ff8 <_GLOBAL_OFFSET_TABLE_+0x10>
  	4003ec:	0f 1f 40 00          	nopl   0x0(%rax)


	00000000004003f0 <puts@plt>:
	4003f0:	ff 25 0a 0c 20 00    	jmpq   *0x200c0a(%rip)        # 601000 <_GLOBAL_OFFSET_TABLE_+0x18>
  	4003f6:	68 00 00 00 00       	pushq  $0x0
	4003fb:	e9 e0 ff ff ff       	jmpq   4003e0 <_init+0x18>

	Disassembly of section .text:

	00000000004004f4 <main>:
	......
		puts("plt test");
  	4004f8:	bf fc 05 40 00       	mov    $0x4005fc,%edi
  	4004fd:	e8 ee fe ff ff       	callq  4003f0 <puts@plt>

1. 在不使用 .plt时， callq 指令后的地址应该是 .got表中 puts() 所对应的项的地址， 在使用 .plt 表的情况下， callq指令后的地址是 puts()对应的puts@plt项的地址
2. puts@plt中， 第一个jmpq的地址部分初始值指定为 0x601000 (0x200c0a + 0x4003f0 = 0x601000), 即puts()对应的.got表项， 但是其中的内容(即地址)在编译时被初始化为0x4003f6, 即puts@plt中第二条指令的地址， 当第一次调用puts()时， 会依次执行这3条命令
3. puts@plt中第三条指令调用， 跳转到puts@plt-0x10， 先将 .got表的第2项(本模块的ID)压栈， 然后跳转到 .got表的第3项(_dl_runtime_resolve函数的地址), 完成puts()的重定位工作
4. 完成重定位之后, .got表中， puts()对应的地址将会被修正，后续再次调用puts()时， puts@plt中第一条指令直接跳转到 puts()函数真正的位置开始执行， 然后返回， 不再执行puts@plt中的其它指令

使用gdb观察.got表的延迟绑定

	$ gdb plt-test
	(gdb) list
	1	#include <stdio.h>
	2	
	3	int main()
	4	{
	5		puts("plt test");
	6		return 0;
	7	}
	(gdb) b 5
	Breakpoint 1 at 0x4004f8: file plt-test.c, line 5.
	(gdb) b 6
	Breakpoint 2 at 0x400502: file plt-test.c, line 6.
	(gdb) x /xg 0x601000
	0x601000 <puts@got.plt>:	0x00000000004003f6
	(gdb) r
	Starting program: /home/sven/src/test/plt-test 
	warning: no loadable sections found in added symbol-file system-supplied DSO at 0x7ffff7ffa000

	Breakpoint 1, main () at plt-test.c:5
	5		puts("plt test");
	(gdb) n
	plt test

	Breakpoint 2, main () at plt-test.c:6
	6		return 0;
	(gdb) x /xg 0x601000
	0x601000 <puts@got.plt>:	0x00007ffff7a8b990
	(gdb) p puts
	$1 = {<text variable, no debug info>} 0x7ffff7a8b990 <puts>

结合上面的汇编代码， 可以看到到， 地址0x601000处的8个byte(即.got表中puts对应的项)， 值为0x00000000004003f6，调用puts()后， 被修正为0x00007ffff7a8b990， 和打印出的puts()的地址值一致

####2.9 .interp 

动态链接器的位置由ELF来指定， 在动态链接的ELF文件中， 有一个名为“.interp”的section用于保存所需的动态链接器的路径, 可通过如下的方式查看

	$ readelf -p .interp exe_file

在linux桌面上， 动态链接器的路径为“/lib/ld-linux.so.2”, 64位 android上通常为“/system/bin/linker64”

linxu上的动态链接器比较特殊， 它是一个共享的ELF对象文件， 但是它也可以被直接执行(linux的系统调用execve()并不关心ELF文件是否是可执行的类型， 只是简单地按照program header table中的描述对ELF文件进行加载， 然后将控制权转交给ELF的入口或者动态连接器的入口) 
	
####2.10 .dynamic

动态链接ELF中最重要的结构是 .dynamic section， 保存了动态链接器所需要的基本信息， 比如， 需要依赖哪些共享对象， 动态链接的符号表和重定位表的位置等

.dynamic的结构数组为

	typedef struct
	{
  		Elf32_Sword   d_tag;                  /* Dynamic entry type */
  		union
    		{
      			Elf32_Word d_val;                 /* Integer value */
      			Elf32_Addr d_ptr;                 /* Address value */
    		} d_un;
	} Elf32_Dyn;         

由一个类型值或者附加的数值或者指针组成, 类型值取值如下：

+ DT_SYMTAB : 动态连接符号表(.dynsym)的地址
+ DT_STRTAB : 动态链接字符串表(.dynstr)的地址
+ DT_STRSZ : 动态链接字符串表的大小
+ DT_HASH : 动态链接hash表的地址
+ DT_SONAME : 本共享对象的name
+ DT_RPATH : 动态链接共享对象的搜索路径
+ DT_INIT : 初始化代码地址
+ DT_FINIT : 结束代码地址
+ DT_NEED : 依赖的共享对象文件
+ DT_REL : 动态链接重定位表的地址
+ DT_RELA : 同上
+ DT_RELENT : 动态链接重定位表条目数
+ DT_RELAENT : 同上
......

.dynamic有点像ELF的文件头， 只不过ELF文件头保存的是静态链接的相关内容， 而.dymanic保存的是动态链接的相关内容

使用“readelf -d”可以读取.dynamic的内容

	$ readelf -d test
	Dynamic section at offset 0xe50 contains 20 entries:
	  Tag        Type                         Name/Value
	0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 	0x000000000000000c (INIT)               0x4003c8
 	0x000000000000000d (FINI)               0x400628
 	0x000000006ffffef5 (GNU_HASH)           0x400298
 	0x0000000000000005 (STRTAB)             0x400318
 	0x0000000000000006 (SYMTAB)             0x4002b8
 	0x000000000000000a (STRSZ)              61 (bytes)
 	0x000000000000000b (SYMENT)             24 (bytes)
 	0x0000000000000015 (DEBUG)              0x0
 	0x0000000000000003 (PLTGOT)             0x600fe8
 	0x0000000000000002 (PLTRELSZ)           48 (bytes)
 	0x0000000000000014 (PLTREL)             RELA
 	0x0000000000000017 (JMPREL)             0x400398
 	0x0000000000000007 (RELA)               0x400380
 	0x0000000000000008 (RELASZ)             24 (bytes)
 	0x0000000000000009 (RELAENT)            24 (bytes)
 	0x000000006ffffffe (VERNEED)            0x400360
 	0x000000006fffffff (VERNEEDNUM)         1
 	0x000000006ffffff0 (VERSYM)             0x400356
 	0x0000000000000000 (NULL)               0x0

###3. Program Header

目标文件或者共享文件的program header table描述了系统执行一个程序所需要的段或者其它信息。目标文件的一个段（segment）包含一个或者多个section。Program header只对可执行文件和共享目标文件有意义，对于程序的链接没有任何意义。结构定义如下：

	typedef struct elf32_phdr{  
		Elf32_Word    p_type;   
		Elf32_Off     p_offset;  
		Elf32_Addr    p_vaddr;        /* virtual address */  
		Elf32_Addr    p_paddr;        /* ignore */  
		Elf32_Word    p_filesz;       /* segment size in file */  
		Elf32_Word    p_memsz;        /* size in memory */  
		Elf32_Word    p_flags;  
		Elf32_Word    p_align;       
	} Elf32_Phdr;  

+ p_type : segement的类型
   + 0 ： PT_NULL  未使用，该segment header其他的成员值都是未定义的， 忽略segement header
   + 1 ： PT_LOAD  该类型指segment header定一个可载入的段，由 p_filesz 和 p_memsz 描述。文件中的字节被映射到内存段的开始处。如果该段的内存大小（ p_memsz ）比文件大小（ p_filesz ）要大，则多出的字节被定义保持为 0
   + 2 ： PT_DYNAMIC 该类型segment header指定动态链接信息 
   + 3 ： PT_INTERP 该类型segment header以一个null结尾的字符串指定动态链接器的路径， 仅对可执行文件有意义， 一个文件只能出现一次
   + 4 ： PT_NOTE  该类型segment header指定辅助信息的位置和大小
   + 5 ： PT_SHLIB 该类型segment header保留且具有未指定的语义
   + 6 ： PT_PHDR 该类型segment header指定了程序头表本身(既在文件中又在该程序的内存映像中)的位置和大小
   + 0x70000000 ： PT_LOPROC 为处理器保留
   + 0x7fffffff ： PT_HIPROC 为处理器保留
+ p_offset : 给出了该segement在文件中相对于该object文件开始处的偏移
+ p_vaddr : 该segment加载到内存中的虚拟地址
+ p_paddr : 为该segment的物理地址而保留的, linux上未使用
+ p_filesz : 该segement在文件中的字节数， 可以为0
+ p_memsz : 该segement在内存中占用的字节数， 可以为0
+ p_flags : 对于可以装载进内存的segement， 给出了和该段相关的标志， 可取或
   + 1 ： PF_X  可执行
   + 2 ： PF_W  可写
   + 4 ： PF_R  可读
   + 0xf0000000 ： PF_MASKPROC 未定义
+ p_align : 给出 了该segment在内存和文件中的对齐值。 0 和 1 表示不需要对齐。否则，p_align 必须为2 的正整数次幂，，并且p_vaddr和p_offset分别以p_align取模计算的结果值应该相等

使用“readelf -l” 可以读取 Programing header信息以及segment和section的映射关系

	$readelf -l test
	Elf file type is EXEC (Executable file)
	Entry point 0x400410
	There are 9 program headers, starting at offset 64

	Program Headers:
  		Type           Offset             VirtAddr           PhysAddr		FileSiz            MemSiz              Flags  Align
  		PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040	0x00000000000001f8 0x00000000000001f8  R E    8
  		INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238	0x000000000000001c 0x000000000000001c  R      1
      			[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  		LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000	0x000000000000074c 0x000000000000074c  R E    200000
  		LOAD           0x0000000000000e28 0x0000000000600e28 0x0000000000600e28	0x00000000000001fc 0x0000000000000218  RW     200000
  		DYNAMIC        0x0000000000000e50 0x0000000000600e50 0x0000000000600e50	0x0000000000000190 0x0000000000000190  RW     8
  		NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254	0x0000000000000044 0x0000000000000044  R      4
  		GNU_EH_FRAME   0x0000000000000650 0x0000000000400650 0x0000000000400650	0x0000000000000034 0x0000000000000034  R      4
  		GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000	0x0000000000000000 0x0000000000000000  RW     8
  		GNU_RELRO      0x0000000000000e28 0x0000000000600e28 0x0000000000600e28	0x00000000000001d8 0x00000000000001d8  R      1 
 	Section to Segment mapping:
  	Segment Sections...
   	00     
   	01     .interp 
   	02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   	03     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss 
   	04     .dynamic 
   	05     .note.ABI-tag .note.gnu.build-id 
   	06     .eh_frame_hdr 
   	07     
   	08     .ctors .dtors .jcr .dynamic .got 

###END