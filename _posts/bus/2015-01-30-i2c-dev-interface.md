---
layout: post
title: "i2c-dev接口"
description:
category: bus
tags: [bus, linux]
mathjax: 
chart:
comments: false
---

###1. i2c-dev  

通常，i2c设备由一个内核驱动程序来控制，但是在用户空间，还是能够直接访问某个i2c设备。因为， i2c-dev接口为我们导出了i2c总线驱动的操作接口， 位于linux内核i2c驱动目录下的 “i2c-dev.c” 文件

针对每一个I2C适配器，i2c-dev 生成一个主设备号为89的字符设备 “/dev/i2c-x”，它们的次设备号为该i2c总线的总线号（所有的256个次设备号全部可用），可以通过检查“/sys/class/i2c-dev”目录查看具体的i2c总线控制器所对应的总线号。也可以使用"i2c-tool"工具包中的 `i2cdetect -l`命令来获取系统中所有的i2c适配器的列表，注意，i2c总线的编号可能是动态分配的，有可能在系统重启后变为不同的值。  

linux kernel 中的i2c-dev.c中实现了“/dev/i2c-x”的文件操作接口

	static const struct file_operations i2cdev_fops = {
		.owner		= THIS_MODULE,
		.llseek		= no_llseek,
		.read		= i2cdev_read,
		.write		= i2cdev_write,
		.unlocked_ioctl	= i2cdev_ioctl,
		.open		= i2cdev_open,
		.release	= i2cdev_release,
	};

最主要的是 read()， write()， ioctl()方法，通过它们可以直接在usrespace进行i2c总线上的读写操作,   

**i2c-dev中的i2cdev-read(), i2cdev-write()有一定的局限性， 它不能够支持含有RepStart模式的时序， 但是，大多数稍微复杂一点的i2c设备都有可能使用RepStart，因此，i2c-dev的设备文件的read(), write()操作接口不具有通用性，这中情况下， 就要使用它的ioctl()接口了**
    
如果需要利用 i2c-dev 接口进行开发， 需要包含如下两个头文件
	
	#include<linux/i2c-dev.h>
	#include<linux/i2c.h>

**另外可以利用 i2c-tools 工具集源码中的 “i2c-dev.h”, "i2cbusses.h", "i2cbusses.c" 加快开发， 这些文件提供了一些封装号的功能**

###2. i2cdev read() 接口

i2cdev 的read接口的内核实现间接调用了 i2c_master_recv() 接口， 在打开 i2cdev 后， 使用ioctl设定要访问的i2c设备的地址， 然后调用read()即可完成读操作

	int addr = 0x0f;
	int fd = 0;
	int count = 0;
	char buf[128];

	/**/
	fd = open("/dev/i2c-x/", O_RDWR);

	/**/
	ioctl(fd, I2C_SLAVE, addr);

	/**/
	read(fd, buf, count);

read()接口只能支持单一的方向， 因此， 不能够支持含有RepStart模式的时序， 因此， i2cdev的 write()接口很少被用户程序使用

###3. i2cdev write() 接口

i2cdev 的write接口的内核实现间接调用了 i2c_master_send() 接口， 在打开i2cdev后， 使用ioctl设定要访问的i2c设备的地址， 然后调用write()即可完成读操作

	int addr = 0x0f;
	int fd = 0;
	int count = 0;
	char cmd[64] = {0x12, 0x43, 0x56};

	/**/
	fd = open("/dev/i2c-x/", O_RDWR);

	/**/
	ioctl(fd, I2C_SLAVE, addr);

	/**/
	write(fd, cmd, 3);

read()接口只能支持单一的方向， 因此， 不能够支持含有RepStart模式的时序, 因此， i2cdev的 write()接口很少被用户程序使用

###4. i2cdev ioctl()

i2cdev的ioctl()接口几乎能够完成所有的操作， 其支持的ioctl cmd如下：

	#define I2C_RETRIES		0x0701		/* 设置收不到ACK时的重试次数，默认为1次 */
	#define I2C_TIMEOUT		0x0702		/* 设置SMBbus的超时时限的jiffies */
	#define I2C_SLAVE			0x0703		/* 设置从机地址 */
	#define I2C_SLAVE_FORCE		0x0706		/* 强制设置从机地址 */
	#define I2C_TENBIT			0x0704		/* 选择地址位长:=0 for 7bit , != 0 for 10 bit */
	#define I2C_FUNCS			0x0705		/* 获取适配器支持的功能*/
	#define I2C_RDWR			0x0707		/* Combined R/W transfer (one STOP only) */
	#define I2C_PEC			0x0708		/* != 0 to use PEC with SMBus */
	#define I2C_SMBUS			0x0720		/* SMBus transfer */                

