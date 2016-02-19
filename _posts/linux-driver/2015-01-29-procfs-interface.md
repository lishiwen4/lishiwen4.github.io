---
layout: post
title: "使用proc文件系统导出信息"
description:
category: linux-driver
tags: [linux, driver]
mathjax: 
chart:
comments: false
---

###1. 创建 /proc  节点 
  
+ struct proc_dir_entry  
  
内核使用 proc_dir_entry 结构体来描述 /proc 文件系统中的一个目录或者文件节点  
  
	struct proc_dir_entry {
		unsigned int low_ino;
		umode_t mode;
		nlink_t nlink;
		kuid_t uid;
		kgid_t gid;
		loff_t size;
		const struct inode_operations *proc_iops;
		const struct file_operations *proc_fops;
		struct proc_dir_entry *next, *parent, *subdir;
		void *data;
		atomic_t count;		/* use count */
		atomic_t in_use;	/* number of callers into module in progress; */
				/* negative -> it's going away RSN */
		struct completion *pde_unload_completion;
		struct list_head pde_openers;	/* who did ->open, but not ->release */
		spinlock_t pde_unload_lock; /* proc_fops checks and pde_users bumps */
		u8 namelen;
		char name[];
	};
  
+ proc_create  
  
内核提供了一套API用于在 proc 文件系统中创建 文件节点/目录：  
  
	static struct proc_dir_entry *proc_create(  const char *name, umode_t mode,
                                                struct proc_dir_entry *parent, 
					const struct file_operations *proc_fops);	
                                                
	@ name： 节点名
	@ mode：	权限位
	@ parent：父目录
	@ proc_fops: 文件操作结构体
    
    	struct proc_dir_entry *proc_mkdir(const char *name, struct proc_dir_entry *parent);

	@ name： 目录名
	@ parent：父目录
    
+ remove_proc_entry()  
  
内核提供 remove_proc_entry 来移除 proc 文件系统中的 文件/目录 项：  
  
	void remove_proc_entry(const char *name, struct proc_dir_entry *parent)；

	@ name： 文件/目录 名
	@ parent：父目录
    
###2. 使用 file_operations 实现 proc 文件读写  导向内核信息  
  
在调用 proc_create（） 创建文件项时， 提供了一个 file_operations结构体， 实现里面的read， write方法即可实现proc 文件读写， demo 程序如下：  
  
	#include <linux/init.h>
	#include <linux/module.h>
	#include <linux/kernel.h>
	#include <asm/uaccess.h>
	#include <asm/stat.h>
	#include <linux/proc_fs.h>

	MODULE_LICENSE("GPL");

	#define LOG_TAG	"my_test1: "

	static char my_buf[256];

	//====================================================================================
	static ssize_t my_proc_read(struct file *fp, char __user *buf, size_t size, loff_t *pos)
	{
		int len = strlen(my_buf);
	
		pr_info(LOG_TAG "enter %s()\n", __func__);

		if( 0 == *pos )	{
			strncpy(buf, my_buf, sizeof(my_buf));
			*pos += len;
			return len;
		} else
			return 0;
	}

	static ssize_t my_proc_write(struct file *file,  const char __user *buf, size_t size, loff_t *pos)
	{
		int len = size < sizeof(my_buf) ? size : sizeof(my_buf);

		pr_info(LOG_TAG "enter %s()\n", __func__);

		if( 0 == *pos)	{
			memset(my_buf, '\0', sizeof(my_buf));
			len = copy_from_user(my_buf, buf, len);
			*pos += len;
			my_buf[len - 1] = '\0';
			return size;
		}else 
			return 0;
	}


	static struct file_operations my_file_ops = {
		.owner 	= THIS_MODULE,
		.read 	= my_proc_read,
		.write 	= my_proc_write,
	};

	//====================================================================================
	static int __init my_module_init(void)
	{
		struct proc_dir_entry * my_proc_node = NULL;

		pr_info(LOG_TAG "enter %s()\n", __func__);

		my_proc_node = proc_create("my_test1", S_IRUGO | S_IWUGO, NULL, &my_file_ops);
		if( NULL == my_proc_node )	{
			pr_err(LOG_TAG "creat /proc/my_test1 failed\n");
			remove_proc_entry("my_test1", NULL);
			return -1;
		}

		pr_info(LOG_TAG "creat /proc/my_test1 success\n");
		return 0;
	}

	static void __exit my_module_exit(void)
	{
		pr_info(LOG_TAG "enter %s()\n", __func__);

		pr_info(LOG_TAG "remove /proc/my_test1\n");
		remove_proc_entry("my_test1", NULL);
	}

	module_init(my_module_init);
	module_exit(my_module_exit);
     
     
###3. 使用 seq_file 实现 proc 文件的读取  
  
seq_file只是在普通的文件read中加入了内核缓冲的功能，从而实现顺序多次遍历，读取大数据量的简单接口。内核使用 seq_file 结构体来描述seq_file:  
	
	struct seq_file {
		char *buf;
		size_t size;
		size_t from;
		size_t count;
		loff_t index;
		loff_t read_pos;
		u64 version;
		struct mutex lock;
		const struct seq_operations *op;
		int poll_event;
	#ifdef CONFIG_USER_NS
		struct user_namespace *user_ns;
	#endif
		void *private;
	};  
     
