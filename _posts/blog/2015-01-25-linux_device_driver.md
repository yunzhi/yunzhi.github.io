---
layout: post
title: linux设备驱动相关
description: 记录下linux设备驱动的相关基础知识
category: blog
---

## LDD3

这部分记录下我在读《LINUX 设备驱动程序》（第三版）（LDD3）时的一些笔记。

### 第一章

1. 在编写驱动程序时，程序员应该特别注意下面这个基本概念：编写访问硬件的内核代码时，不要给用户强加任何特定策略。因为不同的用户有不同的需求，驱动程序应该处理如何时硬件可用的问题，而将怎样使用硬件的问题留给上层应用程序。

2. UNIX系统支持多进程并发的运行，每个进程都请求系统资源，比如运算，内存，网络连接或其他一些资源等。内核负责处理所有这些请求，根据内核完成任务的不同，可以将内核功能分成如下几个部分：

	1）. 进程管理：进程管理负责创建和销毁进程，并处理它们和外部世界之间的连接（输入输出）。不同进程之间的通信（通过信号、管道或者进程间通信原语）是整个系统的基本功能，因此也由内核处理。除此之外，控制进程如何共享CPU调度器也是进程管理的一部分。概括来说，内核进程管理活动是在单个或者多个CPU上实现了多个进程的抽象
	
	2）. 内存管理：内存是计算机的主要资源之一，用来管理内存的策略是决定系统性能的一个关键因素。内核在有限的可用资源之上为每个进程都创建了一个虚拟地址空间。内核的不同部分在和内存管理子系统交互时使用一组函数调用，包括简单的malloc/free函数队对及其他一些复杂的函数
	
	3.） 文件系统：UNIX在很大程度上依赖文件系统的概念，Unix中得每个对象几乎都可以当做文件来看待。内核在没有结构的硬件上构造结构化的文件系统，而文件抽象在整个系统中广泛使用。另外，Linux支持多文件系统类型，也就是在物理介质上组织数据的不同方式。例如，磁盘可以格式化为符合Linux标准的ext3文件系统，也可以格式化为常用的FAT文件系统或者其他种类
	
	4）. 设备控制：几乎每一个系统操作最终都会映射到物理设备上。除了处理器，内存以及其他很有限的几个对象外，所有设备控制操作都由被控制设备相关的代码来完成，这段代码就叫做驱动程序。内核必须为系统中的每件外设嵌入相应的驱动程序，这包括硬件驱动器，键盘等。
	
	5）. 网络功能：网络功能也必须有操作系统来管理，因为大部分网络操作和具体进程无关：数据报的传入是异步事件。在整个进程处理这些数据包之前必须收集、标识和分发这些数据包。系统负责在应用程序和网络接口之间传递数据包，并根据网络活动控制程序的执行。另外，所有路由器和地址的解析都有内核处理。

3. 内核和网络设备驱动程序的通信，完全不同于内核和字符设备以及块设备驱动程序之间的通信，内核调用一套和数据包相关的函数而不是read,write等。

4. 任何从内核得到的内存，都必须在提供给用户进程或者设备之前清零或者以其他方式初始化，否则就可能发生信息泄露（如数据和密码的泄露等）。

### 第二章

1. 每个内核模块的编程方式和事件驱动型有点类似。事件驱动的应用程序和内核代码之间的另一个主要区别是：应用程序在退出时，可以不管资源的释放或者其他清除工作，单模块的退出函数却必须仔细撤销初始化函数所做的一切，否则，在系统重新引导之前某些东西就会残留在系统中

2. 每当应用程序执行系统调用或者被硬件中断挂起时，Unix将执行模式从用户空间切换到内核空间。执行系统调用的内核代码运行在进程上下文中，它代表调用进程执行操作，因此能够访问进程地址空间的所有数据。而处理硬件中断的内核代码和进程是异步的，与任何一个特定进程无关。

3. 并发问题：并发进程；中断；内核定时器，异步运行；SMP，对称多处理器系统；可抢占内核

