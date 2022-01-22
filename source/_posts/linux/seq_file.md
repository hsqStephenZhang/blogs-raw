---
title: Seq File
categories: linux
date: 2022-1-19 16:49:40
---

Seq File 在 Linux 中很常见，因此这里稍微整理一下，也分享一下我的理解。

由于 procfs 的默认操作函数只使用一页的缓存，在处理较大的proc文件时就有点麻烦，并且在输出一系列结构体中的数据时也比较不灵活，因此内核开发者们抽象出了seq_file（Sequence file：序列文件）接口。 提供了一套简单的函数来解决以上 procfs 编程时存在的问题，使得编程更加容易。

同时也需要注意，虽然 seq_file 最一开始是为了方便 procfs 使用，但大家意识到，这玩意很 nice 啊，于是已经渗透到很多别的虚拟文件系统种，比如 debugfs，sysfs，都有很多的应用。

## 1. API

```text
struct seq_file {
    char *buf; // 序列文件对应的数据缓冲区，要导出的数据是首先打印到这个缓冲区，然后才被拷贝到指定的用户缓冲区。
    size_t size;  // 缓冲区大小，默认为1个页面大小，随着需求会动态以2的级数倍扩张，4k,8k,16k...
    size_t from;  // 没有拷贝到用户空间的数据在 buf 中的起始偏移量
    size_t count; // buf中没有拷贝到用户空间的数据的字节数，调用 seq_printf() 等函数向buf写数据的同时相应增加m->count
    size_t pad_until; // useless
    loff_t index;  // 正在或即将读取的数据项索引，和 seq_operations 中的start、next 操作中的 pos 项一致，一条记录为一个索引
    loff_t read_pos;  // 当前读取数据（file）的偏移量，字节为单位
    u64 version;  // 文件的版本
    struct mutex lock;  // 序列化对这个文件的并行操作
    const struct seq_operations *op;  // 指向seq_operations
    int poll_event; 
    const struct file *file; // seq_file 相关的 proc 或其他文件
    void *private;  //指向文件的私有数据
};

seq_open：通常会在打开文件的时候调用，以第二个参数为 seq_operations 表创建 seq_file 结构体。
seq_read, seq_lseek，seq_release：直接对应着文件操作表中的 read, llseek 和 release。
seq_escape：将一个字符串中的需要转义的字符（字节长）以8进制的方式打印到 seq_file。
seq_putc, seq_puts, seq_printf：分别和C语言中的 putc，puts 和 printf 相对应。
seq_path：输出文件名
single_open, single_release: 打开和释放只有一条记录的文件。
seq_open_private, __seq_open_private, seq_release_private：和 seq_open 类似，不过打开 seq_file 的时候创建一小块文件私有数据，这个 private 数据会保存在 file->private 字段中，之后读取或者写入的时候，会有其特定的作用

```

## 2. procfs 和 seq_file 结合

以 `/proc/kallsyms` 这个文件为例，演示一下 procfs 和 seq_file 是如何结合在一起的。

先了解一下 `/proc/kallsyms` 的作用，使用 `sudo cat /proc/kallsyms` 命令，就可以查看这个**虚拟文件**的内容，发现是很多符号，以及符号对应的地址，其实就是目前正在运行的 kernel 的符号信息。这些信息显然不会保存在磁盘当中，是随着系统运行变化的，自然需要通过虚拟文件暴露给用户。其读取正是用到了 seq_file 以及 seq_read 一系列接口

### 2.1 kallsyms 初始化

`/proc/kallsyms` 这个文件，使用的也是 procfs 提供的 proc_create 接口进行创建的

```c
// kernel/kallsyms.c
static const struct proc_ops kallsyms_proc_ops = {
    .proc_open  = kallsyms_open,
    .proc_read  = seq_read,
    .proc_lseek = seq_lseek,
    .proc_release   = seq_release_private,
};

static int __init kallsyms_init(void)
{
    proc_create("kallsyms", 0444, NULL, &kallsyms_proc_ops);
    return 0;
}
```

直接看这里的 `kallsyms_proc_ops` 结构，`proc_read` `proc_lseek` `proc_release` 的回调函数都很普通，关键自然在于 `proc_open` 对应的 `kallsyms_open`，通过这个名字也不难得知，通过 open 系统调用打开 `/proc/kallsyms` 文件的时候，内部则会调用该回调函数进行处理。

那么，关注一下这里的 `kallsyms_open`，内部是调用了 `__seq_open_private` 创建了一个 iter，联系上面 API 说明可知，在打开这个文件的时候，设置了一个 private 字段，用来辅助后面的 read write 操作，而这里对应的就应该是 kallsym_iter 结构了

```c
static int kallsyms_open(struct inode *inode, struct file *file)
{
    struct kallsym_iter *iter;
    iter = __seq_open_private(file, &kallsyms_op, sizeof(*iter));
    if (!iter)
        return -ENOMEM;
    reset_iter(iter, 0);

    iter->show_value = kallsyms_show_value(file->f_cred);
    return 0;
}
```