####4.1 I2C_SLAVE / I2C_SLAVE_FORCE

这两个ioctl cmd 都用于设置从机的地址， 区别是 I2C_SLAVE_FORCE 无论内核中是否已有驱动在使用这个地址都会成功，I2C_SLAVE 只在该地址空闲的情况下成功, 由于i2c-dev创建的i2c_client不加入i2c_adapter的client列表，所以不能防止其它线程使用同一地址，也不能防止驱动模块占用同一地址

####4.2 I2C_FUNCS

i2c总线和SMBus等总线兼容， 但是， 不是所有的i2c adapter都支持所有的特性，有些操作需要i2c adapter支持某些特性才能进行， 因此， 可以使用 I2C_FUNCS ioctl cmd来检查i2c adapter是否支持某些特性 (**使用`i2cdetect -F x`命令也可以列出某一i2c adapter支持的function**), 

	unsigned long funcs；
	int fd = 0;

	/**/
	fd = open("/dev/i2c-x/", O_RDWR);

	/**/
	ioctl(file, I2C_FUNCS, &funcs);
	

返回值的funcs的各bit含义如下：

	#define I2C_FUNC_I2C			0x00000001
	#define I2C_FUNC_10BIT_ADDR		0x00000002
	#define I2C_FUNC_PROTOCOL_MANGLING	0x00000004 /* I2C_M_IGNORE_NAK etc. */
	#define I2C_FUNC_SMBUS_PEC		0x00000008
	#define I2C_FUNC_NOSTART		0x00000010 /* I2C_M_NOSTART */
	#define I2C_FUNC_SMBUS_BLOCK_PROC_CALL	0x00008000 /* SMBus 2.0 */
	#define I2C_FUNC_SMBUS_QUICK		0x00010000
	#define I2C_FUNC_SMBUS_READ_BYTE	0x00020000
	#define I2C_FUNC_SMBUS_WRITE_BYTE	0x00040000
	#define I2C_FUNC_SMBUS_READ_BYTE_DATA	0x00080000
	#define I2C_FUNC_SMBUS_WRITE_BYTE_DATA	0x00100000
	#define I2C_FUNC_SMBUS_READ_WORD_DATA	0x00200000
	#define I2C_FUNC_SMBUS_WRITE_WORD_DATA	0x00400000
	#define I2C_FUNC_SMBUS_PROC_CALL	0x00800000
	#define I2C_FUNC_SMBUS_READ_BLOCK_DATA	0x01000000
	#define I2C_FUNC_SMBUS_WRITE_BLOCK_DATA 0x02000000
	#define I2C_FUNC_SMBUS_READ_I2C_BLOCK	0x04000000 /* I2C-like block xfer  */
	#define I2C_FUNC_SMBUS_WRITE_I2C_BLOCK	0x08000000 /* w/ 1-byte reg. addr. */

####4.3 I2C_RDWR

这一ioctl cmd用于发起连续的i2c传输操作，使用RepStart标记， 操作时， 指定 i2c_msg 数组及i2c_msg的数目， 然后发起该ioctl cmd， 即可进行来连续的传输

**需要i2c adapter 支持 I2C_FUNC_I2C 时这一ioctl cmd才有效**

使用 struct i2c_msg 来存储一次传输(一个方向的连续传输)的数据

	struct i2c_msg {
		__u16 addr;					/* slave address			*/
		__u16 flags;
			#define I2C_M_TEN			0x0010	/* this is a ten bit chip address */
			#define I2C_M_RD			0x0001	/* read data, from slave to master */
			#define I2C_M_STOP			0x8000	/* if I2C_FUNC_PROTOCOL_MANGLING */
			#define I2C_M_NOSTART		0x4000	/* if I2C_FUNC_NOSTART */
			#define I2C_M_REV_DIR_ADDR		0x2000	/* if I2C_FUNC_PROTOCOL_MANGLING */
			#define I2C_M_IGNORE_NAK		0x1000	/* if I2C_FUNC_PROTOCOL_MANGLING */
			#define I2C_M_NO_RD_ACK		0x0800	/* if I2C_FUNC_PROTOCOL_MANGLING */
			#define I2C_M_RECV_LEN		0x0400	/* length will be first received byte */
		__u16 len;					/* msg length				*/
		__u8 *buf;					/* pointer to msg data			*/
	};

