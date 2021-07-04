# Review of the Training

## 前言

入职培训项目，Android和Linux驱动开发相关，可惜我十分抗拒学习Java相关技术，所以没法完整的主导项目的实现，最终结果不尽人意，以下只对BSP部分进行记录。在培训结束的时候指导教师说抛出一个问题，你不能因为不符合你的技术栈你就不去做，因此考虑趁着还有双休，考虑完成前端的部分。

## Chapter I：Requirements

1. 分配一段内存，模拟成一个字符设备，写出其驱动程序。

2. 驱动中应该实现probe, store, show, open, read, write, ioctrl等函数。

3. ioctrl 可以接收特定的命令实现对应的操作。

   a) 当输入 A 时 A 线程启动，且定时打印该段内存的信息（使用 timer 实现）；

   b) 当输入 B 时 B 线程启动，且定时打印该段内存的信息（使用 workqueue 实现）；

4. 能够使用sysfs api创建文件节点（节点位置 /sys/test/meminfo） meminfo显示内存信息，内容由学员发挥。

5. 驱动允许上层APP进行加载和卸载，APP可以对设备节点进行访问。

6. 生成 /dev/write /dev/read /dev/InOut 三个设备节点,并可以供上层进行数据交换

## Chapter II：Introduction

### 2.1驱动编译

```makefile
#dir of linux kernel
KERN_PATH:=/lib/modules/$(shell uname -r )/build

all:
	make -C $(KERN_PATH) M=$(shell pwd) modules 
clean:
	make -C $(KERN_PATH) M=$(shell pwd)  clean
#dynamic load module
obj-m:=probe.o read.o write.o
```

### 2.2模块化机制

模块代码有两种运行方式，一是静态编译连接进内核，在系统启动过程中进行初始化；一是编译成可动态加载的module，通过insmod动态加载重定位到内核。这两种方式可以在Makefile中通过obj-y或obj-m选项进行选择。

而一旦可动态加载的模块目标代码（.ko）被加载重定位到内核，其作用域和静态链接的代码是**完全等价**的。所以这种运行方式的优点显而易见：

1. 可根据系统需要动态加载模块，以扩充内核功能，不需要时将其卸载，以释放内存空间；
2. 当需要修改内核功能时，只需编译相应模块，而不必重新编译整个内核。

缺点在于本机一旦关机，动态加载的module也将被卸载，因此某些内核必须的模块如vfs、platform_bus等都采用静态编译。

## Chapter II：Introduction

### 2.1驱动编译

```makefile
#dir of linux kernel
KERN_PATH:=/lib/modules/$(shell uname -r )/build

all:
	make -C $(KERN_PATH) M=$(shell pwd) modules 
clean:
	make -C $(KERN_PATH) M=$(shell pwd)  clean
#dynamic load module
obj-m:=probe.o read.o write.o
```

### 2.2模块化机制

模块代码有两种运行方式，一是静态编译连接进内核，在系统启动过程中进行初始化；一是编译成可动态加载的module，通过insmod动态加载重定位到内核。这两种方式可以在Makefile中通过obj-y或obj-m选项进行选择。

而一旦可动态加载的模块目标代码（.ko）被加载重定位到内核，其作用域和静态链接的代码是**完全等价**的。所以这种运行方式的优点显而易见：

1. 可根据系统需要动态加载模块，以扩充内核功能，不需要时将其卸载，以释放内存空间；
2. 当需要修改内核功能时，只需编译相应模块，而不必重新编译整个内核。

缺点在于本机一旦关机，动态加载的module也将被卸载，因此某些内核必须的模块如vfs、platform_bus等都采用静态编译。

