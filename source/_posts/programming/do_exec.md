---
title: do_exec 源码解析
---

大家都知道，Linux 下面的可执行文件格式为 elf ，全称是 executable linkable format，中文名为“可执行可链接文件”，这种文件是以二进制的方式存储的，我们也可以将其称为二进制对象（binary object）。

以 gcc 为例，最终需要将很多 '.c' 文件编译为一个 elf 格式的可执行文件，最终才能在 Linux 上运行，那么问题来了，Linux 是如何识别 elf 这种格式，又是如何将其和**进程**结合在一起的呢？接下来我们来研究研究。

## 1. do_execv

### 1.1 strace 大法好

以 shell 为例，只需要指定路径+参数，系统就能执行这个程序了，但这背后就是一个黑箱，摸不着也猜不透，那有什么办法可以一窥究竟呢？strace！

strace 是追踪系统调用的利器，只有明确了你想要的功能是通过哪个系统调用暴露出去的，才不至于在查阅源码的时候像一个无头苍蝇。

通过 `strace ls` 这条指令，得到了下面的输出

```text
execve("/usr/bin/ls", ["ls"], 0x7ffd2dbd43d0 /* 40 vars */) = 0
......
```

最关键的信息是 `execve()`，对应的系统调用为 sys_execve，这就是执行程序的第一步：将可执行程序的`路径+参数`传递给 sys_execve 系统调用

其实在这之前，shell 也为我们做了很多准备工作：

1. 读取并且解析命令
2. 读取 shell 的环境变量
3. 读取 `.bashrc/.zshrc` 等配置文件
4. 依次在 `PATH` 环境变量表示的路径中，查找是否有对应的可执行文件（shell 的内置指令不予讨论）

最终才会调用 `execve(command, args, env);` 这个系统调用

### 1.2 sys_execve

`sys_execve` 接收三个参数，其含义还是非常清晰的，分别是：可执行程序路径，参数，环境变量

`sys_execve` 最终调用了 `do_execveat_common` 这个核心处理函数

```c
// fs/exec.c
SYSCALL_DEFINE3(execve,
        const char __user *, filename,
        const char __user *const __user *, argv,
        const char __user *const __user *, envp)
{
    return do_execve(getname(filename), argv, envp);
}

static int do_execve(struct filename *filename,
    const char __user *const __user *__argv,
    const char __user *const __user *__envp)
{
    struct user_arg_ptr argv = { .ptr.native = __argv };
    struct user_arg_ptr envp = { .ptr.native = __envp };
    return do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
}
```

### 1.3 do_execveat_common

`do_execveat_common` 中，还是做了一些准备工作，比如验证 rlimit 是否超出上限，argv 和 envp 是否超出上限，stack 是否超出上限。

一系列检查无误之后，就将参数从用户空间拷贝到内核空间，将父进程的一些环境变量拷贝过来，并且执行 `bprm_execve`

```c
static int do_execveat_common(int fd, struct filename *filename,
                  struct user_arg_ptr argv,
                  struct user_arg_ptr envp,
                  int flags)
{
    struct linux_binprm *bprm;
    int retval;

    if (IS_ERR(filename))
        return PTR_ERR(filename);

    // 1. 检查 rlimit 是否超出上限
    if ((current->flags & PF_NPROC_EXCEEDED) &&
        atomic_read(&current_user()->processes) > rlimit(RLIMIT_NPROC)) {
        retval = -EAGAIN;
        goto out_ret;
    }

    current->flags &= ~PF_NPROC_EXCEEDED;

    // 2. 分配 binary program 结构体
    bprm = alloc_bprm(fd, filename);
    if (IS_ERR(bprm)) {
        retval = PTR_ERR(bprm);
        goto out_ret;
    }

    // 3. argv 和 envp 不能超出上限(其实没有意义)
    // #define MAX_ARG_STRINGS 0x7FFFFFFF
    retval = count(argv, MAX_ARG_STRINGS);
    if (retval < 0)
        goto out_free;
    bprm->argc = retval;

    retval = count(envp, MAX_ARG_STRINGS);
    if (retval < 0)
        goto out_free;
    bprm->envc = retval;

    // 4. 还是通过 rlimit 检查 stack 是否超出上限
    retval = bprm_stack_limits(bprm);
    if (retval < 0)
        goto out_free;

    // 5. 将 参数/环境变量 从内核空间拷贝到用户空间
    retval = copy_string_kernel(bprm->filename, bprm);
    if (retval < 0)
        goto out_free;
    bprm->exec = bprm->p;

    // 6. 将原先进程的 参数/环境变量 拷贝到新的进程
    retval = copy_strings(bprm->envc, envp, bprm);
    if (retval < 0)
        goto out_free;

    retval = copy_strings(bprm->argc, argv, bprm);
    if (retval < 0)
        goto out_free;

    // 7. 执行 binary program
    retval = bprm_execve(bprm, fd, filename, flags);
out_free:
    free_bprm(bprm);

out_ret:
    putname(filename);
    return retval;
}
```