4. 内核模块可通过访问全局项current来获得当前进程。设备驱动程序只要包含<linux/sched.h>头文件即可引用当前进程。例如，下面的语句通过访问struct task_struct的某些成员来打印当前进程的进程ID和命令名：

		printk(KERN_INFO "The process is \"%s\"(pid %i)\n",
			current->comm,current->pid);
5. 内核具有非常小的栈，他可能只有和一个4K字节大小的页那样小。我们自己的函数必须和整个内核空间调用链一同共享这个栈。因此，如果我们需要大的结构，则应该在调用时动态分配该结构。

6. 内核代码不能实现浮点运算

7. Makefile：

		obj-m := module.o
		make -C ~/kernel-2.6 M=`pwd` modules
	上述命令首先改变目录到-C指定的目录（即内核源代码目录），其中保存了内核的顶层makefile文件。M=选项让该makefile在构造modules目标之前返回模块源代码目录。然后，modules目标指向obj-m变量中设定的模块，上面的code中，为module.o
	
8. insmod： insmod和ld有点类似，他将模块的代码和数据装入内核，然后使用内核的符号表解析模块中任何未解析的符号。然而，与链接器不同，内核不会修改模块的磁盘文件，而仅仅修改内存中的副本。insmode依赖定义在kernel/module的一个系统调用，函数sys_init_module给模块分配内存以便转载模块，然后，该系统调用将模块正文复制到内存区域，并通过内核符号表解析模块中的内核引用，最后调用模块的初始化函数。

9. 公共内核符号表中包含了所有的全局内核项（即函数和变量）的地址。

10. 在首次注册完成之后，代码就应该准备好被内核其他部分调用；在用来支持摸个设施的所有内部初始化完成之前，不要注册任何设施。

11. 模块参数，内核支持的模块参数类型有：bool, invbool, charp, int, long, short, uint, ulong, ushort

		static char* whom = "world";
		module_param(whom, charp, S_IRUGO);
		module_param_array(name, type, num, perm);

### 第三章 字符设备驱动程序

* 1.dev_t类型(linux/types.h中定义)用来保存设备编号——包括12位的主设备号，和20位的次设备号。

		MAJOR(dev_t dev);
		MINOR(dev_t dev);

		MKDEV(int major, int minor);
		
* 2.分配和释放设备编号

		int register_chrdev_region(dev_t first, unsigned int count, char* name);
		int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name);
		
* 3.发布一个驱动时(scull)的安装脚本。

		#!/bin/sh
		module="scull"
		device="scull"
		mode=0644
		
		/sbin/insmod ./$module.ko $* || exit 1
		
		rm -f /dev/${device}[0-3]
		
		major=${awk "\$2=  =\"$module\" {print \$1}" /proc/devices)
		
		mknod /dev/${device}0 c $major 0
		mknod /dev/${device)1 c $major 1
		mknod /dev/${device}2 c $major 2
		mknod /dev/${device)3 c $major 3
		
		group="staff"
		grep -q '^staff:' /etc/group || group="wheel"
		
		chgrp $group /dev/${device}[0-3]
		chown $mode /dev/${device}[0-3]
* 4.文件操作
file_operations 结构或者指向这类结构的指针称为fops. 

		unsigned int (*poll)(struct file*, struct poll_table_struct *);// 是poll, epoll, select这三个系统调用的后端实现。这三个系统调用可用来查询某个或者多个文件描述符上读取或写入是否会被阻塞。poll方法应该返回一个位掩码，用来指出非阻塞的读取或写入是否可能，并且也会向内核提供将调用进程置于休眠状态直到I/O变为可能时的信息。如果驱动程序将poll方法定义为NULL，则设备会被认为即可读，也可写，并且不会阻塞。
		int (*mmap)(struct file*, struct vm_area_struct*);//用于请求将内存映射到进程地址空间。如果设备没有实现这个方法，mmap的系统调用就会返回-ENODEV。

mmap：linux系统调用：负责把文件内容映射到进程的虚拟空间内存

	void* mmap(void* addr,size_t len,int prot,int flags,int fd,off_t offset)
	flags: MAP_SHARED/MAP_PRIVATE

