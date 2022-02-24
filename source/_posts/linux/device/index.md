---
title: linux driver framework
categories: [linux, driver, framework]
date: 2022-2-24 14:16:45
---

## 1. Linux 设备框架

### 1.1 软件架构

首先必须明确一点，驱动是 Linux 中最为复杂，也是代码量最为庞大的部分。并不是因为其架构多么匪夷所思，而是作为一个通用操作系统，Linux 必须要支持 30 多种体系下面的无数硬件设备，这种复杂度已经达到了 O(n*m) 的级别。

设想一下，如果由你为一个设备编写跨平台驱动，肯定想通过尽可能少的代码改动，完成各个平台的适配，这绝非易事。如果使用普通的 `#ifdef` `menuconfig` 等方式来完成配置，也存在很多 boilerplate，因此，一个良好的框架就显得非常关键了。

下面来简化一下我们遇到的问题：让 N 个不同的设备在 M 中不同的 CPU 架构上运行起来。

这个问题咋一看复杂度高得吓人，`O(N*M)`，是一种典型的强耦合，不符合软件工程 **高内聚，低耦合** 的基本原则。为了改进这种**网状架构**，可以引入一个**中间层**（计算机世界所有问题都可以通过引入一个间接的中间层解决），让 N 个设备与中间层耦合，中间层与 M 种 CPU 架构耦合，这样就将复杂度降低了一个数量级。

## 2. 设备驱动模型

上面提到，可以通过分层的思想，降低设备框架的复杂度，具体是如何操作的呢？首先来看几个关键的概念

## 2.1 设备，总线，驱动

- 设备：某种物理设备，比如网卡，硬盘，USB设备等各式各样的外设。
- 总线：设备之间传递数据的通道，可以多个设备共用一条总线，也可以单个设备独享总线，一般总线的一端总是 CPU。
- 驱动：与特定设备相关的程序，用来告诉 CPU 如何控制设备工作。

其中，设备和驱动是紧密相连的，设备依赖驱动来发挥功能，但是，正如上面对软件架构所分析的那样，如果设备和驱动之间直接进行匹配，后果就是强耦合，高复杂度。庆幸的一点是，还有总线！

总线充当了设备与驱动之间的中间层，分别集中对设备，驱动进行管理，就完成了解耦。

### 2.2 总线

#### 2.2.1 数据结构

总线对应下面的结构体：

```c
struct bus_type {
	const char		*name;
	const char		*dev_name;
	struct device		*dev_root;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	int (*num_vf)(struct device *dev);

	int (*dma_configure)(struct device *dev);

	const struct dev_pm_ops *pm;

	const struct iommu_ops *iommu_ops;

	struct subsys_private *p;
	struct lock_class_key lock_key;

	bool need_parent_lock;
};
```

各个字段的含义：
- name：总线的名称，每注册一种新的总线，就会在 `/sys/bus` 下面创建一个名为 name 的新目录
- bus_groups、dev_groups、drv_groups：分别表示 总线、设备、驱动的属性
- match：向总线注册一个新的设备或新的驱动，调用该函数进行匹配，
- probe：总线完成设备与驱动的匹配之后，调用该函数，最终调用驱动提供的 probe 函数
- remote：删除总线时，调用该函数
- p：私有结构，其中的 klist_devices 和 klist_drivers 分别保存了该总线上所有的设备和驱动

实际上，可以将总线也看成一个设备，不过这种设备比较特殊，它存在的目的是为了管理别的普通设备

#### 2.2.2 总线注册/注销

既然是设备，也就对应 register/unregister 函数

```c
int bus_register(struct bus_type *bus)
{
	int retval;
	struct subsys_private *priv;
	struct lock_class_key *key = &bus->lock_key;

    // 1. 分配空间
	priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

	priv->bus = bus;
	bus->p = priv;

    // 2. 初始化 notifier
	BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

    // 3. 设置 bus 名称
	retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
	if (retval)
		goto out;

	priv->subsys.kobj.kset = bus_kset;
	priv->subsys.kobj.ktype = &bus_ktype;
	priv->drivers_autoprobe = 1;

    // 4. 注册 kset
	retval = kset_register(&priv->subsys);
	if (retval)
		goto out;

    // 5. 在 /sys/bus/xxx_name 下创建 uevent 文件
	retval = bus_create_file(bus, &bus_attr_uevent);
	if (retval)
		goto bus_uevent_fail;

    // 6. 创建 bus 目录下的 devices 目录
	priv->devices_kset = kset_create_and_add("devices", NULL,
						 &priv->subsys.kobj);
	if (!priv->devices_kset) {
		retval = -ENOMEM;
		goto bus_devices_fail;
	}

    // 7. 创建 bus 目录下的 drivers 目录
	priv->drivers_kset = kset_create_and_add("drivers", NULL,
						 &priv->subsys.kobj);
	if (!priv->drivers_kset) {
		retval = -ENOMEM;
		goto bus_drivers_fail;
	}

    // 8. 初始化 klist_devices 和 klist_drivers
	INIT_LIST_HEAD(&priv->interfaces);
	__mutex_init(&priv->mutex, "subsys mutex", key);
	klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
	klist_init(&priv->klist_drivers, NULL, NULL);

    // 9. 创建 bus 目录下的 drivers_autoprobe 和 drivers_probe 文件
	retval = add_probe_files(bus);
	if (retval)
		goto bus_probe_files_fail;

	retval = bus_add_groups(bus, bus->bus_groups);
	if (retval)
		goto bus_groups_fail;

	return 0;

    // 失败处理
	return retval;
}
EXPORT_SYMBOL_GPL(bus_register);

void bus_unregister(struct bus_type *bus)
{
	pr_debug("bus: '%s': unregistering\n", bus->name);
	if (bus->dev_root)
		device_unregister(bus->dev_root);
	bus_remove_groups(bus, bus->bus_groups);
	remove_probe_files(bus);
	kset_unregister(bus->p->drivers_kset);
	kset_unregister(bus->p->devices_kset);
	bus_remove_file(bus, &bus_attr_uevent);
	kset_unregister(&bus->p->subsys);
}
EXPORT_SYMBOL_GPL(bus_unregister);
```