```C
#include <linux/module.h>        // module_init  module_exit
#include <linux/init.h>            // __init   __exit

// install module
static int __init demo_init(void)
{    
    printk(KERN_INFO "helloworld~\n");
    return 0;
}

// uninstall module
static void __exit demo_exit(void)
{
    printk(KERN_INFO "goodbye world~\n");
}


module_init(demo_init);
module_exit(demo_exit);

//use MODULE_xxx to decribe mudole info
MODULE_LICENSE("GPL");                // license
MODULE_AUTHOR("aston");                //author
```

- insmod *.ko：安装驱动，使用module_init定义一个变量名init_module，其指针指向的地址与初始化函数相同

### 2.3字符设备注册

#### 2.3.1 static struct char_device_struct

所有字符型设备都由一个一个char_device_struct结构体描述：

```c
static struct char_device_struct {
	struct char_device_struct *next;	//list struct
	unsigned int major;		//major num
	unsigned int baseminor;	//minor num
	int minorct;			//minor num range
	char name[64];			//device name
	struct cdev *cdev;		
} *chrdevs[CHRDEV_MAJOR_HASH_SIZE];		//array range 0~255
```

#### 2.3.2字符设备注册函数

```c
static inline int register_chrdev(unsigned int major, const char *name, const struct file_operations *fops)
{
	return __register_chrdev(major, 0, 256, name, fops);
}
 
int __register_chrdev(unsigned int major, unsigned int baseminor,
		      unsigned int count, const char *name,
		      const struct file_operations *fops)
{
	struct char_device_struct *cd;
	struct cdev *cdev;
 
	cd = __register_chrdev_region(major, baseminor, count, name);
	
	cdev = cdev_alloc();
 
	cdev->owner = fops->owner;
	cdev->ops = fops;
	kobject_set_name(&cdev->kobj, "%s", name);
		
	err = cdev_add(cdev, MKDEV(cd->major, baseminor), count);
 
	cd->cdev = cdev;
 
	return major ? 0 : cd->major;
}
```

register_chrdev调用了__register_chrdev_region，强制设定次设备号范围为0~255，并且封装了cdev_init和cdev_add。当在用户空间打开设备文件时内核可以根据设备号快速定位此设备文件的cdev->file_operations结构体,从而调用驱动底层的open,close,read,write,ioctl等函数。

```c
int register_chrdev_region(dev_t from, unsigned count, const char *name)
{
	struct char_device_struct *cd;
	dev_t to = from + count;
	dev_t n, next;
 
	for (n = from; n < to; n = next) {
		next = MKDEV(MAJOR(n)+1, 0);
		if (next > to)
			next = to;
		cd = __register_chrdev_region(MAJOR(n), MINOR(n),
			       next - n, name);
	}
	return 0;
}
```

> register_chrdev_region根据要求的范围申请连续设备编号，同时需要手动cdev_init和cdev_add