### 1.4 bprm_execve

到了 `bprm_execve` 就需要做一些真正有空的操作了，比如：检查 credentials，打开可执行程序文件，检查路径正确性。然后会跳转到 `exec_binprm`

```c
static int bprm_execve(struct linux_binprm *bprm,
               int fd, struct filename *filename, int flags)
{
    struct file *file;
    struct files_struct *displaced;
    int retval;

    // 1. 取消任何跨越 execve 的 io_uring task
    io_uring_task_cancel();

    // 2. 设置文件描述符不共享
    retval = unshare_files(&displaced);
    if (retval)
        return retval;

    // 3. 检查 binary program 的凭据
    retval = prepare_bprm_creds(bprm);
    if (retval)
        goto out_files;

    check_unsafe_exec(bprm);
    current->in_execve = 1;

    // 4. 打开文件
    file = do_open_execat(fd, filename, flags);
    retval = PTR_ERR(file);
    if (IS_ERR(file))
        goto out_unmark;

    sched_exec();

    // 5. 检查路径是否正确
    bprm->file = file;
    if (bprm->fdpath &&
        close_on_exec(fd, rcu_dereference_raw(current->files->fdt)))
        bprm->interp_flags |= BINPRM_FLAGS_PATH_INACCESSIBLE;

    // 6. 检查 security
    retval = security_bprm_creds_for_exec(bprm);
    if (retval)
        goto out;

    // 7. 执行 binary program
    retval = exec_binprm(bprm);
    if (retval < 0)
        goto out;

    // 8. 执行成功，释放资源
    ...
}
```

### 1.5 exec_binprm

`exec_binprm` 中有一个非常重要的循环，在循环当中，调用了 `search_binary_handler`

往下看就知道

看到这里你应该会有一个疑问：什么叫 binary handler？下面会详细解释

```c
static int exec_binprm(struct linux_binprm *bprm)
{
    pid_t old_pid, old_vpid;
    int ret, depth;

    // 1. 获取旧的 pid
    old_pid = current->pid;
    rcu_read_lock();
    old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
    rcu_read_unlock();

    // 2. 允许最多4层的 binary format 改写
    for (depth = 0;; depth++) {
        struct file *exec;
        if (depth > 5)
            return -ELOOP;

        // 2.1 查找 binary handler，并且通过 binary handler 来执行文件
        ret = search_binary_handler(bprm);
        if (ret < 0)
            return ret;
        if (!bprm->interpreter)
            break;

        // 2.2 成功找到 interpreter
        exec = bprm->file;
        bprm->file = bprm->interpreter;
        bprm->interpreter = NULL;

        allow_write_access(exec);
        if (unlikely(bprm->have_execfd)) {
            if (bprm->executable) {
                fput(exec);
                return -ENOEXEC;
            }
            bprm->executable = exec;
        } else
            fput(exec);
    }

    audit_bprm(bprm);
    trace_sched_process_exec(current, old_pid, bprm);
    ptrace_event(PTRACE_EVENT_EXEC, old_vpid);
    proc_exec_connector(current);
    return 0;
}
```

## 2. search_binary_handler

众所周知，Linux 中策略和机制分离。换句话说，Linux 提供框架，而具体的机制由开发者来决定（当然也经常会提供默认的机制），binary handler 就是一个很鲜明的例子

