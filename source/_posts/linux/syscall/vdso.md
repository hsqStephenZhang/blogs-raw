---
title: Linux vDSO 机制
categories: [linux, vsyscall, vDSO]
date: 2022-2-10 14:52:45
---

## 1. 什么是 vDSO

众所周知，操作系统为我们管理硬件资源，并以系统调用的方式对用户进程提供 API，但是 syscall 很慢，涉及陷入内核以及上下文切换。对于少量频繁调用的系统调用（比如获取当期系统时间）来说，是否可以某种安全的方式开放到用户空间，让用户直接访问而不需要经过 syscall 呢？

vDSO 就是用来解决这个问题的。

vDSO 全程为 `virtual dynamic shared object`，`dynamic shared object` 这个名词大家应该有所耳闻，就是 Linux 下的动态库的全称，而 **virtual** 表明，这个动态库是通过某种手段虚拟出来的，并不真正存在于 Linux 文件系统中。

要验证这点也很简单，只需要通过 ldd 命令，查看一些可执行文件所依赖的动态库即可，

```text
$ldd /bin/ls   
    linux-vdso.so.1 (0x00007ffe4e4ce000)
    libcap.so.2 => /usr/lib/libcap.so.2 (0x00007f7bf818e000)
    libc.so.6 => /usr/lib/libc.so.6 (0x00007f7bf7fc2000)
    /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f7bf81e8000)
```

可以明显看出，在ls 这个可执行文件依赖的动态库列表中，除了 `linux-vdso.so.1` 都有明确的路径

同时还可以通过 proc 文件系统中进程的内存映射（memory map）情况来映射这一点：

```text
$cat /proc/1/maps
....
7fd37e90f000-7fd37e911000 rw-p 0002f000 103:02 13244335                  /usr/lib/ld-2.33.so
7ffc2f7ce000-7ffc2f7ef000 rw-p 00000000 00:00 0                          [stack]
7ffc2f7f7000-7ffc2f7fb000 r--p 00000000 00:00 0                          [vvar]
7ffc2f7fb000-7ffc2f7fd000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

可以看出，vDSO 确实是以共享库的形式存在于每一个进程当中的

通过 vDSO，进程访问一些系统提供的 API，就可以直接在自己的地址空间访问，而不需要进行`用户-内核态`的状态切换了

## 2. vDSO 实现原理

`linux-vdso.so.1` 既然不是一个实实在在的文件，那其中的内容就应该直接保存在内存中，Linux 使用 vdso_image 来表示

### 2.1 vDSO image

在 `arch/x86/entyr/vdso/vdso-image-64.c` 文件中，定义了下面的 vdso_image

```c
static unsigned char raw_data[8192] __ro_after_init __aligned(PAGE_SIZE) = {
    0x7F, 0x45, 0x4C, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x03, 0x00, 0x3E, 0x00, 
    ...
};

const struct vdso_image vdso_image_64 = {
    .data = raw_data,
    .size = 8192,
    .alt = 3013,
    .alt_len = 91,
    .sym_vvar_start = -16384,
    .sym_vvar_page = -16384,
    .sym_pvclock_page = -12288,
    .sym_hvclock_page = -8192,
    .sym_timens_page = -4096,
};
```

`vdso_image.raw_data` 对应的就是 vDSO 提供的所有系统调用的二进制指令，一共有 8192 字节，相当于下面的结构: `static struct page *pages[2];`

`vdso_iamge_64` 自然需要保存到全局变量中才能发挥作用，这就涉及接下来要提到的 vDSO init

### 2.2 vDSO init

vDSO 通过 `init_vdso` 来初始化，通过条件编译对 32/64 bit 的 image 进行选择。同时也需要通过 `subsys_initcall(init_vdso);` 将 `init_vdso` 放到 initcall 列表中。

`init_vdso_image` 这里不过多介绍，主要是用来优化指令，毕竟 vdso_image 中提供的二进制指令是手动放在一个数组中的，还有相当大的优化空间

```c
static int __init init_vdso(void)
{
    BUILD_BUG_ON(VDSO_CLOCKMODE_MAX >= 32);

    init_vdso_image(&vdso_image_64);

#ifdef CONFIG_X86_X32_ABI
    init_vdso_image(&vdso_image_x32);
#endif

    return 0;
}
subsys_initcall(init_vdso);
```

### 2.3 vDSO 和 可执行程序

如果你对 Linux 可执行程序的 `加载-执行`机制有所研究，就知道对于 elf 格式的可执行程序而言，最终调用了 `load_elf_binary` 这个回调函数，在这个函数中，会根据 elf 文件头中的描述，设置好新进程的各个段，并将 elf 文件中的内容拷贝到相应位置。

为什么好端端的，要提到可执行程序加载呢？这是因为，在系统初始化完成之后，vdso_image 已经设置完毕，只需要在每次加载二进制可执行程序的时候，分配一块内存空间，将 vdso_image 加载到该位置即可

这就是 `arch_setup_additional_pages` 所要完成的任务了

```c
int arch_setup_additional_pages(struct linux_binprm *bprm, int uses_interp)
{
    if (!vdso64_enabled)
        return 0;

    return map_vdso_randomized(&vdso_image_64);
}
```

`map_vdso_randomized` 会通过 stack protect 机制，选择一个随机的加载地址，并调用 `map_vdso` 完成 mapping 工作，该函数内容较多，这里不赘述

最终，vDSO 会向用户提供四个 syscall

```text
__vdso_clock_gettime
__vdso_getcpu
__vdso_gettimeofday
__vdso_time
```

你还别不信，可以自行验证一下：

1. `cat /proc/1/maps` 找到 `[vdso]` 对应的内存位置
2. 通过 dd dump 内存，`dd if=/proc/1/mem of=/tmp/linux-vdso.so skip=140728627781632 ibs=1 count=4096`，其中 skip 的值为 vdso 的内存起始地址，count 为这块内存的大小
3. objdump 查看 `linux-vdso.so` 中所有符号 `objdump -T /tmp/linux-vdso.so`，最终结果如下

    ```text
    linux-vdso.so:     file format elf64-x86-64

    DYNAMIC SYMBOL TABLE:
    0000000000000740  w   DF .text  000000000000015d  LINUX_2.6   clock_gettime
    0000000000000600 g    DF .text  0000000000000127  LINUX_2.6   __vdso_gettimeofday
    00000000000008a0  w   DF .text  0000000000000044  LINUX_2.6   clock_getres
    00000000000008a0 g    DF .text  0000000000000044  LINUX_2.6   __vdso_clock_getres
    0000000000000600  w   DF .text  0000000000000127  LINUX_2.6   gettimeofday
    0000000000000730 g    DF .text  0000000000000010  LINUX_2.6   __vdso_time
    0000000000000730  w   DF .text  0000000000000010  LINUX_2.6   time
    0000000000000740 g    DF .text  000000000000015d  LINUX_2.6   __vdso_clock_gettime
    0000000000000000 g    DO *ABS*  0000000000000000  LINUX_2.6   LINUX_2.6
    00000000000008f0 g    DF .text  0000000000000025  LINUX_2.6   __vdso_getcpu
    00000000000008f0  w   DF .text  0000000000000025  LINUX_2.6   getcpu
    ```