解除映射

	int munmap(void *start,size_t length)

mmap设备操作，映射一个设备是指把用户空间的一段地址关联到设备内存上。当程序读写这段用户空间的地址时，他实际上是在访问设备。对驱动程序员而言，需要知道设备的物理地址，用户空间的虚拟地址由系统分配，两种地址通过页式管理的方式进行关联，就是建立虚拟地址到物理地址的页表。

驱动中的mmap方法在file_operations中，函数原型：

	int (*mmap)(struct file *,struct vm_aren_struct*)
可以一次使用remap_pfn_range一次性建立所有的页表

	int remap_pfn_range(struct vm_aren_struct *vma, unsigned long addr, unsigned long pfn, unsigned long size, pgprot_t prot)
	
	vma:虚拟内存区域指针
	vir_addr:虚拟地址的起始值
	pfn：要映射的物理地址所在的物理页帧号，可将物理地址>>PAGE_SHIFT（4K）得到
	
下面这个用户程序展示了mmap系统调用的用法，本例中，使用open打开一个文件（例如文本文件）后，对获取到得文件句柄进行mmap，然后直接操作这段映射后的进程内存。注意，mmap操作不能增加原来的文件长度。

    #include <stdio.h>
    #include<sys/types.h>
    #include<sys/stat.h>
    #include<fcntl.h>
    #include<unistd.h>
    #include<sys/mman.h>
    #include <string.h>
    
    int main()
    {
    	int fd;
    	char *start;
    	char buf[100];
    	
    	fd = open("testfile.txt",O_RDWR);
            
    	start=mmap(NULL,100,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    	
    	strcpy(buf,start);
    	printf("buf = %s\n",buf);	
    
    	strcpy(start,"Buf Is Not Null!");
    	
    	munmap(start,100);
    	close(fd);  
    	
    	return 0;	
    }

* 5.file结构，位于<linux/fs.h>中。

file结构代表一个打开的文件（系统中每个打开的文件在内核空间中都有一个对应的file结构）。该结构与C库中（用户空间下）的FILE结构没有任何关联。

注册一个字符设备驱动推荐的方法：

	int register_chrdev(unsigned int major, const char* name, struct file_operations* fops);
	int unregister_chrdev(insigned int major, const char* name);

* 6. container_of(pointer, container_type, container_field);

例如：

		struct scull_dev* dev;
		
		dev = container_of(inode->i_cdev, struct scull_dev, cdev);



### 第十四章 Linux设备模型

* 1.sysfs文件系统：sysfs is a ram-based filesystem initially based on ramfs. It provides a means to export kernel data structures, their attributes, and the linkages between them to userspace.---documentation/filesystems/sysfs.txt. 
> sysfs 被加载在/sys/目录下，子目录包括block,bus,class,devices,kernel,module,firmware,fs,power

* 2.Kobject 实现了基本的面向对象管理机制，是构成Linux2.6设备模型的核心结构。它与sysfs文件系统紧密相连，在kernel中注册的每个kobject对象对应sysfs文件系统中得一个文件或目录。

		void kobject_init(struct kobject* kobj)
		int kobject_add(struct kobject* kobj)
		int kobject_init_and_add(struct kobject* kobj, struct kobj_type* ktype, struc kobject* parent, const char *fmt, ...)
		void kobject_del(struct kobject* kobj)


> kobject 包含一个struct kobj_type *ktype的成员，这个结构是

		struct kobj_type{
			void (*release)(struct kboject* kobj);
			struct sysfs_ops* sysfs_ops;
			struct attribute** default_attrs;
		};
		struct sysfs_ops{
			ssize_t (*show)(struct kobject*, struct attribute*, char* );/*当用户读属性文件时，该函数被调用，该函数将属性值存入buffer中返回给用户态*/
			ssize_t (*store)(struct kobject*, struct attribute*, const char*);/*当用户写属性文件时，该函数被调用，用于存储用户传入的属性值*/
		};

> kset 是具有相同类型kobject的集合，在sysfs中体现为一个目录，在内核中用kset数据结构表示，定义为

		struct kset{
			struct list_head list;
			spinlock_t list_lock;
			struct kobject kobj;
			struct kset_uevent_ops *uevent_ops;/*用于处理热插拔*/
		}
		struct kset_uevent_ops内有三个函数，filter, name, uevent.当filter返回值是0时，其他两个函数不会被调用。

![kset and kobject](/images/kset_kboject/kset_kobject.png)

在linux系统中，当系统配置发生变化时，如：添加kset到系统；移动kobject，一个通知会从内核空间发送到用户空间，这就是热插拔事件。热插拔时间会导致用户空间中相应的处理程序如 udev，mdev被调用，这些处理程序会通过加载驱动程序，创建设备节点等来响应热插拔事件。

一个用法实例：

    #include <linux/device.h>
    #include <linux/module.h>
    #include <linux/kernel.h>
    #include <linux/init.h>
    #include <linux/string.h>
    #include <linux/sysfs.h>
    #include <linux/stat.h>
    #include <linux/kobject.h>
     
    MODULE_LICENSE("Dual BSD/GPL");
     
    struct kset kset_p;
    struct kset kset_c;
    
    int kset_filter(struct kset *kset, struct kobject *kobj)
    {
            printk("Filter: kobj %s.\n",kobj->name);
            return 1;
    }
     
    const char *kset_name(struct kset *kset, struct kobject *kobj)
    {
            static char buf[20];
            printk("Name: kobj %s.\n",kobj->name);
            sprintf(buf,"%s","kset_name");
            return buf;
    }
     
    int kset_uevent(struct kset *kset, struct kobject *kobj,struct kobj_uevent_env *env)
    {
            int i = 0;
            printk("uevent: kobj %s.\n",kobj->name);
    
            while( i < env->envp_idx){
                    printk("%s.\n",env->envp[i]);
                    i++;
            }
    
            return 0;
    }
    
    struct kset_uevent_ops uevent_ops = 
    {
            .filter = kset_filter, //决定是否将事件传递到用户空间。如果返回0，将不传递事件
            .name   = kset_name, //用于将字符串传递给用户空间的热插拔处理程序
            .uevent = kset_uevent, //将用户空间需要的参数添加到环境变量中
    };
     
    int kset_test_init()
    {
            printk("kset test init.\n");
            kobject_set_name(&kset_p.kobj,"kset_p");
            kset_p.uevent_ops = &uevent_ops;
            kset_register(&kset_p);
             
            kobject_set_name(&kset_c.kobj,"kset_c");
            kset_c.kobj.kset = &kset_p;
            kset_register(&kset_c); //执行这个函数后，kset_uevent_ops的三个函数都会被调用
            return 0;
    }
     
    int kset_test_exit()
    {
            printk("kset test exit.\n");
            kset_unregister(&kset_p);
            kset_unregister(&kset_c);
            return 0;
    }
     
    module_init(kset_test_init);
    module_exit(kset_test_exit);

加载(insmod)该驱动后，kmsg打印去下log：
> kset test init.
> Filter: kobj kset_c.
> Name: kobj kset_c.
> uevent: kobj kset_c.
> ACTION=add.
> DEVPATH=/kset_p/kset_c.
> SUBSUSTEM=kset_name.

同时会创建在 /sys/kset_p/kset_c/ 的目录结构 


### 第十五章 内存映射的DMA



## About Linux kernel and driver

这部分介绍下一些相关的知识点，来自网络和平时学习看到的，待整理。

* 1.寄存器和内存的区别不同在于寄存器操作有副作用（side effect或边际效果）：读取摸个地址时可能导致该地址内容发生变化，比如很多设备的终端寄存器只要读取，就会清零
* 2.X86处理存在I/O空间的概念，I/O空间是相对内存空间而言的，他们是彼此独立的地址空间，在32位的x86系统中，I/O空间的大小为64K，内存空间为4G。
* 3.X86存在I/O空间，ARM只支持内存空间。当一个寄存器或内存位于I/O空间时，称为I/O端口，当寄存器或内存位于内存空间时，称为I/O内存
* 4.对I/O端口的操作需要按如下步骤完成：
> 1. 申请
>> struct resource *request_region(unsigned long first, unsigned long n, const char *name);
>> 系统中端口的分配情况记录在/proc/ioports中查看到
> 2. 访问
>> inb/outb
>> inw/outw
> 3. 释放
>> release_region(unsigned long start, unsigned long)

* 5.操作I/O内存
> 申请I/O内存
	
		struct resource *request_mem_region(unsigned long start, unsigned long len,char* name)；
>> 申请成功后会在/proc/iomem 中列出来
>> 申请成功的地址是物理地址，在访问I/O地址之前，要使用ioremap对该物理地址到虚拟地址的映射
> 访问：
> 释放I/O内存：
>> void iounmap(void *addr)
>> void release_mem_region()

* 6.上拉是将不确定的信号通过一个电阻与电源相连，固定在高电平。下拉是将不确定的信号通过一个电阻与地相连，固定在低电平。上拉是对器件注入电流（输入），下拉是输出电流（输出）。当一个接有上拉电阻的I/O端口设为输入状态是，它的常态为高电平，可用于检测低电平的输入。


		

> 总线方法：
> > int（*match）(struct device* dev,struct struct device_driver* drv)
> > int(* uevent)(struct devuce *dev, cahr **envp, int num_envp, char *buffer, int buffer_size)
> 在为用户空间产生热插拔事件之前，这个方法允许总线添加环境变量

> platform 总线是2.6内核加入的一种虚拟总线。有两部分组成: platform_device和platform_driver
> 优势在于platform机制将设备本身的资源注册进内核，由内核统一管理，在驱动程序使用这些资源时使用统一的接口，这样提高了程序的可移植性。
> int platform_device_add
> struct platform_device *platform_device_alloc(const char* name, int id)
> 平台设备资源使用struct resource 来描述：
		struct resource {
			resource_size start;
			resource_size end;
			const char *name;
			unsigned long flags; // IO,MEN,IRQ地址, IORESOURCE_MEM, IRQSOURCE_IRQ
			struct resource *parent, *sibling, *child;
		}
		struct resource* platform_get_resource(struct platform_device* dev, unsigned int type, unsigned int num); //type:资源的类型,即 IORESOURCE_ME这种
		
		int platform_driver_register(struct platform_driver* drv);//注册platform_driver
		函数流程： 
		platform_driver_register [drv->driver.bus = &platform_bus_type]  --->  driver_register --->  bus_add_driver --->  driver_attach ---> bus_for_each_dev ---> __driver_attach [drv->bus->match]--->  driver_probe_device ---> really_probe
		
9. 中断: 外设的处理速度一般慢于CPU，CPU不能一直等待外部事件，所以设备必须有一种方法来通知CPU它的工作进度，这种方法就是中断
> 中断实现， 有如下两个步骤：
> > 1.向内核注册中断 request（unsigned int irq，void(*handler)(int, void *, struct pt_regs *),unsigned long flags, const char* devname,void *dev_id）
>>> IRQF_DISABLED 如果设置该位，表示是一个快速中断处理程序；否则为慢速
>>> IRQF_SHARED 共享中断：就是将不同的设备挂到同一个中断信号线上，linux对共享的支持主要是为PCI设备服务。有三个特别之处： 1. 申请共享中断时，2. 必须加上该flags, dev_id参数必须是唯一的，3. 共享中断处理程序中，不能使用disable_irq(unsigned int irq)
>>>> 快速中断处理程序保证中断处理的原子性（不被打断），而慢速中断则不保证

> > 2. 实现中断处理函数
> 中断处理程序运行在中断上下文种，限制有三：1.不能向用户空间发送或者接受数据；2.不能使用可能引起阻塞的函数(wait,wait_interruptible)；3.不能使用可能引起调度的函数（schedule）
> 中断处理函数的流程：1. 判断是否本设备产生了中断（因为共享中断处理时，会调用所有注册的中断处理函数，通过该步骤判断哪个设备产生了中断）；2.清楚中断位（有些设备支持自动清楚，则不需要这一步）；3.中断处理，通常是数据接受；4.唤醒等待处理数据的进程
> 
> fops中的poll 函数中有两步操作，1.调用poll_wait函数来指明要使用的等待队列；2.返回是否可读、可写的掩码；
	static int read(struct file* filp, char __user* buff, size_t count, loff_t *offp)
	{
		unsigned long err;
		if(!ev_press){
			if(filp->f_flags & O_NONBLOCK)
				return -EAGAIN;
			else
				wait_event_interruptible();
		}
		ev_press = 0;
		
		err = copy_to_user(buff,(const void *)from)
	}

10. input 输入子系统： linux系统提供了input子系统，按键，触摸屏，鼠标等输入型设备都可以利用input接口函数来实现设备驱动。

drivers						handler

usb							keyboard handler

ps/2		core			mouse handler

serial					


diver ---> input core ---> handler

驱动层：将底层硬件输入转化为统一事件形式，向输入核心汇报
输入核心层：为驱动层提供输入设备注册与操作接口；通知事件处理层对事件进行处理；在/proc下产生相应的设备信息
事件处理层：主要作用是和用户空间交互 

设备描述：input设备用input_dev结构体描述，使用input 子系统实现输入设备驱动的时候，驱动的核心工作是向系统上报输入事件

input_register_device
inpuut_unregister_device

设备驱动通过set_bit告诉input子系统它支持那些事件，那些按键，例如：
		set_bit(EV_KEY,button_dev,.evbit)

struct input_dev 有两个成员：
evbit
evkey

事件类型：
EV_RST reset
EV_REL 相对坐标
EV_MSC 其它
EV_SND 声音
EV_KEY 按键
EV_FF 力反馈
EV_ABS 绝对坐标
EV_LED LED
EV_REP repeat

当事件类型为EV_KEY时，还需要指明按键类型：
BTN_LEFT
BTN_RIGHT
BTN_0
BTN_1等

input-report_key(struct input+dev* dev, unsigned int code, int value)
input_report_rel
input_report_abs

code : 如果事件类型EV_KEY，code可以为BTN_LEFT等
value：按下1，抬起0

报告结束：input_sync

实例，在一次触摸屏设备驱动中，一次点击的整个报告如下：
input_report_abs(input_dev, ABS_X, x);//x坐标
input_report_abs(input_dev, ABS_Y, y);//y坐标
input_report_abs(input_dev, ABS_PRESSURE, 1);
input_sync(input_dev);//同步


### Linux kernel组成
1. 系统调用接口
2. 内存管理子系统
3. 进程管理子系统
4. archtecture and
5. 虚拟文件系统
6. 网络协议栈
7. 驱动

Linux内核主要由进程调度（SCHED）、内存管理（MM）、虚拟文件系统（VFS）、网络接口（NET）和进程间通信（IPC）等5个子系统组成。


### 地址类型
物理地址：CPU地址总线上的寻址物理内存的地址信号
逻辑地址：程序编译之后，汇编代码中的地址
虚拟地址：又称线性地址，在32位CPU架构下，寻址范围为4G

### 地址转换
CPU要将一个逻辑地址转换为一个物理地址，两步：
+ 首先CPU利用段式内存管理单元，将逻辑地址转换成线性地址。
+ 再利用页式内存管理单元，把线性地址最终转换为物理地址

段式管理：

16位CPU内部拥有20为地址总线，寻址范围为1M。
逻辑地址 = 段基地址+段内偏移地址
计算公式：
物理地址 = 段寄存器<<4 + 逻辑地址

32位CPU采用两种模式：实模式 和 保护模式
实模式：内存管理与16位CPU是一致的
保护模式：段选择器


页式管理：







[Yunzhi]:    http://yunzhi.github.io  "Yunzhi"
