---
title: char device driver
categories: [linux, device, driver, char]
date: 2022-2-25 14:05:45
---

## 1. 什么是字符设备驱动

Linux将设备分为三大类：字符设备、块设备和网络设备。字符设备是其中较为基础的一类，它的读写操作需要一个字节一个字节地进行，无法随机读取设备中的某一字节的数据。比较常见的字符设备有鼠标、键盘、串口等。

## 2. char device driver internal

### 2.1 数据结构

Linux 使用 cdev 结构体来表示一个字符设备

```c
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;
	dev_t dev;
	unsigned int count;
} __randomize_layout;
```

kojb 不必多说，嵌入在 cdev 中，从而构成设备间的层次关系。

其中，dev_t 表示设备号，一共有 32 bit，其中 24 bit 为主设备号，8 bit 为次设备号，可以通过下面的宏快速操作

```c
#define MAJOR(dev)	((dev)>>8)
#define MINOR(dev)	((dev) & 0xff)
#define MKDEV(ma,mi)	((ma)<<8 | (mi))
```

另一个关键成员 ops 则定义了字符设备驱动提供给虚拟文件系统的接口函数。

### 2.2 设备初始化

每种设备都需要分配空间+初始化之后才能使用，字符设备也不例外，分别对应 cdev_alloc 和cdev_device_add。

```c
struct cdev *cdev_alloc(void)
{
	struct cdev *p = kzalloc(sizeof(struct cdev), GFP_KERNEL);
	if (p) {
		INIT_LIST_HEAD(&p->list);
		kobject_init(&p->kobj, &ktype_cdev_dynamic);
	}
	return p;
}
```

```c
int cdev_device_add(struct cdev *cdev, struct device *dev)
{
	int rc = 0;

	if (dev->devt) {
		cdev_set_parent(cdev, &dev->kobj);

		rc = cdev_add(cdev, dev->devt, 1);
		if (rc)
			return rc;
	}

	rc = device_add(dev);
	if (rc)
		cdev_del(cdev);

	return rc;
}
```

同时需要注意，调用 `cdev_device_add` 添加设备之前，还需要通过 `register_chrdev_region` 申请设备号，从而赋予该设备唯一的标识

### 2.3 文件系统 API

file_operations 中的回调函数是字符设备驱动的关键，这些函数会在虚拟文件系统的 open，read，write，close 等操作时被内核回调。

下面通过一些具体的例子来说明字符设备驱动应该如何设置 file_operations

#### 2.3.1 tty

```c
static const struct file_operations tty_fops = {
	.llseek		= no_llseek,
	.read		= tty_read,
	.write		= tty_write,
	.poll		= tty_poll,
	.unlocked_ioctl	= tty_ioctl,
	.compat_ioctl	= tty_compat_ioctl,
	.open		= tty_open,
	.release	= tty_release,
	.fasync		= tty_fasync,
	.show_fdinfo	= tty_show_fdinfo,
};
```


