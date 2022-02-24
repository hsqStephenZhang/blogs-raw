---
title: kobject
categories: [linux, driver, kobject, kset]
date: 2022-2-24 17:12:45
---


## 1. kobject 是什么

在 {% post_link linux/device/index Linux 设备驱动框架 %} 一文中，我们提到了 Linux 设备驱动是以 Device，Bus，Driver，Class 这几种结构为核心，从而方便管理。

由于设备与设备，设备与总线之间有非常明显的拓扑关系（设备接在总线上，设备也会有依赖关系），因此，可以通过树状结构进行管理，而 kobject 就是这种管理方式的核心，同时辅助以 kset/ktype，构成了设备框架的基石。

## 2. kobject internal

### 2.1 数据结构

```c
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```

关键字段的含义：
- name：设备名
- parent：指向父对象的指针，
- kset：该 kobject 所属的集合
- ktype：kobject 的类型
- sd：文件系统的目录项
- kref：引用计数

```c
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```

关键字段含义：
- list：所包含的 kobject 链表
- kobj：该 kset 也是一个特殊的 kobject
- uevent_ops：用于处理 uevent 的操作

```c
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	struct attribute **default_attrs;	/* use default_groups instead */
	const struct attribute_group **default_groups;
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
	void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

关键字段含义：
- release：释放 kobject 时，调用该函数，相当于析构函数
- sysfs_ops：处理 sysfs 的 show/store 操作，对应文件系统的 read/write
- attribute：在 sysfs 中暴露给用户的 driver 属性，可以对 driver 进行控制



### 2.2 kobject && kset

#### 2.2.1 kobject 和 kset 关联与区别

kobject 本身并不单独使用，而是内嵌在别的结构体中。如果用面向对象的思想来理解，那么 kobject 相当于**基类**，包含了 kobject 的结构体则是**子类**。kset 就是一个典型的子类型。

正因为 kobject 这种**可嵌入**的特点，可以很容易将各种数据结构按照树状结构组织起来，通过 `contrainer_of` 可以快速获取到包含 kobject 特定结构，因此，包含了 kobject 的 high-level 数据结构也有着和 kobject 类似的功能（子类继承了父类）。

前面说到，kobject 对应 sysfs 中的一个文件/目录，而 kset 作用与此类似，只不过有一个限定条件：如果需要在sysfs的目录中包含多个子目录，那需要将它定义成一个 kset，其余情况均使用 kobject 表示。

#### 2.2.2 kobject 和 kset 的使用

因为 kobject 和 kset 仅仅作为一个结构体嵌入到别的复杂结构中，充当树状结构的骨架，所以其操作也比较简单：

```text

kobject_init_and_add --------
                            |
                        kobject_add ---- kobject_add_varg ---- kobject_add_internal
                            |                                           |
kobject_create_and_add ------                                           |
                                                                        |
                                                                        |
kset_create_and_add ------ kset_create                                  |
                      ---- kset_register --------------------------------
```

主要代码都是分配结构体，初始化各个字段，构建拓扑关系（设置 kset->list 和 kobject->parent）