可以看出，`__seq_open_private` 没什么神奇的，就是分配了 private 所需要的空间，并且将 file->private 指向该 private 字段，然后返回给外层的调用者。

有了这个保存在 file->private 中的 iter，我们就赋予了 seq_read 一个**状态**。

这点我觉得非常重要，因为在遍历 kernel symbols 的过程中，必须要清楚当前读取到了哪一步，或者说当前的状态是什么样的。对于有一些静态信息来说，则无关紧要，不需要保存**状态**，也就不用通过 `__seq_open_private` 来打开文件了。

与之对应，在 release 文件的时候，需要销毁这个额外分配出来的 private 字段，因此，proc_release 对应的是 `seq_release_private`，而非普通的 `seq_release`。

并且 `__seq_open_private` 传入了一个很重要的参数：`kallsyms_op`，这就是遍历并读取 seq_file 的几个核心 API，看上去非常简单，事实上也正是如此，通过 `start, next, stop, show` 的配合，决定该 seq_file 的读取如何开始，何时结束，如何展示，这就是典型的内核提供的机制，**用户**提供策略（当然不是普通的用户），填充好回调函数，就能按照机制走流程，在特定的位置上调用这些回调函数即可。

```c
static const struct seq_operations kallsyms_op = {
    .start = s_start,
    .next = s_next,
    .stop = s_stop,
    .show = s_show
};
```

```c
void *__seq_open_private(struct file *f, const struct seq_operations *ops,
        int psize)
{
    int rc;
    void *private;
    struct seq_file *seq;

    private = kzalloc(psize, GFP_KERNEL_ACCOUNT);
    if (private == NULL)
        goto out;

    rc = seq_open(f, ops);
    if (rc < 0)
        goto out_free;

    seq = f->private_data;
    seq->private = private;
    return private;

out_free:
    kfree(private);
out:
    return NULL;
}
EXPORT_SYMBOL(__seq_open_private);
```

### 2.2. seq_read API

`seq_operations` 中的 `start, next, stop, show` 最终都是要在读取文件的时候起作用的，seq_file 的读取文件的函数为 `seq_read`，在作用了一些简单的初始化操作之后，比如分配 buf，设置好 size，紧接着就调用了 `seq_read_iter`，这才是读取文件的核心 API

```c
ssize_t seq_read(struct file *file, char __user *buf, size_t size, loff_t *ppos)
{
    struct iovec iov = { .iov_base = buf, .iov_len = size};
    struct kiocb kiocb;
    struct iov_iter iter;
    ssize_t ret;

    init_sync_kiocb(&kiocb, file);
    iov_iter_init(&iter, READ, &iov, 1, size);

    kiocb.ki_pos = *ppos;
    ret = seq_read_iter(&kiocb, &iter);
    *ppos = kiocb.ki_pos;
    return ret;
}
EXPORT_SYMBOL(seq_read);
```

`seq_read_iter` 中，省去一些细枝末节的部分，就是下面这样的结构，先调用 `op->start(...)`，然后在一个 while 循环中，通过 `op->show()` 进行展示，通过 `op->next()` 进行迭代，直到 **EOF** 或者发生了错误，这才跳出循环，调用 `op->stop()`，同时，循环中还需要注意 m->buf 是否超出了原先分配的大小，如果超出了，需要重新分配缓冲，并且调用 `op->start()` **重启**

```c
ssize_t seq_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    // ....
    m->from = 0;
    p = m->op->start(m, &m->index);
    while (1) {
        err = PTR_ERR(p);
        if (!p || IS_ERR(p))    // EOF or an error
            break;
        err = m->op->show(m, p);
        if (err < 0)        // hard error
            break;
        if (unlikely(err))    // ->show() says "skip it"
            m->count = 0;
        if (unlikely(!m->count)) { // empty record
            p = m->op->next(m, p, &m->index);
            continue;
        }
        if (!seq_has_overflowed(m)) // got it
            goto Fill;
        // need a bigger buffer
        m->op->stop(m, p);
        kvfree(m->buf);
        m->count = 0;
        m->buf = seq_buf_alloc(m->size <<= 1);
        if (!m->buf)
            goto Enomem;
        p = m->op->start(m, &m->index);
    }
    // EOF or an error
    m->op->stop(m, p);
    m->count = 0;
    goto Done;
    // ...
    copied = m->count ? -EFAULT : err;
    return copied;
}
EXPORT_SYMBOL(seq_read_iter);
```

而所有输出的内容，其实都会保存在 m->buf 当中，最后一次性从内核空间返回给读取该文件的用户

### 2.3 kallsyms 如何迭代

现在我们已经基本了解了 seq_file 是如何运转的了，接下俩接着解决 kallsyms 的问题，我们还剩下一个问题没有讨论：`seq_operations` 中回调函数的实现细节

