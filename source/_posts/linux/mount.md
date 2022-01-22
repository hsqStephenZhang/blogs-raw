---
title: Mount
categories: linux
date: 2022-1-19 23:26:40
---

## 1. 什么是 mount

一个文件系统创建之后，并无法直接使用，相当于完成了注册，但是还未登录，对于文件系统来说，登录的这个步骤就是 mount。

一般来说，如果有一个磁盘设备，比如将一个 ext2 格式的移动硬盘(假设分配的设备名为 `/dev/sdb1`)插到了 Linux 主机上，若系统没有给我们自动挂载，就需要我们手动输入 `mount` 命令来挂载，如：

`mount -t ext2 /dev/sdb1 /mnt/new_disk`

这条命令的意思是：将 `/dev/sdb1` 上的 ext2 文件系统挂载到 `/mnt/new_disk` 上，其实不指定 `-t ext2` 也是可以的，系统可以探测出来具体使用的什么文件系统。

## 2. mount 系统调用

从另一个角度来看上面的 mount 命令，不管怎么样，最终还是得切换到内核态，执行系统调用，mount 对应的系统调用为

```man
NAME
       mount - mount filesystem

SYNOPSIS
       #include <sys/mount.h>

       int mount(const char *source, const char *target,
                 const char *filesystemtype, unsigned long mountflags,
                 const void *data);
```

其精确定义为：mount()  attaches the filesystem specified by source (which is often a pathname referring to a device, but can also be the pathname of a directory or file, or a dummy string) to the location (a directory or file) specified  by  the  pathname  in target.