```c
static struct char_device_struct *
__register_chrdev_region(unsigned int major, unsigned int baseminor,
			   int minorct, const char *name)
{
	struct char_device_struct *cd, **cp;
	int ret = 0;
	int i;
 
	cd = kzalloc(sizeof(struct char_device_struct), GFP_KERNEL);

	mutex_lock(&chrdevs_lock);
 
	/*
	 * 如果major为0则分配一个没有使用的主设备号
	 * 注意，从chrdevs[255]开始向下查找
	 */
	if (major == 0) {
		for (i = ARRAY_SIZE(chrdevs)-1; i > 0; i--) {
			if (chrdevs[i] == NULL)
				break;
		}
 
		if (i == 0) {
			ret = -EBUSY;
			goto out;
		}
		major = i;
		ret = major;
	}
 
	cd->major = major;
	cd->baseminor = baseminor;
	cd->minorct = minorct;
	strlcpy(cd->name, name, sizeof(cd->name));
	
	// return major % CHRDEV_MAJOR_HASH_SIZE;
	// 根据major获得哈希chrdevs数组索引
	i = major_to_index(major);
 
	// 寻找新设备插入的位置
	// *cp表示当前元素存在，使用拉链法解决散列冲突
	for (cp = &chrdevs[i]; *cp; cp = &(*cp)->next)
		if ((*cp)->major > major ||		//当前当前设备的主设备号大于要添加的设备的主设备号
	       ((*cp)->major == major && 	//如果主设备号相同，则根据次设备号找插入的位置
		   (((*cp)->baseminor >= baseminor) || ((*cp)->baseminor + (*cp)->minorct > baseminor))) )
			break;
 
	// 防止重叠：新申请的连续次设备号不能和已申请的设备号重叠
	if (*cp && (*cp)->major == major) {
		int old_min = (*cp)->baseminor;
		int old_max = (*cp)->baseminor + (*cp)->minorct - 1;
		int new_min = baseminor;
		int new_max = baseminor + minorct - 1;
 
		/* New driver overlaps from the left.  */
		if (new_max >= old_min && new_max <= old_max) {
			ret = -EBUSY;
			goto out;
		}
 
		/* New driver overlaps from the right.  */
		if (new_min <= old_max && new_min >= old_min) {
			ret = -EBUSY;
			goto out;
		}
	}
	// 将新申请的设备插入散列表
	cd->next = *cp;
	*cp = cd;	
	mutex_unlock(&chrdevs_lock);
	return cd;
out:
	mutex_unlock(&chrdevs_lock);
	kfree(cd);
	return ERR_PTR(ret);
}
```

> 2.6之前版本的内核使用register_chrdev来进行字符型设备的分配，每一个主设备号只能存放一种设备，它们使用相同的 file_operation 结构体，也就是说**内核最多支持 256 个字符设备驱动程序**。
>
> 从2.6版本开始新增了一个 register_chrdev_region 函数，它支持将同一个主设备号下的次设备号进行分段，每一段供给一个字符设备驱动程序使用，拓展了支持的设备数量。

内核使用一个chrdevs散列表来管理所有的字符设备驱动程序，数组范围[0~255]，索引方法为major%255，并使用拉链法解决寻址冲突。char_device_struct中的next指针可以指向主设备号相同的其他字符设备驱动程序，它们主设备号相同，各自次设备号相互之间不重叠，**但共享同一个file_operation**。

**chrdevs数组的结构图**

使用register_chrdev的话，不能指定此设备号，那么就会默认baseminor=0，count=256,造成了浪费。因为一个主设备号下的所有此设备号都只能对应同一个字符设备。使用__register_chrdev_region的话，则可以指定baseminor和count，就可以让一个主设备号下每一个次设备号都对应分配的字符设备。

```c
if (major) {
	devid = MKDEV(major, 0);
	 /* (major,0~1)对应fops, (major, 2~255)都不对应fops */
	register_chrdev_region(devid, 2, "demo"); 
} else {
	/* (major,0~1)对应 fops, (major, 2~255)都不对应fops */
	alloc_chrdev_region(&devid, 0, 2, "demo"); 
	major = MAJOR(devid);                     
}
cdev_init(&cdev, &fops);
cdev_add(&cdev, devid, 2);
```

### 2.4.应用层调用驱动

在字符设备注册阶段，register_chrdev将指向对应file_operation结构体的指针放入chrdevs数组major对应的位置中，file_operation包含了对硬件的所有操作，应用层通过调用file_operation的属性来对硬件进行相关操作。

```c
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
    ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
    int (*readdir) (struct file *, void *, filldir_t);
    unsigned int (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, int datasync);
    int (*aio_fsync) (struct kiocb *, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
    ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    int (*check_flags)(int);
    int (*flock) (struct file *, int, struct file_lock *);
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
    int (*setlease)(struct file *, long, struct file_lock **);
};
```

在用户空间open字符设备文件时，首先调用def_chr_fops->chrdev_open()函数(所有字符设备都调用此函数)，chrdev_open会调用kobj_lookup函数找到设备的cdev->kobject，从而得到设备的cdev,进而获得file_operations。

## Chapter III：A Simple Demo



## Chapter IV：Development