虽然绝大多数的可执行程序格式为 elf，但还有各种各样的脚本啊，还有比较旧的可执行格式需要兼容，甚至可能还有更加现代化的格式...... 每一种格式的文件，都可以制定自己的执行机制（也就是各种回调函数）

再来看 `search_binary_handler` 就不难理解了：查找并执行对应格式的处理函数（binary handler）。

自然，遍历注册到系统中的 `binary format` 链表是最常见，也是最容易实现的方式。依次判定是否符合该格式，如果符合，就可以调用其特定的 `load_binary` 回调函数进行加载，如果所有格式的无法识别，本次 execve 也就失败了

Linux 支持下面的 binary format：

1. binfmt_script - 支持需要解释器执行的脚本，文件以 "#!" 开头
2. binfmt_misc - 根据内核的运行时配置，支持不同的二进制格式
3. binfmt_elf - 支持 elf 格式
4. binfmt_aout - 支持 a.out 格式
5. binfmt_flat - 支持 flat 格式
6. binfmt_elf_fdpic - 支持 elf FDPIC 格式;
7. binfmt_em86 - 支持在 Alpha 架构上运行 intel 的 elf 二进制文件

```c
static int search_binary_handler(struct linux_binprm *bprm)
{
    bool need_retry = IS_ENABLED(CONFIG_MODULES);
    struct linux_binfmt *fmt;
    int retval;

    retval = prepare_binprm(bprm);
    if (retval < 0)
        return retval;

    retval = security_bprm_check(bprm);
    if (retval)
        return retval;

    retval = -ENOENT;
 retry:

    // 遍历 binary format 链表，查找并调用相应的 load_binary 处理函数
    read_lock(&binfmt_lock);
    list_for_each_entry(fmt, &formats, lh) {
        if (!try_module_get(fmt->module))
            continue;
        read_unlock(&binfmt_lock);

        retval = fmt->load_binary(bprm);

        read_lock(&binfmt_lock);
        put_binfmt(fmt);
        if (bprm->point_of_no_return || (retval != -ENOEXEC)) {
            read_unlock(&binfmt_lock);
            return retval;
        }
    }
    read_unlock(&binfmt_lock);

    if (need_retry) {
        if (printable(bprm->buf[0]) && printable(bprm->buf[1]) &&
            printable(bprm->buf[2]) && printable(bprm->buf[3]))
            return retval;
        if (request_module("binfmt-%04x", *(ushort *)(bprm->buf + 2)) < 0)
            return retval;
        need_retry = false;
        goto retry;
    }

    return retval;
}
```

接下来我们需要关注两点：

1. 如何匹配可执行文件格式
2. 如何载入可执行文件

其实，Linux 中将这两个步骤合并到了 `load_binary` 中，若 load_binary 返回 `-ENOEXEC`，表明格式不匹配，就遍历剩下的可执行文件格式，如果格式匹配，就可以完成剩下的载入工作。

### 2.1 load_binary

有两种最为大家熟知的执行文件的方式：

1. 由 gcc 这样的编译器直接编译为 elf 格式的文件
2. 在文件开头添加 `#! /bin/bash` 或者 `#! /usr/bin/python3` 的脚本

后者对应 `load_script`，比较简单，就当作大餐前的小点心了

该函数中，首先是比较 bprm->buf 的开头两个字母是否为 `#!`，如果匹配成功，表明这是一个脚本，需要找到对应的解释器来执行，具体哪个解释器呢？就需要后面的 `/bin/bash` 或者 `/usr/bin/python3` 来指定

```c
static int load_script(struct linux_binprm *bprm)
{
    const char *i_name, *i_sep, *i_arg, *i_end, *buf_end;
    struct file *file;
    int retval;

    /* Not ours to exec if we don't start with "#!". */
    if ((bprm->buf[0] != '#') || (bprm->buf[1] != '!'))
        return -ENOEXEC;
    ...
}
```

值得注意的是，在解释器的路径之后，还可以跟上对应的参数，load_script 函数都会解析出来，例如：`#! /usr/bin/env python3`

### 2.2 load_elf_binary

接下来是我们的重头戏：load_elf_binary

