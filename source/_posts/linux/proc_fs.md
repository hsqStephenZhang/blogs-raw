---
title: Proc Fs
categories: linux
date: 2022-1-19 16:49:40
---

## 1. procfs 是什么

众所周知，文件系统是 Linux 的骨架，proc 文件系统就是其中一个**展示程序运行状态**的**虚拟**文件系统。

如果你查看 `/proc` 目录下的内容，会发现，除了一堆数字命名的文件夹，还有很多杂七杂八的文件，比如 `kallsyms cpuinfo meminfo`，这些文件也都是系统的一些信息，比如 kallsyms 可以展示当前 vmlinux 的所有符号，cpuinfo 展示的是 cpu 的一些参数，meminfo 展示的是内存的一些参数。这些实现都比较简单；稍微复杂一些的是以数字命名的文件夹，也就是所有进程的信息，这也正是 proc 文件系统名字的由来。

之前的一篇 {% post_link linux/seq_file 文章 %}中已经介绍了 procfs 下 `/proc/kallsyms` 以及 seq_file 的实现机制，而本文着重 procfs 的整体框架，并着重分析和 process 有关的 `/proc/${pid}` 一系列内容

## 2. procfs 的 inode 以及 file_operations

了解一个文件系统，可以直接从 inode_operations && file_operations 入手

在 `fs/proc/root.c` 可以查到，procfs 的根目录为 `/proc`，根目录项为 `proc_root`，对应 inode_operations 为 `proc_root_inode_operations`，最关键的就是 lookup 回调函数

其实这种虚拟文件系统的 lookup 已经比 ext 这种磁盘文件系统简化很多了，大部分 dentry 都是非常有规律地保存在内存中的，而不像磁盘那样还要和硬件存取打交道。

```c
// fs/proc/root.c
static const struct inode_operations proc_root_inode_operations = {
    .lookup        = proc_root_lookup,
    // ...
};

struct proc_dir_entry proc_root = {
    .proc_iops    = &proc_root_inode_operations, 
    .proc_dir_ops    = &proc_root_operations,
    .name        = "/proc",
    // ...
};
```

`proc_root_lookup` 逻辑非常简单：首先尝试 `proc_pid_lookup`，失败之后尝试 `proc_lookup`。从名字可以猜想一波，是不是前者和进程 pid 有关，后者比较普通呢？

```c
static struct dentry *proc_root_lookup(struct inode * dir, struct dentry * dentry, unsigned int flags)
{
    if (!proc_pid_lookup(dentry, flags))
        return NULL;

    return proc_lookup(dir, dentry, flags);
}
```

The anwser is: Yes!

上面已经提到，procfs 的内容正好可以分为两大类：1.进程信息；2.额外的系统信息，而 `/proc/${pid}` 和 `/proc/cpuinfo` 这两种文件，对应的查找方式就分别为 `proc_pid_lookup` 和 `proc_lookup`

后者是比较常见一些的，通过 rb_tree 进行目录项的查找，前者是 procfs 的基础，所以接下来探究一下 proc_pid_lookup 做了什么

如下所示，`proc_pid_lookup` 主要做了下面的几件事：

1. 通过 dentry 获取文件的 fs_info，从而得到 namespace
2. 通过 pid 查找进程 task_struct 结构体
3. 判断该进程是否是隐藏起来的，如果是，就无法在 procfs 中显示
4. 初始化 pid 对应的 dentry

最后一点非常有意思，什么叫初始化 dentry？这是因为，procfs 是一个虚拟文件系统，所有的文件并没有持久化保存，而是随用随取，自然，这些目录项也没有必要在系统启动的时候就创建好，而是等到 lookup 操作的时候再按需创建