需求中的前两点已经在Chapter III中实现，不再过多赘述，以下只针对需求中的难点以及我开发过程中遇到的问题和解决方法做一个review。

### 4.1 probe()

- bus是物理总线的抽象。
- device是设备抽象，存在于bus之上。
- driver是驱动抽象，注册到bus上，用于驱动bus上的特定device。
- device和driver通过bus提供的match方法来匹配（通常是使用设备ID进行匹配）。
- driver匹配到device后，调用driver的probe接口驱动device。
- 一个driver可以驱动多个相同的设备或者不同的设备。
- probe函数其实就是接着init函数的工作完成设备的注册。

在理解了以上概念的基础上，应该对probe函数有了一个初步的了解，需求中的probe函数只是在bus上driver_mattch_device匹配成功后的与init相似的设备初始化函数，我们只需要简单了解probe函数的调用过程。

### 4.2 ioctl()

> Tips:ioctl()在2.6.35之后被unlocked_ioctl()所替代，区别在于ioctl由大内核锁保护，而unlocked_ioctl()需要自行实现锁机制。

#### 4.2.1 unlocked_ioctl()

```c
/*
函数说明：应用程序向设备发送特定指令

Parameters：
	file - 打开的文件指针
	cmd - 发送的命令
	arg - 
Returns：
	long int
*/
#include <"linux/ioctl.h">
long demo_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
	switch(cmd){
		case A:
			timer_thread_run();
			break;
		case B:
			wq = alloc_workqueue("wq",0,0);
			INIT_WORK(&task, print_workqueue);
			queue_work(wq, &task);
			break;
		default:
			break;
	}
	printk("%s,%d\n",__func__,__LINE__);
	return 0;
}
```

unlocked_ioctl()实际根据应用程序发送来的指令执行特定的功能，实际是一个switch-case语句，其中第二个参数cmd是选择的参数。

cmd是一个32位无符号整数，被分为四个部分：

- **type**:魔数，占8位（_IOC_TYPEBITS），用于区分不同的驱动
- **number**:序号，占8位（_IOC_NRBITS），用于给命令编号
- **direction**：方向（读写），占2位（_IOC_DIRBITS）
- **size**:数据大小，占14位（_IOC_SIZEBITS）

内核要求cmd需要以这样的形式组织，并在<asm/ioctl.h>中给出了_IO(type, number)宏定义用于将十进制数转化为符合格式的cmd指令。

```c
/*
说明：ioctl api
Parameters：
        fd - 文件
        cmd - 经_IO重定义后的用户指令
        TYPE - 用户指定的魔数
*/
#include <"asm-generic/ioctl.h">
#define TYPE 'A'
#define A _IO(TYPE, 1)
#define B _IO(TYPE, 2)

ioctl(fd, A);
ioctl(fd, B);
```

#### 4.2.2 timer

内核定时器：基于时钟滴答`jiffies`计数器，在不阻塞当前进程的情况下于未来某个特定时间点调度执行某个动作

```c
struct timer_list {
	/*
	 * All fields that change during normal runtime grouped to the
	 * same cacheline
	 */
	struct hlist_node	entry;
	unsigned long		expires;
	void			(*function)(struct timer_list *);
	u32			flags;

#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
};
```

- `expires`：期望timer执行的`jiffies`值，到达`jiffies`值 时，将执行`function`绑定的函数
- 在==内核2.8==之前`timer_list`结构体比现在多一个`data`成员，作为传递给`function`的参数，本次开发使用的内核版本为==4.15==
- `timer_list`结构体使用前必须初始化，旧版本使用`init_timer`或是`TIMER_INITIALIZER`完成初始化，新版采用`timer_setup(timer, callback, flags)`完成初始化

```c
struct timer_list timer;
timer.expires = jiffies + 3 * HZ;
timer_setup(&timer, print_process, 0);
add_timer(&timer);
// linux-2.8
// init_timer(&timer);
// timer.function = print_process;
// timer.data = 0;
// add_timer(&timer);
```