```c
static int load_elf_binary(struct linux_binprm *bprm)
{
    // 1. 初始化参数
    struct file *interpreter = NULL; /* to shut gcc up */
     unsigned long load_addr = 0, load_bias = 0;
    int load_addr_set = 0;
    unsigned long error;
    struct elf_phdr *elf_ppnt, *elf_phdata, *interp_elf_phdata = NULL;
    struct elf_phdr *elf_property_phdata = NULL;
    unsigned long elf_bss, elf_brk;
    int bss_prot = 0;
    int retval, i;
    unsigned long elf_entry;
    unsigned long e_entry;
    unsigned long interp_load_addr = 0;
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long reloc_func_desc __maybe_unused = 0;
    int executable_stack = EXSTACK_DEFAULT;
    struct elfhdr *elf_ex = (struct elfhdr *)bprm->buf;
    struct elfhdr *interp_elf_ex = NULL;
    struct arch_elf_state arch_state = INIT_ARCH_ELF_STATE;
    struct mm_struct *mm;
    struct pt_regs *regs;

    retval = -ENOEXEC;
    /* 比较文件头的前四个字节，查看是否为 elf 魔数 */
    if (memcmp(elf_ex->e_ident, ELFMAG, SELFMAG) != 0)
        goto out;

    if (elf_ex->e_type != ET_EXEC && elf_ex->e_type != ET_DYN)
        goto out;
    if (!elf_check_arch(elf_ex))
        goto out;
    if (elf_check_fdpic(elf_ex))
        goto out;
    if (!bprm->file->f_op->mmap)
        goto out;

    // 2. 加载程序头表，通过 kernel_read 读入整个 program header table
    elf_phdata = load_elf_phdrs(elf_ex, bprm->file);
    if (!elf_phdata)
        goto out;

    elf_ppnt = elf_phdata;

    // 3. 寻找和处理 .interpreter 段
    // 解释器段的类型为 PT_INTERP，找到之后读入缓冲区
    for (i = 0; i < elf_ex->e_phnum; i++, elf_ppnt++) {
        char *elf_interpreter;

        if (elf_ppnt->p_type == PT_GNU_PROPERTY) {
            elf_property_phdata = elf_ppnt;
            continue;
        }

        if (elf_ppnt->p_type != PT_INTERP)
            continue;

        /*
         * This is the program interpreter used for shared libraries -
         * for now assume that this is an a.out format binary.
         */
        retval = -ENOEXEC;
        if (elf_ppnt->p_filesz > PATH_MAX || elf_ppnt->p_filesz < 2)
            goto out_free_ph;

        retval = -ENOMEM;
        elf_interpreter = kmalloc(elf_ppnt->p_filesz, GFP_KERNEL);
        if (!elf_interpreter)
            goto out_free_ph;

        // 解释器段其实只是一个字符串，内容为解释器的文件绝对路径
        retval = elf_read(bprm->file, elf_interpreter, elf_ppnt->p_filesz,
                  elf_ppnt->p_offset);
        if (retval < 0)
            goto out_free_interp;
        /* make sure path is NULL terminated */
        retval = -ENOEXEC;
        if (elf_interpreter[elf_ppnt->p_filesz - 1] != '\0')
            goto out_free_interp;

        // 有了解释器的路径之后，就可以打开这个文件
        interpreter = open_exec(elf_interpreter);
        kfree(elf_interpreter);
        retval = PTR_ERR(interpreter);
        if (IS_ERR(interpreter))
            goto out_free_ph;

        /*
         * If the binary is not readable then enforce mm->dumpable = 0
         * regardless of the interpreter's permissions.
         */
        would_dump(bprm, interpreter);

        interp_elf_ex = kmalloc(sizeof(*interp_elf_ex), GFP_KERNEL);
        if (!interp_elf_ex) {
            retval = -ENOMEM;
            goto out_free_ph;
        }

        // 通过 kernel_read 读入 interpreter 的头部
        retval = elf_read(interpreter, interp_elf_ex,
                  sizeof(*interp_elf_ex), 0);
        if (retval < 0)
            goto out_free_dentry;

        break;

out_free_interp:
        kfree(elf_interpreter);
        goto out_free_ph;
    }

    elf_ppnt = elf_phdata;
    for (i = 0; i < elf_ex->e_phnum; i++, elf_ppnt++)
        switch (elf_ppnt->p_type) {
        case PT_GNU_STACK:
            if (elf_ppnt->p_flags & PF_X)
                executable_stack = EXSTACK_ENABLE_X;
            else
                executable_stack = EXSTACK_DISABLE_X;
            break;

        case PT_LOPROC ... PT_HIPROC:
            retval = arch_elf_pt_proc(elf_ex, elf_ppnt,
                          bprm->file, false,
                          &arch_state);
            if (retval)
                goto out_free_dentry;
            break;
        }

    // 4. 检查并读取解释器的 program header table
    if (interpreter) {
        retval = -ELIBBAD;
        /* Not an ELF interpreter */
        if (memcmp(interp_elf_ex->e_ident, ELFMAG, SELFMAG) != 0)
            goto out_free_dentry;
        /* Verify the interpreter has a valid arch */
        if (!elf_check_arch(interp_elf_ex) ||
            elf_check_fdpic(interp_elf_ex))
            goto out_free_dentry;

        // 载入 interpreter program headers
        interp_elf_phdata = load_elf_phdrs(interp_elf_ex,
                           interpreter);
        if (!interp_elf_phdata)
            goto out_free_dentry;

        // 暂时忽略
        elf_property_phdata = NULL;
        elf_ppnt = interp_elf_phdata;
        for (i = 0; i < interp_elf_ex->e_phnum; i++, elf_ppnt++)
            switch (elf_ppnt->p_type) {
            case PT_GNU_PROPERTY:
                elf_property_phdata = elf_ppnt;
                break;

            case PT_LOPROC ... PT_HIPROC:
                retval = arch_elf_pt_proc(interp_elf_ex,
                              elf_ppnt, interpreter,
                              true, &arch_state);
                if (retval)
                    goto out_free_dentry;
                break;
            }
    }

    retval = parse_elf_properties(interpreter ?: bprm->file,
                      elf_property_phdata, &arch_state);
    if (retval)
        goto out_free_dentry;

    retval = arch_check_elf(elf_ex,
                !!interpreter, interp_elf_ex,
                &arch_state);
    if (retval)
        goto out_free_dentry;

    // 清除父进程的所有代码
    retval = begin_new_exec(bprm);
    if (retval)
        goto out_free_dentry;

    // 设置 elf 可执行文件的特性
    SET_PERSONALITY2(*elf_ex, &arch_state);
    if (elf_read_implies_exec(*elf_ex, executable_stack))
        current->personality |= READ_IMPLIES_EXEC;

    if (!(current->personality & ADDR_NO_RANDOMIZE) && randomize_va_space)
        current->flags |= PF_RANDOMIZE;

    setup_new_exec(bprm);

    // 为下面的动态连接器执行获取内核空间 page
    retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
                 executable_stack);
    if (retval < 0)
        goto out_free_dentry;
    
    elf_bss = 0;
    elf_brk = 0;

    start_code = ~0UL;
    end_code = 0;
    start_data = 0;
    end_data = 0;

    // 5. 装入目标程序的段
    for(i = 0, elf_ppnt = elf_phdata;
        i < elf_ex->e_phnum; i++, elf_ppnt++) {
        int elf_prot, elf_flags;
        unsigned long k, vaddr;
        unsigned long total_size = 0;
        unsigned long alignment;

        if (elf_ppnt->p_type != PT_LOAD)
            continue;

        if (unlikely (elf_brk > elf_bss)) {
            unsigned long nbyte;
                
            /* There was a PT_LOAD segment with p_memsz > p_filesz
               before this one. Map anonymous pages, if needed,
               and clear the area.  */
            retval = set_brk(elf_bss + load_bias,
                     elf_brk + load_bias,
                     bss_prot);
            if (retval)
                goto out_free_dentry;
            nbyte = ELF_PAGEOFFSET(elf_bss);
            if (nbyte) {
                nbyte = ELF_MIN_ALIGN - nbyte;
                if (nbyte > elf_brk - elf_bss)
                    nbyte = elf_brk - elf_bss;
                if (clear_user((void __user *)elf_bss +
                            load_bias, nbyte)) {
                    /*
                     * This bss-zeroing can fail if the ELF
                     * file specifies odd protections. So
                     * we don't check the return value
                     */
                }
            }
        }

        elf_prot = make_prot(elf_ppnt->p_flags, &arch_state,
                     !!interpreter, false);

        elf_flags = MAP_PRIVATE | MAP_DENYWRITE | MAP_EXECUTABLE;

        vaddr = elf_ppnt->p_vaddr;
        if (elf_ex->e_type == ET_EXEC || load_addr_set) {
            elf_flags |= MAP_FIXED;
        } else if (elf_ex->e_type == ET_DYN) {
            if (interpreter) {
                load_bias = ELF_ET_DYN_BASE;
                if (current->flags & PF_RANDOMIZE)
                    load_bias += arch_mmap_rnd();
                alignment = maximum_alignment(elf_phdata, elf_ex->e_phnum);
                if (alignment)
                    load_bias &= ~(alignment - 1);
                elf_flags |= MAP_FIXED;
            } else
                load_bias = 0;

            load_bias = ELF_PAGESTART(load_bias - vaddr);

            total_size = total_mapping_size(elf_phdata,
                            elf_ex->e_phnum);
            if (!total_size) {
                retval = -EINVAL;
                goto out_free_dentry;
            }
        }

        error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
                elf_prot, elf_flags, total_size);
        if (BAD_ADDR(error)) {
            retval = IS_ERR((void *)error) ?
                PTR_ERR((void*)error) : -EINVAL;
            goto out_free_dentry;
        }

        if (!load_addr_set) {
            load_addr_set = 1;
            load_addr = (elf_ppnt->p_vaddr - elf_ppnt->p_offset);
            if (elf_ex->e_type == ET_DYN) {
                load_bias += error -
                             ELF_PAGESTART(load_bias + vaddr);
                load_addr += load_bias;
                reloc_func_desc = load_bias;
            }
        }
        k = elf_ppnt->p_vaddr;
        if ((elf_ppnt->p_flags & PF_X) && k < start_code)
            start_code = k;
        if (start_data < k)
            start_data = k;

        /*
         * Check to see if the section's size will overflow the
         * allowed task size. Note that p_filesz must always be
         * <= p_memsz so it is only necessary to check p_memsz.
         */
        if (BAD_ADDR(k) || elf_ppnt->p_filesz > elf_ppnt->p_memsz ||
            elf_ppnt->p_memsz > TASK_SIZE ||
            TASK_SIZE - elf_ppnt->p_memsz < k) {
            /* set_brk can never work. Avoid overflows. */
            retval = -EINVAL;
            goto out_free_dentry;
        }

        k = elf_ppnt->p_vaddr + elf_ppnt->p_filesz;

        if (k > elf_bss)
            elf_bss = k;
        if ((elf_ppnt->p_flags & PF_X) && end_code < k)
            end_code = k;
        if (end_data < k)
            end_data = k;
        k = elf_ppnt->p_vaddr + elf_ppnt->p_memsz;
        if (k > elf_brk) {
            bss_prot = elf_prot;
            elf_brk = k;
        }
    }

    e_entry = elf_ex->e_entry + load_bias;
    elf_bss += load_bias;
    elf_brk += load_bias;
    start_code += load_bias;
    end_code += load_bias;
    start_data += load_bias;
    end_data += load_bias;

    /* Calling set_brk effectively mmaps the pages that we need
     * for the bss and break sections.  We must do this before
     * mapping in the interpreter, to make sure it doesn't wind
     * up getting placed where the bss needs to go.
     */
    retval = set_brk(elf_bss, elf_brk, bss_prot);
    if (retval)
        goto out_free_dentry;
    if (likely(elf_bss != elf_brk) && unlikely(padzero(elf_bss))) {
        retval = -EFAULT; /* Nobody gets to see this, but.. */
        goto out_free_dentry;
    }

    // 填写程序入口的地址
    // 如果有 interpreter，设置为 interpreter 的入口地址，否则就是正常的入口

    if (interpreter) {
        elf_entry = load_elf_interp(interp_elf_ex,
                        interpreter,
                        load_bias, interp_elf_phdata,
                        &arch_state);
        if (!IS_ERR((void *)elf_entry)) {
            /*
             * load_elf_interp() returns relocation
             * adjustment
             */
            interp_load_addr = elf_entry;
            elf_entry += interp_elf_ex->e_entry;
        }
        if (BAD_ADDR(elf_entry)) {
            retval = IS_ERR((void *)elf_entry) ?
                    (int)elf_entry : -EINVAL;
            goto out_free_dentry;
        }
        reloc_func_desc = interp_load_addr;

        allow_write_access(interpreter);
        fput(interpreter);

        kfree(interp_elf_ex);
        kfree(interp_elf_phdata);
    } else {
        elf_entry = e_entry;
        if (BAD_ADDR(elf_entry)) {
            retval = -EINVAL;
            goto out_free_dentry;
        }
    }

    kfree(elf_phdata);

    set_binfmt(&elf_format);

    // 填写目标文件的参数，环境变量等必要信息

    retval = create_elf_tables(bprm, elf_ex,
              load_addr, interp_load_addr, e_entry);
    if (retval < 0)
        goto out;

    // 调整内存映射内容
    mm = current->mm;
    mm->end_code = end_code;
    mm->start_code = start_code;
    mm->start_data = start_data;
    mm->end_data = end_data;
    mm->start_stack = bprm->p;

    if ((current->flags & PF_RANDOMIZE) && (randomize_va_space > 1)) {
        /*
         * For architectures with ELF randomization, when executing
         * a loader directly (i.e. no interpreter listed in ELF
         * headers), move the brk area out of the mmap region
         * (since it grows up, and may collide early with the stack
         * growing down), and into the unused ELF_ET_DYN_BASE region.
         */
        if (IS_ENABLED(CONFIG_ARCH_HAS_ELF_RANDOMIZE) &&
            elf_ex->e_type == ET_DYN && !interpreter) {
            mm->brk = mm->start_brk = ELF_ET_DYN_BASE;
        }

        mm->brk = mm->start_brk = arch_randomize_brk(mm);
#ifdef compat_brk_randomized
        current->brk_randomized = 1;
#endif
    }

    if (current->personality & MMAP_PAGE_ZERO) {
        /* Why this, you ask???  Well SVr4 maps page 0 as read-only,
           and some applications "depend" upon this behavior.
           Since we do not have the power to recompile these, we
           emulate the SVr4 behavior. Sigh. */
        error = vm_mmap(NULL, 0, PAGE_SIZE, PROT_READ | PROT_EXEC,
                MAP_FIXED | MAP_PRIVATE, 0);
    }

    regs = current_pt_regs();

    finalize_exec(bprm);
    
    // 将用户进程的 pt_regs 中的 sp pc 分别设置好，这样，从系统调用返回的时候，就自动跳转到指定的位置执行了
    start_thread(regs, elf_entry, bprm->p);
    retval = 0;
out:
    return retval;

    // 发生错误，清理分配的结构体
out_free_dentry:
    kfree(interp_elf_ex);
    kfree(interp_elf_phdata);
    allow_write_access(interpreter);
    if (interpreter)
        fput(interpreter);
out_free_ph:
    kfree(elf_phdata);
    goto out;
}
```

### 2.3 register_binfmt && unregister_binfmt

在 `search_binary_handler` 中，遍历了 formats 这个链表并依次调用 `load_binary` 回调函数，也就是说，formats 保存的就是系统支持的所有可执行程序格式

前面说到，Linux 提供策略，具体机制由开发人员决定，自然，不同种类的格式需要注册到系统中才能生效，也就有了 register && unregister 两种操作

```c
static LIST_HEAD(formats);

static inline void register_binfmt(struct linux_binfmt *fmt)
{
    __register_binfmt(fmt, 0);
}

void __register_binfmt(struct linux_binfmt * fmt, int insert)
{
    BUG_ON(!fmt);
    if (WARN_ON(!fmt->load_binary))
        return;
    write_lock(&binfmt_lock);
    insert ? list_add(&fmt->lh, &formats) :
         list_add_tail(&fmt->lh, &formats);
    write_unlock(&binfmt_lock);
}

void unregister_binfmt(struct linux_binfmt * fmt)
{
    write_lock(&binfmt_lock);
    list_del(&fmt->lh);
    write_unlock(&binfmt_lock);
}
```