```c
struct dentry *proc_pid_lookup(struct dentry *dentry, unsigned int flags)
{
    struct task_struct *task;
    unsigned tgid;
    struct proc_fs_info *fs_info;
    struct pid_namespace *ns;
    struct dentry *result = ERR_PTR(-ENOENT);

    tgid = name_to_int(&dentry->d_name);
    if (tgid == ~0U)
        goto out;

    // 1. 获取 fs_info
    fs_info = proc_sb_info(dentry->d_sb);
    ns = fs_info->pid_ns;
    rcu_read_lock();
    
    // 2. 获取 pid 对应的 task
    task = find_task_by_pid_ns(tgid, ns);
    if (task)
        get_task_struct(task);
    rcu_read_unlock();
    if (!task)
        goto out;

    // 3. 判断该 task 是否被隐藏
    if (fs_info->hide_pid == HIDEPID_NOT_PTRACEABLE) {
        if (!has_pid_permissions(fs_info, task, HIDEPID_NO_ACCESS))
            goto out_put_task;
    }

    // 4. 初始化 dentry
    result = proc_pid_instantiate(dentry, task, NULL);
out_put_task:
    put_task_struct(task);
out:
    return result;
}
```

`proc_pid_instantiate` 主要是分配了 inode，并且设置了对应的 `inode_operations && file_operations`，这样的话，下一次更深入一层，在 `/proc/${pid}` 下查找目录项的时候，就会触发 `proc_tgid_base_inode_operations.proc_tgid_base_lookup`

```c

static const struct inode_operations proc_tgid_base_inode_operations = {
    .lookup        = proc_tgid_base_lookup, // !important
    // ...
};

static struct dentry *proc_pid_instantiate(struct dentry * dentry,
                   struct task_struct *task, const void *ptr)
{
    struct inode *inode;

    // 创建 inode
    inode = proc_pid_make_inode(dentry->d_sb, task, S_IFDIR | S_IRUGO | S_IXUGO);
    if (!inode)
        return ERR_PTR(-ENOENT);

    // 设置回调函数
    inode->i_op = &proc_tgid_base_inode_operations;
    inode->i_fop = &proc_tgid_base_operations; // 不太重要
    inode->i_flags|=S_IMMUTABLE;

    set_nlink(inode, nlink_tgid);
    pid_update_inode(task, inode);

    d_set_d_op(dentry, &pid_dentry_operations);
    return d_splice_alias(inode, dentry);
}
```

重新整理一下，从 `/proc` 到 `/proc/${pid}`，其 inode 对应的操作是不同的，前者需要兼顾进程信息和普通系统信息两种类型的文件，后者则负责查找每一个进程所对应的虚拟文件（进程状态）

## 2. procfs 如何显示进程状态

前面说到，通过一级级的 dentry 查找，触发 inode 的 lookup 函数之后，来到了 `proc_tgid_base_lookup`。

比如，如果执行命令 `cat /proc/1/maps`，就会进入 `proc_pident_lookup`，参数中 `dentry->d_name.name == 'maps'` ，所以这个函数中就需要比较 `/proc/1` 目录下，是否有和 'maps' 名字相同的文件

暂时不了解 pid_entry 结构没有关系，就算盲猜，也知道这个结构当中保存了目录项的 `name,name's length` 两个属性，这个函数的操作也就非常简单了，就是通过一个 `proc_pident_lookup`，循环比较 `pid_entry` 数组中的所有元素和当前查找的 dentry，比较名字是否相同。有意思的一点是，时间复杂度并不是 O(N)，因为 `/proc/${pid}` 下的所有文件都是固定的，几乎没有改动，所以复杂度还是 O(1) 的，源码中也注明了: 'Yes, it does not scale. And it should not'

```c
static struct dentry *proc_pident_lookup(struct inode *dir, 
                     struct dentry *dentry,
                     const struct pid_entry *p,
                     const struct pid_entry *end)
{
    struct task_struct *task = get_proc_task(dir);
    struct dentry *res = ERR_PTR(-ENOENT);

    if (!task)
        goto out_no_task;

    for (; p < end; p++) {
        if (p->len != dentry->d_name.len)
            continue;
        if (!memcmp(dentry->d_name.name, p->name, p->len)) {
            res = proc_pident_instantiate(dentry, task, p);
            break;
        }
    }
    put_task_struct(task);
out_no_task:
    return res;
}

static struct dentry *proc_tgid_base_lookup(struct inode *dir, struct dentry *dentry, unsigned int flags)
{
    return proc_pident_lookup(dir, dentry,
                  tgid_base_stuff,
                  tgid_base_stuff + ARRAY_SIZE(tgid_base_stuff));
}
```