可以看出，`s_next s_start` 都是调用了 `update_iter` 进行调用，如果返回为 0，表明遇到了 EOF 或者发生了错误，直接返回 NULL 就好了，这样在 `seq_read_iter` 当中，就能及时跳出循环。

而这里的 `s_stop` 什么都没做，其实也没有什么好做的，并没有什么资源啊的等待释放。

`s_show` 给人一种亲切感，因为这个函数中的 `seq_printf` 输出的格式，不就是执行 `sudo cat /proc/kallsyms` 得到的单行的文件内容吗！

```c
static void *s_next(struct seq_file *m, void *p, loff_t *pos)
{
    (*pos)++;

    if (!update_iter(m->private, *pos))
        return NULL;
    return p;
}

static void *s_start(struct seq_file *m, loff_t *pos)
{
    if (!update_iter(m->private, *pos))
        return NULL;
    return m->private;
}

static void s_stop(struct seq_file *m, void *p)
{
}

static int s_show(struct seq_file *m, void *p)
{
    void *value;
    struct kallsym_iter *iter = m->private;

    /* Some debugging symbols have no name.  Ignore them. */
    if (!iter->name[0])
        return 0;

    value = iter->show_value ? (void *)iter->value : NULL;

    if (iter->module_name[0]) {
        char type;

        /*
         * Label it "global" if it is exported,
         * "local" if not exported.
         */
        type = iter->exported ? toupper(iter->type) :
                    tolower(iter->type);
        seq_printf(m, "%px %c %s\t[%s]\n", value,
               type, iter->name, iter->module_name);
    } else
        seq_printf(m, "%px %c %s\n", value,
               iter->type, iter->name);
    return 0;
}
```

那么，`update_iter` 做了什么呢？就留给读者自己去研究吧！

## 3. 自己写一个 proc seeq_file

了解 seq_file 的使用方式之后，自己来写一个类似 kallsyms 的模块也是轻而易举的事情了。

如下所示，是输出当前的所有进程，同时更新 offset 这个变量（其实就是 line number），除了 seq_file 和 procfs 的处理以外，还用到了 `init_task` 这个全局变量，是系统初始化时候创建的进程，同时，这些进程都以链表的方式存放，可以调用 `next_task` 获取下一个进程

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/percpu.h>
#include <linux/init.h>
#include <linux/sched.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Girish Joshi");
MODULE_DESCRIPTION("HelloWorld Linux Kernel Module.");

static struct proc_dir_entry *entry;
static loff_t offset = 1;

static void *l_start(struct seq_file *m, loff_t * pos)
{
    loff_t index = *pos;
    loff_t i = 0;
    struct task_struct * task ;

    if (index == 0) {
        seq_printf(m, "Current all the processes in system:\n"
                "%-24s%-5s%-5s\n", "name", "pid","index");
        printk(KERN_EMERG "++++++++++=========>%5d\n", 0);
        return &init_task;
    }else {
        for(i = 0, task=&init_task; i < index; i++){
            task = next_task(task);    
        }
        BUG_ON(i != *pos);
        if(task == &init_task){
            return NULL;
        }

        printk(KERN_EMERG "++++++++++>%5d\n", task->pid);
        return task;
    }
}

static void *l_next(struct seq_file *m, void *p, loff_t * pos)
{
    struct task_struct * task = (struct task_struct *)p;

    task = next_task(task);
    if ((*pos != 0) && (task == &init_task)) {
        return NULL;
    }

    printk(KERN_EMERG "=====>%5d\n", task->pid);
    offset = ++(*pos);

    return task;
}

static void l_stop(struct seq_file *m, void *p)
{
    printk(KERN_EMERG "------>\n");
}

static int l_show(struct seq_file *m, void *p)
{
    struct task_struct * task = (struct task_struct *)p;

    seq_printf(m, "%-24s%-5d\t%-5lld\n", task->comm, task->pid, offset);
    return 0;
}

// for se_file traverse
static struct seq_operations kallprocesses_seq_fops = {
    .start = l_start,
    .next  = l_next,
    .stop  = l_stop,
    .show  = l_show
};

static int kallprocesses_open(struct inode *inode, struct file *file)
{
        seq_open(file, &kallprocesses_seq_fops);
        return 0;
}

// for proc fs's file operations
static const struct proc_ops kallprocesses_proc_ops = {
        .proc_open      = kallprocesses_open,
        .proc_read      = seq_read,
        .proc_lseek     = seq_lseek,
        .proc_release   = seq_release,
};

static int __init hello_init(void)
{
    entry = proc_create("kallprocesses", 0444, NULL, &kallprocesses_proc_ops);
    printk(KERN_INFO "Hello world!\n");
    return 0;    // Non-zero return means that the module couldn't be loaded.
}

static void __exit hello_cleanup(void)
{
    printk(KERN_INFO "Cleaning up module.\n");
}

module_init(hello_init);
module_exit(hello_cleanup);
```