seq_file一般只提供只读接口，在使用seq_read时，主要靠下述四个操作来完成内核自定义缓冲区的遍历的输出操作，其中pos作为遍历的iterator，在seq_read函数中被多次使用，用以定位当前从内核自定义链表（不止是链表， 还可以是哈希表， 数组， 红黑树）中读取的当前位置，当多次读取时，pos非常重要，且pos总是遵循从0,1,2...end+1遍历的次序，其即必须作为遍历内核自定义链表的下标，也可以作为返回内容的标识。但是我在使用中仅仅将其作为返回内容的标示，并没有将其作为遍历链表的下标，从而导致返回数据量大时造成莫名奇妙的错误，注意：start返回的void*v如果非0，被show输出后，在作为参数传递给next函数，next可以对其修改，也可以忽略；当next或者start返回NULL时，在seq_open中控制路径到达seq_end。

	struct seq_operations {
    	void * (*start) (struct seq_file *m, loff_t *pos);
    	void (*stop) (struct seq_file *m, void *v);
    	void * (*next) (struct seq_file *m, void *v, loff_t *pos);
    	int (*show) (struct seq_file *m, void *v);
	};

在读取 proc file 时：  
  
1. 如果只有一条记录， 调用过程是： start->show->stop, start->stop.  
2. 如果有多条记录并且能通过缓冲区一次传第完， 调用过程是，继续 start->show->next->show->next-> ... ->nex->show->stop, start->stop .  
3. 如果有多条记录， 比且不能一次通过缓冲区传送完成， 调用过程是： start->show->next->show->...->show->stop, start->show->next->show->...->show->stop, ... , start->stop.  
  
seq_file 的缓冲区通常为1K， 但是会动态增长以便能容纳至少一条记录（当单条记录的大小大于1页的时候，这个就很有用）， 通常， 应该在start中进行必要的加锁， 在stop中进行必要的解锁一边避免竟态。  
  
当使用seq_read时， 需要实现seq_operations, 并且将file和seq_file关联起来（通常在open操作中进行）， 同时， seq_file也提供了游离的 seq_lseek, seq_release用于支持llseek和release操作。  
  
使用seq_file 的demo如下：  
  
	#include <linux/init.h>
	#include <linux/module.h>
	#include <linux/kernel.h>
	#include <asm/uaccess.h>
	#include <asm/stat.h>
	#include <linux/proc_fs.h>
	#include <linux/seq_file.h>

	#define LOG_TAG	"my_test2: "

	MODULE_LICENSE("GPL");

	static char my_buf[][32] = {	{"i am the 0th record"},	//buf[0]
					{"i am the 1th record"},	//buf[1]
					{"i am the 2th record"},	//buf[2]
					{"i am the 3th record"},	//buf[3]
					{"i am the 4th record"},	//buf[4]
	};

	//==========================================================================
	static void* my_seq_start (struct seq_file *m, loff_t *pos)
	{
		pr_info(LOG_TAG "enter %s()\n", __func__);

		if( *pos < 0 || *pos > 4 /*my_buf[][32] index*/)	{
			pr_err(LOG_TAG "invalid pos %ld\n", (long)*pos);
			return NULL;
		}else	
			return my_buf[*pos];
	}

	static void my_seq_stop (struct seq_file *m, void *v)
	{
		pr_info(LOG_TAG "enter %s()\n", __func__);
		pr_info(LOG_TAG "do nothing\n");
	}

	static void* my_seq_next (struct seq_file *m, void *v, loff_t *pos)
	{
		pr_info(LOG_TAG "enter %s()\n", __func__);

		if( ++(*pos) < 0  || *pos > 4 /*my_buf[][32] index*/)	{
			pr_info(LOG_TAG "end\n");
			return NULL;
		}else
			return my_buf[*pos];
	}

	static int my_seq_show (struct seq_file *m, void *v)
	{
		seq_printf(m, "%s\n", (char*)v);
		return 0;	
	}

	static struct seq_operations my_seq_ops = {
		.start	=	my_seq_start,
		.stop	=	my_seq_stop,
		.next	=	my_seq_next,
		.show	=	my_seq_show,
  	};
	//==========================================================================

	static int my_proc_open(struct inode * inode , struct file * file) 
	{ 
        return seq_open(file, &my_seq_ops ); 
	}

	static struct file_operations my_file_ops = {
		.owner 	 = THIS_MODULE,
		.open	 = my_proc_open,
		.read 	 = seq_read,
		.llseek  = seq_lseek,
		.release = seq_release,
	};

	//==========================================================================
	static int __init my_module_init(void)
	{
		struct proc_dir_entry * my_proc_node = NULL;

		pr_info(LOG_TAG "enter %s()\n", __func__);

		my_proc_node = proc_create("my_test2", S_IRUGO, NULL, &my_file_ops);
		if( NULL == my_proc_node )	{
			pr_err(LOG_TAG "creat /proc/my_test2 failed\n");
			remove_proc_entry("my_test2", NULL);
			return -1;
		}

		pr_info(LOG_TAG "creat /proc/my_test2 success\n");
		return 0;
	}

	static void __exit my_module_exit(void)
	{
		pr_info(LOG_TAG "enter %s()\n", __func__);

		pr_info(LOG_TAG "remove /proc/my_test2\n");
		remove_proc_entry("my_test2", NULL);
	}

	module_init(my_module_init);
	module_exit(my_module_exit);	