#### 4.2.3 workqueue

`workqueue`:允许内核代码请求某个函数在将来的时间被调用

- `create_workqueue(const char *name)`每个工作队列在创建时内核会在系统中的每个处理器上为该工作队列创建内核线程
- 向工作队列提交任务时需要填充一个`work_struct`结构

```c
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```

- `INIT_WORK`j将完成`work_struct`结构的初始化工作
- `queue_work`和`queue_delay_work`将`work`添加到指定的`workqueue`中

```c
static struct workqueue_struct *wq;
static struct work_struct task;
wq = alloc_workqueue("wq",0,0);
INIT_WORK(&task, print_workqueue);
queue_delay_work(wq, &task, jiffies + 3 * HZ);
```

### 4.3 sysfs

#### 4.3.1 sysfs的创建

`kobject`是隐藏在`sysfs`虚拟文件系统后的机制，对于`sysfs`中的每个目录，内核中都会存在一个对应的`kobject`。每个`kobject`输出一个或多个属性，在`sysfs`目录下表现为文件，因此要在`sysfs`目录下创建文件节点实际需要创建一个`kobject`。

```c
struct kobject *my_kobj = NULL;
my_kobj = kobject_create_and_add("test", NULL);
```

- `test`通过`kobject_set_name`设置为唯一的`kobject`名，即`sysfs`中的目录名，`kobject`的`kset`和`parent`均为`NULL`的情况下，会在`sysfs`目录最高层创建文件。

#### 4.3.2 sysfs.read/sysfs.write

创建`kobject`的时候会将一系列属性保存在`kobj_type`结构中

```c
struct kobj_type {
    void (*release) (struct kobject *);
    struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
}
```

`default_attrs`成员保存了属性列表用于创建该类型的每一个`kobject`，`sysfs_fs`提供实现这些属性的方法

```c
struct attribute {
	const char		*name;
	umode_t			mode;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	bool			ignore_lockdep:1;
	struct lock_class_key	*key;
	struct lock_class_key	skey;
#endif
};
```

```c
struct sysfs_ops {
	ssize_t	(*show)(struct kobject *, struct attribute *, char *);
	ssize_t	(*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

由于需求要求创建`sysfs/test/meminfo`层次目录，而通过`kobject`和`kset`构建`sysfs`目录下的层次结构需要对两种结构间关系有深入了解，但这两种结构实在过于复杂，因此在查看源码后发现通过`attribute_group`可以创建子目录。

```c
struct attribute_group {
	const char		*name;
	umode_t			(*is_visible)(struct kobject *,
					      struct attribute *, int);
	umode_t			(*is_bin_visible)(struct kobject *,
						  struct bin_attribute *, int);
	struct attribute	**attrs;
	struct bin_attribute	**bin_attrs;
};
```

- `name`：If specified, the attribute group will be created in a new subdirectory with this name.

```c
static struct kobj_attribute my_sysfs_read =__ATTR(sysshow, S_IRUSR, demo_show, NULL);
static struct kobj_attribute my_sysfs_write =__ATTR(syswrite, S_IWUSR, NULL,demo_store);
static struct attribute *my_sysfs_attr[] = {
	&my_sysfs_read.attr,
	&my_sysfs_write.attr,
	NULL,
};
static struct attribute_group my_sysfs_attr_group = {
	.name = "meminfo",		//make sub_dir
	.attrs = my_sysfs_attr,
};
```

```c
#define __ATTR(_name, _mode, _show, _store) {				\
	.attr = {.name = __stringify(_name),				\
		 .mode = VERIFY_OCTAL_PERMISSIONS(_mode) },		\
	.show	= _show,						\
	.store	= _store,						\
}

struct kobj_attribute {
	struct attribute attr;
	ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr,
			char *buf);
	ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr,
			 const char *buf, size_t count);
};
```