使用 struct i2c_rdwr_ioctl_data 来组织一次复合传输(使用RepStart，可改变传输方向， 只有一个stop信号)的所有 i2c_msg

	struct i2c_rdwr_ioctl_data {
		struct i2c_msg *msgs;	/* i2c_msg */
		int nmsgs;		/* i2c_msg 的数目*/
	};

例如

	_u8 _buf[] = {0x20, 0x00, 0x01};
	_u8 write_buf[16] = {0};

	struct i2c_msg[2] = msgs{
		{
			.addr = 0x0f,
			.flags = 0,
			.len = 3,
			.buf = write_buf,
		}

		{
			.addr = 0x0f,
			.flags = 0 | I2C_M_RD,
			.len = 6,
			.buf = read_buf,
	};

	struct i2c_rdwr_ioctl_data ioctl_data = {
		.msgs = msgs,
		.nmsgs = 2,
	};

	/**/
	fd = open("/dev/i2c-x/", O_RDWR);

	/**/
	ioctl(fd, I2C_SLAVE, addr);

	/**/
	ioctl(fd, I2C_RDWR, &ioctl_data);

以上的示例完成了一次i2c复合操作， 先写入3byte数据， 再读取6byte数据

####4.4 I2C_SMBUS

I2C_RDWR 用于完成 i2c plain data 操作， 而I2C_SMBUS则可以用于完成 SMBus 命令集操作， 使用来保存SMBus命令操作的数据

	struct i2c_smbus_ioctl_data {
		char read_write;			/* 0:read   1:write*/
		__u8 command;			
		int size;				/* SMBus传输的类型 */
		union i2c_smbus_data *data;		/* SMBus传输的数据 */
	};

	union i2c_smbus_data {
		__u8 byte;
		__u16 word;
		__u8 block[I2C_SMBUS_BLOCK_MAX + 2]; /* block[0] is used for length */
                                                /* and one more for PEC */
	};

SMBus的传输类型(即 i2c_smbus_ioctl_data.size 的取值)有：

	#define I2C_SMBUS_QUICK         	0
	#define I2C_SMBUS_BYTE          	1
	#define I2C_SMBUS_BYTE_DATA     	2 
	#define I2C_SMBUS_WORD_DATA     	3
	#define I2C_SMBUS_PROC_CALL     	4
	#define I2C_SMBUS_BLOCK_DATA        	5
	#define I2C_SMBUS_I2C_BLOCK_BROKEN  	6
	#define I2C_SMBUS_BLOCK_PROC_CALL   	7       /* SMBus 2.0 */
	#define I2C_SMBUS_I2C_BLOCK_DATA    	8

例如， i2c-tools 工具集的源码中，i2c_smbus_access() 展示了 I2C_SMBUS ioctl cmd 的用法

	__s32 i2c_smbus_access(int file, char read_write, __u8 command,
                                     int size, union i2c_smbus_data *data)
	{
		struct i2c_smbus_ioctl_data args;

		args.read_write = read_write;
		args.command = command;
		args.size = size;
		args.data = data;
		return ioctl(file,I2C_SMBUS,&args);
	}

另外 i2c-tools 中的“i2c-dev.h”封装了 I2C_SMBUS ioctl cmd, 提供了如下接口用于发起不同传输类型的SMBus命令操作

	__s32 i2c_smbus_access(int file, char read_write, __u8 command, int size, union i2c_smbus_data *data)
	__s32 i2c_smbus_write_quick(int file, __u8 value)
	__s32 i2c_smbus_read_byte(int file)
	__s32 i2c_smbus_write_byte(int file, __u8 value)
	__s32 i2c_smbus_read_byte_data(int file, __u8 command)
	__s32 i2c_smbus_write_byte_data(int file, __u8 command, __u8 value)
	__s32 i2c_smbus_read_word_data(int file, __u8 command)
	__s32 i2c_smbus_write_word_data(int file, __u8 command, __u16 value)
	__s32 i2c_smbus_process_call(int file, __u8 command, __u16 value)
	__s32 i2c_smbus_read_block_data(int file, __u8 command, __u8 *values)
	__s32 i2c_smbus_write_block_data(int file, __u8 command, __u8 length, const __u8 *values)
	__s32 i2c_smbus_read_i2c_block_data(int file, __u8 command, __u8 length, __u8 *values)
	__s32 i2c_smbus_write_i2c_block_data(int file, __u8 command, __u8 length, const __u8 *values)
	__s32 i2c_smbus_block_process_call(int file, __u8 command, __u8 length, __u8 *values)

可以利用这些接口加快开发

###5. 参考

关于i2c-dev的接口的使用可以参考著名的 i2c-tools 工具集的源码