`struct pid_entry` 用来描述一个进程下的虚拟文件，其结构如下，核心也是 iop 和 fop 这两个 operations，这样就可以控制 `/proc/${pid}` 下面每一个文件的打开读取方式，以及每一个子目录的查找方式

```c
struct pid_entry {
    const char *name; // 目录项名称
    unsigned int len; // 目录项名称长度
    umode_t mode; // 文件权限
    const struct inode_operations *iop; // inode 操作
    const struct file_operations *fop; // file 操作
    union proc_op op; //
};
```

并且，通过 NOD，DIR 等一系列宏，又可以减少很多 boilerplate 样板代码

```c
#define NOD(NAME, MODE, IOP, FOP, OP) {            \
    .name = (NAME),                    \
    .len  = sizeof(NAME) - 1,            \
    .mode = MODE,                    \
    .iop  = IOP,                    \
    .fop  = FOP,                    \
    .op   = OP,                    \
}

#define DIR(NAME, MODE, iops, fops)    \
    NOD(NAME, (S_IFDIR|(MODE)), &iops, &fops, {} )
#define LNK(NAME, get_link)                    \
    NOD(NAME, (S_IFLNK|S_IRWXUGO),                \
        &proc_pid_link_inode_operations, NULL,        \
        { .proc_get_link = get_link } )
#define REG(NAME, MODE, fops)                \
    NOD(NAME, (S_IFREG|(MODE)), NULL, &fops, {})
#define ONE(NAME, MODE, show)                \
    NOD(NAME, (S_IFREG|(MODE)),            \
        NULL, &proc_single_file_operations,    \
        { .proc_show = show } )
#define ATTR(LSM, NAME, MODE)                \
    NOD(NAME, (S_IFREG|(MODE)),            \
        NULL, &proc_pid_attr_operations,    \
        { .lsm = LSM })
```

因此，一个文件的所有属性，或者说状态信息，都被抽象为了一个个子文件，或者子文件夹，也就可以使用 pid_entry 数组来表示这个结构

```c
static const struct pid_entry tid_base_stuff[] = {
    DIR("fd",        S_IRUSR|S_IXUSR, proc_fd_inode_operations, proc_fd_operations),
    DIR("fdinfo",    S_IRUSR|S_IXUSR, proc_fdinfo_inode_operations, proc_fdinfo_operations),
    DIR("ns",     S_IRUSR|S_IXUGO, proc_ns_dir_inode_operations, proc_ns_dir_operations),
#ifdef CONFIG_NET
    DIR("net",        S_IRUGO|S_IXUGO, proc_net_inode_operations, proc_net_operations),
#endif
    REG("environ",   S_IRUSR, proc_environ_operations),
    REG("auxv",      S_IRUSR, proc_auxv_operations),
    ONE("status",    S_IRUGO, proc_pid_status),
    ONE("personality", S_IRUSR, proc_pid_personality),
    ONE("limits",     S_IRUGO, proc_pid_limits),
#ifdef CONFIG_SCHED_DEBUG
    REG("sched",     S_IRUGO|S_IWUSR, proc_pid_sched_operations),
#endif
    NOD("comm",      S_IFREG|S_IRUGO|S_IWUSR,
             &proc_tid_comm_inode_operations,
             &proc_pid_set_comm_operations, {}),
#ifdef CONFIG_HAVE_ARCH_TRACEHOOK
    ONE("syscall",   S_IRUSR, proc_pid_syscall),
#endif
    REG("cmdline",   S_IRUGO, proc_pid_cmdline_ops),
    ONE("stat",      S_IRUGO, proc_tid_stat),
    ONE("statm",     S_IRUGO, proc_pid_statm),
    // ......
}
```

相信聪明的你们已经可以举一反三了，如果是 `REG` 宏表示的 regular file，设定好 file_operations 之后，就可以通过 seq_file 接口和 `open/read` 等系统调用交互了，如果是 `DIR` 宏表示的 directory file，还需要提供新的 inode_operations，如果下一级还有目录，再提供一个 inode_operations 即可 ......