如果对于 kobject/kset 感到疑惑，可以详细阅读另一篇 {% post_link linux/device/kobject 文章%}，这里只需要认识到，kobject/kset 是一种设备树状管理的方式，对应 sysfs 中的目录和文件，可以将其类比为**多叉树的结点**

可以看到，register 操作大部分都是在 创建 sysfs 下所需的文件（夹），通过 `/sys/bus/BUS_NAME/devices` 和 `/sys/bus/BUS_NAME/drivers` 两个目录向用户暴露设备以及驱动列表，通过其余的一些文件暴露 bus 本身的一些属性。

unregister 操作这里不再赘述

### 2.3 设备

#### 2.3.1 数据结构

```c
struct device {
	struct kobject kobj;
	struct device		*parent;

	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */

	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
};
```

关键字段的含义：
- kobj：内嵌的 kobject，用于设备树状结构管理
- parent：该设备的父对象
- init_name：设备名称
- bus：归属于哪一个总线
- release：设备注销时，执行该回调函数

#### 2.3.2 设备注册/注销

设备需要注册之后才能使用，对应  `device_register`

```c
int device_register(struct device *dev)
{
	device_initialize(dev);
	return device_add(dev);
}
EXPORT_SYMBOL_GPL(device_register);

int device_add(struct device *dev)
{
	struct device *parent;
	struct kobject *kobj;
	struct class_interface *class_intf;
	int error = -EINVAL;
	struct kobject *glue_dir = NULL;

	dev = get_device(dev);
	if (!dev)
		goto done;

	if (!dev->p) {
		error = device_private_init(dev);
		if (error)
			goto done;
	}

    // 1. 设置 device 对应 kobject name
	if (dev->init_name) {
		dev_set_name(dev, "%s", dev->init_name);
		dev->init_name = NULL;
	}

	/* subsystems can specify simple device enumeration */
	if (!dev_name(dev) && dev->bus && dev->bus->dev_name)
		dev_set_name(dev, "%s%u", dev->bus->dev_name, dev->id);

	if (!dev_name(dev)) {
		error = -EINVAL;
		goto name_error;
	}

	pr_debug("device: '%s': %s\n", dev_name(dev), __func__);

    // 设置 device parent
	parent = get_device(dev->parent);
	kobj = get_device_parent(dev, parent);
	if (IS_ERR(kobj)) {
		error = PTR_ERR(kobj);
		goto parent_error;
	}
	if (kobj)
		dev->kobj.parent = kobj;

	/* use parent numa_node */
	if (parent && (dev_to_node(dev) == NUMA_NO_NODE))
		set_dev_node(dev, dev_to_node(parent));

	// 将 dev 注册到 sysfs 中（需要指定父结点）
	error = kobject_add(&dev->kobj, dev->kobj.parent, NULL);
	if (error) {
		glue_dir = get_glue_dir(dev);
		goto Error;
	}

	// 事件通知
	error = device_platform_notify(dev, KOBJ_ADD);
	if (error)
		goto platform_error;

    // 创建 device 的 sysfs 属性文件
	error = device_create_file(dev, &dev_attr_uevent);
	if (error)
		goto attrError;

	error = device_add_class_symlinks(dev);
	if (error)
		goto SymlinkError;
	error = device_add_attrs(dev);
	if (error)
		goto AttrsError;

    // 将 device 添加到 bus 的 klist_devices 中，并且在 sysfs 中创建一些 link 文件
	error = bus_add_device(dev);
	if (error)
		goto BusError;
	error = dpm_sysfs_add(dev);
	if (error)
		goto DPMError;
	device_pm_add(dev);

	if (MAJOR(dev->devt)) {
		error = device_create_file(dev, &dev_attr_dev);
		if (error)
			goto DevAttrError;

		error = device_create_sys_dev_entry(dev);
		if (error)
			goto SysEntryError;

		devtmpfs_create_node(dev);
	}

	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_ADD_DEVICE, dev);

    // uevent 通知
	kobject_uevent(&dev->kobj, KOBJ_ADD);

	if (dev->fwnode && !dev->fwnode->dev) {
		dev->fwnode->dev = dev;
		fw_devlink_link_device(dev);
	}

    // 完成 device 和 driver 的匹配，若匹配成功，执行 probe 函数
	bus_probe_device(dev);
	if (parent)
		klist_add_tail(&dev->p->knode_parent,
			       &parent->p->klist_children);

	if (dev->class) {
		mutex_lock(&dev->class->p->mutex);
		/* tie the class to the device */
		klist_add_tail(&dev->p->knode_class,
			       &dev->class->p->klist_devices);

		/* notify any interfaces that the device is here */
		list_for_each_entry(class_intf,
				    &dev->class->p->interfaces, node)
			if (class_intf->add_dev)
				class_intf->add_dev(dev, class_intf);
		mutex_unlock(&dev->class->p->mutex);
	}
done:
	put_device(dev);
	return error;

    // 错误处理 ...
	goto done;
}
EXPORT_SYMBOL_GPL(device_add);
```

关键的一些操作已经添加了注释，这里还需要强调一下 `bus_probe_device`。

当设备和驱动匹配成功之后，会执行一些初始化操作，这一步必不可少，因为很多脏活累活都是在 probe 回调函数中完成的。

该 probe 函数就在 `device_driver` 结构中，而完成匹配+执行 probe 函数的任务，就交给了 `bus_probe_device`，其关键函数调用链如下所示：

```text
bus_probe_device
    --> device_initial_probe
        --> __device_attach
            --> __device_attach_driver
                --> driver_match_device
                --> driver_probe_device
                    --> really_probe
                        --> driver->probe/bus->probe
```

如果弄清楚了 `device_register` 的操作，那么 `device_unregister` 对你来说应该不是难事，只需要反向执行注册时执行的操作即可，详细操作可以参考 `device_del` 函数。

```c
void device_unregister(struct device *dev)
{
	pr_debug("device: '%s': %s\n", dev_name(dev), __func__);
	device_del(dev);
	put_device(dev);
}
EXPORT_SYMBOL_GPL(device_unregister);
```

### 2.4 驱动

#### 2.4.1 数据结构

```c
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;

	const struct dev_pm_ops *pm;
	void (*coredump) (struct device *dev);

	struct driver_private *p;
};
```

关键字段含义：
- name：驱动名称，bus 通常会将该名称与设备名匹配
- bus：归属于哪个总线
- owner：该驱动的拥有者，一般为 THIS_MODULE
- probe：当驱动和设备匹配之后，会执行该函数，对设备初始化
- remove：设备从系统中移出或者系统关闭的时候，执行该函数

#### 2.4.2 驱动注册/注销

驱动 注册/注销 对应 `driver_register` 和 `driver_unregister`。

```c
int driver_register(struct device_driver *drv)
{
	int ret;
	struct device_driver *other;

	if (!drv->bus->p) {
		pr_err("Driver '%s' was unable to register with bus_type '%s' because the bus was not initialized.\n",
			   drv->name, drv->bus->name);
		return -EINVAL;
	}

    // 判断回调函数是否完备
	if ((drv->bus->probe && drv->probe) ||
	    (drv->bus->remove && drv->remove) ||
	    (drv->bus->shutdown && drv->shutdown))
		pr_warn("Driver '%s' needs updating - please use "
			"bus_type methods\n", drv->name);

    // 根据 driver 名称在bus中查找该driver是否已注册
	other = driver_find(drv->name, drv->bus);
	if (other) {
		pr_err("Error: Driver '%s' is already registered, "
			"aborting...\n", drv->name);
		return -EBUSY;
	}

    // 将 driver 添加到 bus 中，同时完成匹配+probe操作
	ret = bus_add_driver(drv);
	if (ret)
		return ret;
	ret = driver_add_groups(drv, drv->groups);
	if (ret) {
		bus_remove_driver(drv);
		return ret;
	}
    // uevent 通知
	kobject_uevent(&drv->p->kobj, KOBJ_ADD);

	return ret;
}
EXPORT_SYMBOL_GPL(driver_register);
```

与 device 类似，添加一个驱动时，也需要查找有没有匹配的设备，如果有，就执行 probe 函数，这一步是在 `bus_probe_device` 中完成的，其函数调用链如下：

```text
bus_add_driver
    --> driver_attach
        --> __device_attach
            --> __device_attach_driver
                --> driver_match_device
                --> driver_probe_device
                    --> really_probe
                        --> driver->probe/bus->probe
```

同理，驱动的注销对应 `driver_unregister`，详细操作位于 `bus_remove_driver`，也基本是注册驱动时的逆向操作

```c
void driver_unregister(struct device_driver *drv)
{
	if (!drv || !drv->p) {
		WARN(1, "Unexpected driver unregister!\n");
		return;
	}
	driver_remove_groups(drv, drv->groups);
	bus_remove_driver(drv);
}
EXPORT_SYMBOL_GPL(driver_unregister);
```
