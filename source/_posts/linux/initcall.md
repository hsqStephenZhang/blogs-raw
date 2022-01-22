---
title: Linux initcall 机制
categories: [linux, basic]
date: 2022-1-18 13:47:40
---

## 1. initcall 设计思想

linux 对驱动程序提供静态编译进内核和动态加载两种方式，当我们试图将一个驱动程序编译进内核时，开发者通常提供一个xxx_init() 函数接口，以启动这个驱动程序同时提供某些服务。

那么，根据常识来说，这个 xxx_init() 函数肯定是要在系统启动的某个时候被调用，才能启动这个驱动程序。

最简单直观地做法就是：开发者试图添加一个驱动程序时，在内核启动 init 程序的某个地方直接添加调用自己驱动程序的 xxx_init()函数，在内核启动时自然会调用到这个程序。

但是在 linux 这么庞大的系统中，如果驱动程序是这么个玩法，那简直是一场灾难。

不难想到另一种方式，就是集中提供一个地方，如果你要添加你的驱动程序，你就将你的初始化函数在这个地方进行添加，在内核启动的时候统一扫描这个地方，再执行这一部分的所有被添加的驱动程序。

那到底怎么添加呢？直接在C文件中作一个列表，在里面添加初始化函数？我想随着驱动程序数量的增加，这个列表会让人头昏眼花。

当然，对于 linux 社区的大神来说，这些都不是事，他们选择的做法是：

底层实现上，在内核编译得到的 elf 文件中，自定义一个段，这个段里面专门用来存放这些初始化函数的地址（也会被拷贝到镜像中），内核启动时，只需要在这个段地址处取出函数指针，一个个执行即可。

对上层而言，linux内核提供 xxx_init(init_func)宏定义接口,驱动开发者只需要将驱动程序的 init_func 使用 `__init` 来修饰，这个函数就被自动添加到了上述的段中，开发者完全不需要关心实现细节。

同时，如果是运行过程当中添加的模块，也需要执行 init 函数 和 exit 函数，其原理也和 initcall 有关

对于各种各样的驱动而言，可能存在一定的依赖关系，需要遵循先后顺序来进行初始化，考虑到这个，linux也对这一部分做了分级处理。

## 2. 深入 initcall

下面先举一个例子，然后一步步探究 Linux 中对于 initcall 的处理方式

```c
// fs/io_uring.c
static int __init io_uring_init(void){
    ...
};
__initcall(io_uring_init);
```

io_uring 是一个比较新的异步 IO 机制，`fs/io_uring.c` 中定义了 io_uring_init 这个函数，字如其名，一看就是用作初始化的，该函数头上也加上了 `__init` 这个宏，并且，使用 `__initcall`，虽然我们还不知道具体的作用，但再次印证了 “初始化函数” 这个方向没有问题

或者如果编写一个内核模块，也需要通过 `module_init` 和 `module_exit`，如果缺少了这两个宏，内核模块的编译就无法通过

```c
static int __init hello_init(void)
{
    printk(KERN_INFO "Hello world!\n");
    return 0;
}

static void __exit hello_cleanup(void)
{
    printk(KERN_INFO "Cleaning up module.\n");
}

module_init(hello_init);
module_exit(hello_cleanup);
```

那么问题来了，`__init` `__initcall` `module_init` 是些什么鬼，背后有什么不为人知的魔法吗？

### 2.1 __init 是什么鬼

其实，`__init` 的作用非常简单，主要就是将符号定义在 `.init.text` 这个 section 当中，而不是放在函数默认的 '.text' section 中。

```c
// include/linux/init.h
#define __init  __section(".init.text") __cold  __latent_entropy __noinitretpoline
```

`__initcall` 定义在 `include/linux/init.h` 文件中，从下往上查找，最终发现，也是定义了符号所处的 section

```c
// include/linux/init.h

#define ___define_initcall(fn, id, __sec) \
 static initcall_t __initcall_##fn##id __used \
  __attribute__((__section__(#__sec ".init"))) = fn;
#endif

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

#define device_initcall(fn)  __define_initcall(fn, 6)

#define __initcall(fn) device_initcall(fn)
```

如果将上面的宏代入 `__initcall(io_uring_init)` 并展开之后，其实相当于
`static initcall_t __initcall_io_uring_init6 __attribute__((__used__)) __attribute__((__section__(".initcall6.init"))) = io_uring_init;`

这行代码的意思是：声明一个类型为`initcall_t` 的函数指针 `__initcall_io_uring_init6`，并且将其放在 '.initcall6.init' 段中，并且函数指针指向 io_uring_init 这个初始化函数。同时，`__used__` 这个声明，会告诉编译器：保留这个符号，不要因为当前没有使用这个符号，而直接优化

其实，`init.h` 文件中还定义了很多别的类似的宏，比如：

```c
#define pure_initcall(fn)  __define_initcall(fn, 0)

#define core_initcall(fn)  __define_initcall(fn, 1)
#define core_initcall_sync(fn)  __define_initcall(fn, 1s)
#define postcore_initcall(fn)  __define_initcall(fn, 2)
#define postcore_initcall_sync(fn) __define_initcall(fn, 2s)
#define arch_initcall(fn)  __define_initcall(fn, 3)
#define arch_initcall_sync(fn)  __define_initcall(fn, 3s)
#define subsys_initcall(fn)  __define_initcall(fn, 4)
#define subsys_initcall_sync(fn) __define_initcall(fn, 4s)
#define fs_initcall(fn)   __define_initcall(fn, 5)
#define fs_initcall_sync(fn)  __define_initcall(fn, 5s)
#define rootfs_initcall(fn)  __define_initcall(fn, rootfs)
#define device_initcall(fn)  __define_initcall(fn, 6)
#define device_initcall_sync(fn) __define_initcall(fn, 6s)
#define late_initcall(fn)  __define_initcall(fn, 7)
#define late_initcall_sync(fn)  __define_initcall(fn, 7s)
```

这里 `__define_initcall(fn, 0)` 中的数字作用，先按下不表，后续会揭露

而内核模块必不可少的 `module_init` 就相当于 `__initcall`，同样是定义了一个位于 '.initcall6.init' section 的函数指针

```c
#define module_init(x) __initcall(x);
```

Soga，原来所谓的 `__initcall` `module_init`，只不过是将创建一个位于 '.initcallx.init' section 的函数指针，指向特定的 init 函数。

那么问题来了，这些函数指针是如何起作用的呢？init 函数需要我们手动调用吗？还是有更加神奇的魔法，可以自动完成一系列 init 函数的调用呢？

### 2.3 initcall 调用时机

从内核 C 函数起始部分，也就是 `start_kernel` 开始往下挖，在 `do_initcalls` 可以找到对于散布在各个模块中的 initcall 的调用

```text
start_kernel  
 -> rest_init();
  -> kernel_thread(kernel_init, NULL, CLONE_FS);
   -> kernel_init()
    -> kernel_init_freeable();
     -> do_basic_setup();
      -> do_initcalls();  
```

`do_initcalls` 大致的作用是，遍历 0-initcall_levels，并且依次调用对应 level 的 do_initcall_level 函数

```c
static initcall_t *initcall_levels[] __initdata = {
 __initcall0_start,
 __initcall1_start,
 __initcall2_start,
 __initcall3_start,
 __initcall4_start,
 __initcall5_start,
 __initcall6_start,
 __initcall7_start,
 __initcall_end,
};

static void __init do_initcalls(void)
{
 // ...
 for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
  strcpy(command_line, saved_command_line);
  do_initcall_level(level, command_line);
 }
    // ...
}
```

这里就可以揭秘之前卖下的一个关子了，`__define_initcall(fn, 6)` 中的数字，就对应了 do_initcall_level 中的 level，level 越小，优先级越高，越早调用模块中的 initcall 函数。

这一点也可以从 `pure_initcall` `core_initcall` 这些宏定义的名字看出，core 也就是**核心**的意思，自然优先级很高才对，印证了 `__define_initcall` 中 level 的含义。

接下来的 `do_initcall_level` 所做的事情是：依次调用 `[initcall_levels[level],initcall_levels[level+1])` 范围内的每一个函数指针，按照我们的理解，自然是应该调用我们之前通过 `__initcall` `module_init` 定义的函数指针才对。

```c
static void __init do_initcall_level(int level, char *command_line)
{
    // ...
 for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
  do_one_initcall(initcall_from_entry(fn));
}
```

但是如果你搜索一下 Linux 源码，会发现根本没有 `__initcall0_start` `__initcall1_start` ... 这些符号，难道是凭空冒出来的？不可能！既然不是直接定义的符号，那就是通过宏间接拼接出来的符号名！果然，通过 `__initcall + level + _start` 这个规律，搜索到了 `INIT_CALLS_LEVEL` 这个宏，正是用来声明 `initcall_levels` 中的一系列符号的

```c
// include/asm-generic/vmlinux.lds.h
#define INIT_CALLS_LEVEL(level)      \
  __initcall##level##_start = .;    \
  KEEP(*(.initcall##level##.init))   \
  KEEP(*(.initcall##level##s.init))   \
```

不过需要注意的文件名，`vmlinux.lds.h`，发现只有在 `vmlinux.lds.S` 中才被 include 进来，这是因为，`INIT_CALLS_LEVEL`，并不是作用于 C语言编译过程中，而是在 linker 链接的过程中发挥其**魔法**。

比如对于 `.initcall0.init`，有很多同样位于这个 section 的函数指针，散步在各个object 文件中，只需要在链接的时候，通过 linker 脚本的这个语法，将其聚合在一起，也就是顺序存放在 `__initcall0_start` 开始的地方即可

针对 INIT_CALLS_LEVEL，Linux 中还做了别的封装，比如 `INIT_CALLS` `INIT_DATA_SECTION`，最重要的一步是则是在 `vmlinux.lds.S` 中调用这个宏，将该指针数组链接起来，并且提供`__initcall1_start` 等一系列符号，给 `do_initcalls` 使用

```c
// include/asm-generic/vmlinux.lds.h
#define INIT_CALLS       \
  __initcall_start = .;     \
  KEEP(*(.initcallearly.init))    \
  INIT_CALLS_LEVEL(0)     \
  INIT_CALLS_LEVEL(1)     \
  INIT_CALLS_LEVEL(2)     \
  INIT_CALLS_LEVEL(3)     \
  INIT_CALLS_LEVEL(4)     \
  INIT_CALLS_LEVEL(5)     \
  INIT_CALLS_LEVEL(rootfs)    \
  INIT_CALLS_LEVEL(6)     \
  INIT_CALLS_LEVEL(7)     \
  __initcall_end = .;

#define INIT_DATA_SECTION(initsetup_align)    \
 .init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {  \
  INIT_DATA      \
  INIT_SETUP(initsetup_align)    \
  INIT_CALLS      \
  CON_INITCALL      \
  INIT_RAM_FS      \
 }
```

```asm
// ...
INIT_DATA_SECTION(16)
// ...
```

由于 `INIT_CALLS_LEVEL(x)` 和 `INIT_CALLS_LEVEL(x+1)` 是按顺序调用的，所以，在 `__initcall0_start` `__initcall1_start` 之间的所有函数指针，就都是 level=0 的init 函数了，再回头看 `for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)`，是不是就清楚了很多呢？

## 3. 内核模块

前面说到，内核模块的源码中，也必须要通过 `module_init(__initcall)` 这样的方式来定义初始化函数指针，那我们就来看看，该函数指针是何时被调用的

```bash
sys_init_module
 -> load_module
  -> ...
  -> do_init_module
   -> do_mod_ctors
   -> do_one_initcall(mod->init)
```

加载一个模块的时候，需要调用 init_module 这个系统调用，提供模块的信息(umod)，以及一些参数(uargs)，在完成了一系列检查，重定位等操作之后，直接调用 `mod->init` 这个初始化函数

这里并不和我们想象中一样，也去获取 `__initcall0_start` `__initcall1_start` 之间的所有初始化函数，而是用户在调用 `init_module` 这个 syscall 之前，就需要将 init 函数从指定的 `.initcall6.init` 中取出来，放在 umod->init 中。这一切都是由 modprobe 或者 insmod 帮我们做的，如果通过 `readelf -a xxx.o` 会发现，并不存在 `.initcall6.init'` 这样的 section，而是 `.init.text`，我猜测，这是以为编译内核模块的时候，已经针对 `.initcall6.init` 进行了额外处理，转换成了 `.init.text`，毕竟单独加载一个内核模块，也不需要这个信息了，如果哪位yy对这里熟悉，欢迎交流讨论。

## 4. 附录 vmlinux initcall 有关符号表

编译得到 vmlinux 之后，通过 `readelf -a ./vmlinux | rg initcall`，查看有关 initcall 的符号

```bash
$ readelf -a ./vmlinux | rg initcall
ffffffff811007d2  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff811007f1  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff8110081d  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81100828  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81100a89  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81100a98  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81100adb  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81100ae6  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff811b2806  1d7ab00000004 R_X86_64_PLT32    ffffffff81003be0 do_one_initcall - 4
ffffffff81826af2  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81826b05  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81826b2c  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81826b41  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81a851e7  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81a851f2  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81a8522f  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81a8523a  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81a8529e  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81a852a9  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81a89b5d  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81a89b68  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff81a8abc9  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff81a8abd4  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff82adf860  1b13600000001 R_X86_64_64       ffffffff8397f000 initcall_debug + 0
ffffffff837be7fe  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff837be809  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 5
ffffffff837becdb  1c0d20000000b R_X86_64_32S      ffffffff83944090 __initcall_start + 0
ffffffff837bed1a  19bb40000000b R_X86_64_32S      ffffffff839440f8 __initcall0_start + 0
ffffffff837bed33  1d7ab00000004 R_X86_64_PLT32    ffffffff81003be0 do_one_initcall - 4
ffffffff837bee66  1d7ab00000004 R_X86_64_PLT32    ffffffff81003be0 do_one_initcall - 4
ffffffff837f374e  1b1360000000b R_X86_64_32S      ffffffff8397f000 initcall_debug + 0
ffffffff837f3759  1b13600000002 R_X86_64_PC32     ffffffff8397f000 initcall_debug - 4
ffffffff837f4988  1da510000000b R_X86_64_32S      ffffffff83944bc8 __con_initcall_start + 0
ffffffff837f49a7  1a6d80000000b R_X86_64_32S      ffffffff83944bdc __con_initcall_end + 0
ffffffff8389a060  19bb400000001 R_X86_64_64       ffffffff839440f8 __initcall0_start + 0
ffffffff8389a068  1d31400000001 R_X86_64_64       ffffffff83944110 __initcall1_start + 0
ffffffff8389a070  1db8100000001 R_X86_64_64       ffffffff83944228 __initcall2_start + 0
ffffffff8389a078  1a43600000001 R_X86_64_64       ffffffff839442bc __initcall3_start + 0
ffffffff8389a080  1a53b00000001 R_X86_64_64       ffffffff83944318 __initcall4_start + 0
ffffffff8389a088  1b67a00000001 R_X86_64_64       ffffffff839445b0 __initcall5_start + 0
ffffffff8389a090  1b27d00000001 R_X86_64_64       ffffffff839446c8 __initcall6_start + 0
ffffffff8389a098  1c6b700000001 R_X86_64_64       ffffffff83944a90 __initcall7_start + 0
ffffffff8389a0a0  19cac00000001 R_X86_64_64       ffffffff83944bc8 __initcall_end + 0
0000000256fc  1b13600000001 R_X86_64_64       ffffffff8397f000 initcall_debug + 0
00000002df77  1b13600000001 R_X86_64_64       ffffffff8397f000 initcall_debug + 0
   183: ffffffff810031e0    72 FUNC    LOCAL  DEFAULT    1 trace_initcall_f[...]
   184: ffffffff8201d8e3    84 FUNC    LOCAL  DEFAULT    1 trace_initcall_s[...]
   201: ffffffff810037a0   253 FUNC    LOCAL  DEFAULT    1 initcall_blacklisted
   202: ffffffff8301a0e0    16 OBJECT  LOCAL  DEFAULT   26 blacklisted_initcalls
   205: ffffffff8201da4a   108 FUNC    LOCAL  DEFAULT    1 trace_initcall_level
   208: ffffffff837bde65   308 FUNC    LOCAL  DEFAULT   37 initcall_blacklist
   226: ffffffff8397f010     8 OBJECT  LOCAL  DEFAULT   60 initcall_calltime
   229: ffffffff8389a020    64 OBJECT  LOCAL  DEFAULT   41 initcall_level_names
   230: ffffffff8389a060    72 OBJECT  LOCAL  DEFAULT   41 initcall_levels
   235: ffffffff839420c8    24 OBJECT  LOCAL  DEFAULT   41 __setup_initcall[...]
   237: ffffffff82adf840    40 OBJECT  LOCAL  DEFAULT   16 __param_initcall[...]
   254: ffffffff839397a0     8 OBJECT  LOCAL  DEFAULT   41 __event_initcall[...]
   255: ffffffff8301a500   144 OBJECT  LOCAL  DEFAULT   26 event_initcall_finish
   257: ffffffff839397a8     8 OBJECT  LOCAL  DEFAULT   41 __event_initcall[...]
   258: ffffffff8301a5a0   144 OBJECT  LOCAL  DEFAULT   26 event_initcall_start
   260: ffffffff839397b0     8 OBJECT  LOCAL  DEFAULT   41 __event_initcall[...]
   261: ffffffff8301a640   144 OBJECT  LOCAL  DEFAULT   26 event_initcall_level
   270: ffffffff82600148     9 OBJECT  LOCAL  DEFAULT    3 str__initcall__t[...]
   339: ffffffff839446bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_popul[...]
   463: ffffffff83944318     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_vdso4
   479: ffffffff8394431c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sysen[...]
   480: ffffffff839446c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ia32_[...]
   538: ffffffff83944090     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
   674: ffffffff839446cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_amd_u[...]
   740: ffffffff839446d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_amd_i[...]
   783: ffffffff839446d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_amd_i[...]
   820: ffffffff839446d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_msr_init6
   873: ffffffff83944320     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fixup[...]
  1139: ffffffff839442bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bts_init3
  1290: ffffffff839442c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pt_init3
  1485: ffffffff83944110     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_p[...]
  2390: ffffffff83944094     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
  2461: ffffffff83944098     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
  2750: ffffffff839445b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_nmi_w[...]
  2840: ffffffff839446dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
  2874: ffffffff839446e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_i8259[...]
  3056: ffffffff839442c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_boot_[...]
  3073: ffffffff839442c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sbf_init3
  3085: ffffffff83944114     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_e820_[...]
  3116: ffffffff839446c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_i[...]
  3161: ffffffff83944324     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_topol[...]
  3172: ffffffff839442cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_arch_[...]
  3245: ffffffff83944118     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpufr[...]
  3256: ffffffff839446e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
  3334: ffffffff839446e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_add_r[...]
  3580: ffffffff839446ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_i8237[...]
  3809: ffffffff839446f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_umwai[...]
  3884: ffffffff839442d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
  3894: ffffffff83944328     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
  4008: ffffffff83944a8c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mchec[...]
  4010: ffffffff83944a90     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mchec[...]
  4148: ffffffff83944a94     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sever[...]
  4261: ffffffff839446f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_therm[...]
  4320: ffffffff8394432c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mtrr_[...]
  4345: ffffffff839442d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mtrr_[...]
  4468: ffffffff839445b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_save_[...]
  4470: ffffffff83944a98     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_micro[...]
  4610: ffffffff83944a9c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_resct[...]
  4806: ffffffff839442d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_activ[...]
  4935: ffffffff83944aa0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hpet_[...]
  5027: ffffffff839442dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ffh_c[...]
  5040: ffffffff8394411c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_reboo[...]
  5063: ffffffff839446f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_msr_init6
  5087: ffffffff839446fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpuid[...]
  5302: ffffffff83944aa4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_updat[...]
  5348: ffffffff83944120     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
  5350: ffffffff83944aa8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lapic[...]
  5487: ffffffff83944aac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_print[...]
  5502: ffffffff83944ab0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_print_ICs7
  5571: ffffffff8394409c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
  5582: ffffffff83944700     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ioapi[...]
  5913: ffffffff839445b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hpet_[...]
  6019: ffffffff839445bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
  6056: ffffffff839442e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_activ[...]
  6058: ffffffff839442e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kvm_a[...]
  6124: ffffffff839440a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kvm_s[...]
  6227: ffffffff83944704     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
  6250: ffffffff83944708     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_add_p[...]
  6253: ffffffff8394470c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_start[...]
  6297: ffffffff83944710     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sysfb[...]
  6392: ffffffff83944714     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_audit[...]
  6709: ffffffff83944ab4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_creat[...]
  6737: ffffffff83944ab8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpa_s[...]
  6846: ffffffff83944abc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pat_m[...]
  6900: ffffffff839442e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_gigan[...]
  6907: ffffffff83944718     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pt_du[...]
  7039: ffffffff83944ac0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_creat[...]
  7206: ffffffff8394471c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iosf_[...]
  7416: ffffffff83944720     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
  7452: ffffffff83944ac4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
  7457: ffffffff83944724     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
  7525: ffffffff83944124     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_alloc[...]
  7527: ffffffff83944128     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpu_h[...]
  7547: ffffffff83944728     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpuhp[...]
  7794: ffffffff839440a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_spawn[...]
  7881: ffffffff8394472c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iores[...]
  8183: ffffffff83944330     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_uid_c[...]
  9086: ffffffff8394412c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wq_sy[...]
  9470: ffffffff83944334     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_param[...]
  9695: ffffffff83944130     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ksysf[...]
  9898: ffffffff83944338     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_user_[...]
 10184: ffffffff839440a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_migra[...]
 10663: ffffffff83944ac8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sched[...]
 11222: ffffffff8394433c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 11230: ffffffff83944acc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sched[...]
 11232: ffffffff83944730     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 11295: ffffffff83944134     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sched[...]
 11377: ffffffff83944734     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_psi_p[...]
 11695: ffffffff839445c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 11715: ffffffff83944ad0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpu_l[...]
 11769: ffffffff83944ad4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pm_de[...]
 11774: ffffffff83944138     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pm_init1
 11936: ffffffff83944bac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_softw[...]
 11938: ffffffff8394413c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pm_di[...]
 12049: ffffffff83944140     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_swsus[...]
 12099: ffffffff83944738     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_snaps[...]
 12112: ffffffff83944340     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pm_sy[...]
 12120: ffffffff83944144     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_em_de[...]
 12201: ffffffff83944ad8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_print[...]
 12412: ffffffff83944228     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_irq_s[...]
 12784: ffffffff8394473c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_irq_g[...]
 13019: ffffffff83944740     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_irq_p[...]
 13151: ffffffff83944148     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rcu_s[...]
 13177: ffffffff8394414c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rcu_s[...]
 13191: ffffffff83944150     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rcu_s[...]
 13305: ffffffff839440ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_srcu_[...]
 13307: ffffffff83944adc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 13375: ffffffff839440b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rcu_s[...]
 13395: ffffffff839440b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rcu_s[...]
 13400: ffffffff839440b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_check[...]
 13408: ffffffff839440bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rcu_s[...]
 13801: ffffffff83944ae0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_swiot[...]
 13843: ffffffff8394422c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dma_a[...]
 13862: ffffffff839440c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 13864: ffffffff839440c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 13907: ffffffff839442ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kcmp_[...]
 13960: ffffffff83944344     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_creat[...]
 14578: ffffffff83944744     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_timek[...]
 14647: ffffffff839445c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_clock[...]
 14661: ffffffff83944748     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 14719: ffffffff83944154     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 14726: ffffffff8394474c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 14787: ffffffff83944750     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_alarm[...]
 14870: ffffffff83944754     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 15201: ffffffff83944758     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_clock[...]
 15305: ffffffff83944ae4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tk_de[...]
 15313: ffffffff83944348     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_time_[...]
 15323: ffffffff83944158     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_futex[...]
 15405: ffffffff8394475c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 15727: ffffffff83944760     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 15949: ffffffff83944764     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kalls[...]
 15991: ffffffff8394434c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crash[...]
 16005: ffffffff83944350     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crash[...]
 16161: ffffffff8394415c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cgrou[...]
 16169: ffffffff83944354     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cgrou[...]
 16418: ffffffff83944358     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cgrou[...]
 16430: ffffffff83944160     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cgrou[...]
 16646: ffffffff8394435c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_user_[...]
 16677: ffffffff83944768     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pid_n[...]
 16694: ffffffff8394476c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ikcon[...]
 16701: ffffffff839440c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpu_s[...]
 16727: ffffffff83944230     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_audit[...]
 16862: ffffffff83944770     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_audit[...]
 16873: ffffffff83944774     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_audit[...]
 16882: ffffffff83944778     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_audit[...]
 16940: ffffffff839440cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 16942: ffffffff83944ae8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_debug[...]
 17151: ffffffff83944360     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hung_[...]
 17232: ffffffff8394477c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_secco[...]
 17340: ffffffff83944780     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_utsna[...]
 17357: ffffffff83944aec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tasks[...]
 17382: ffffffff83944234     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_relea[...]
 17399: ffffffff83944784     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 17420: ffffffff83944788     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 17445: ffffffff83944164     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ftrac[...]
 17866: ffffffff83944bb0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_laten[...]
 17907: ffffffff839445c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 17909: ffffffff83944bb4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_clear[...]
 17911: ffffffff83944bb8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_traci[...]
 18219: ffffffff839440d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18340: ffffffff839445cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18342: ffffffff839440d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18429: ffffffff83944168     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18472: ffffffff83944af0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18508: ffffffff8394478c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_stack[...]
 18538: ffffffff83944790     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18561: ffffffff839445d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18563: ffffffff8394416c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18616: ffffffff83944794     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 18748: ffffffff839440d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_event[...]
 19145: ffffffff83944170     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 19147: ffffffff839445d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 19342: ffffffff83944364     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_send_[...]
 19344: ffffffff839445d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_e[...]
 19434: ffffffff83944174     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 19436: ffffffff839445dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 19839: ffffffff839445e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 19866: ffffffff839445e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 19913: ffffffff83944220     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 19943: ffffffff839440f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_j[...]
 20392: ffffffff839445e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_init5
 20470: ffffffff83944af4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_m[...]
 20487: ffffffff83944af8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_task_[...]
 20511: ffffffff83944afc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_p[...]
 20741: ffffffff83944b00     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 20865: ffffffff83944368     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dev_m[...]
 20894: ffffffff8394436c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpu_m[...]
 20959: ffffffff83944370     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_netns[...]
 20976: ffffffff83944374     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_stack[...]
 21054: ffffffff83944378     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_btf_v[...]
 21718: ffffffff83944238     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kcsan[...]
 21726: ffffffff839440dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_stati[...]
 21796: ffffffff83944798     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_perf_[...]
 22291: ffffffff839440e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_jump_[...]
 22399: ffffffff8394479c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_syste[...]
 22401: ffffffff83944b04     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_load_[...]
 22420: ffffffff839447a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_black[...]
 22690: ffffffff8394437c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_oom_init4
 23153: ffffffff839447a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kswap[...]
 23668: ffffffff839447a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_extfr[...]
 23714: ffffffff8394423c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bdi_c[...]
 23716: ffffffff83944380     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_defau[...]
 23718: ffffffff83944384     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cgwb_init4
 23792: ffffffff839447ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mm_co[...]
 23794: ffffffff83944240     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mm_sy[...]
 23815: ffffffff83944388     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_percp[...]
 23960: ffffffff839447b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_slab_[...]
 24254: ffffffff8394438c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kcomp[...]
 24506: ffffffff839447b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_worki[...]
 24613: ffffffff83944178     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 24660: ffffffff83944b08     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fault[...]
 24836: ffffffff83944390     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 24838: ffffffff83944394     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 24840: ffffffff83944398     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 25129: ffffffff839447b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 25292: ffffffff83944244     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 25526: ffffffff8394439c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_swap_[...]
 25550: ffffffff839447bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_procs[...]
 25552: ffffffff83944b0c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_max_s[...]
 25560: ffffffff839443a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_swapf[...]
 25686: ffffffff839447c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 25699: ffffffff83944b10     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 25831: ffffffff839443a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_huget[...]
 26145: ffffffff839443a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ksm_init4
 26291: ffffffff839447c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_slab_[...]
 26625: ffffffff839443ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hugep[...]
 26639: ffffffff83944b14     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_split[...]
 26879: ffffffff839440fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_memor[...]
 26890: ffffffff839443b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mem_c[...]
 26892: ffffffff8394417c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mem_c[...]
 27118: ffffffff83944180     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_memor[...]
 27202: ffffffff839447c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 27255: ffffffff839447cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_zbud6
 27301: ffffffff839447d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_zs_init6
 27346: ffffffff839447d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 27383: ffffffff83944b18     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_check[...]
 27398: ffffffff83944184     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cma_i[...]
 27465: ffffffff83944b1c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cma_d[...]
 27495: ffffffff839443b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_page_[...]
 27534: ffffffff83944b20     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_set_h[...]
 28652: ffffffff839445ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 29017: ffffffff839447d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fcntl[...]
 29700: ffffffff839447dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 30460: ffffffff839445f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cgrou[...]
 30465: ffffffff839447e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_start[...]
 31646: ffffffff839447e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blkde[...]
 31766: ffffffff839447e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dio_init6
 31827: ffffffff83944188     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fsnot[...]
 31899: ffffffff839447ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dnoti[...]
 31925: ffffffff839445f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_inoti[...]
 32023: ffffffff839447f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fanot[...]
 32086: ffffffff839445f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_event[...]
 32185: ffffffff839445fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_anon_[...]
 32343: ffffffff839447f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_userf[...]
 32378: ffffffff839447f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_aio_setup6
 32546: ffffffff839447fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_io_ur[...]
 32897: ffffffff839443b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_io_wq[...]
 32920: ffffffff83944600     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 33097: ffffffff83944b24     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fscry[...]
 33356: ffffffff83944b28     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fsver[...]
 33501: ffffffff83944604     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 33503: ffffffff8394418c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_filel[...]
 33664: ffffffff83944190     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 33715: ffffffff83944194     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 33722: ffffffff83944198     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 33739: ffffffff8394419c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 34019: ffffffff83944608     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iomap[...]
 34244: ffffffff8394460c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dquot[...]
 34342: ffffffff83944610     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_quota[...]
 34719: ffffffff83944614     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34724: ffffffff83944618     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34734: ffffffff8394461c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34740: ffffffff83944620     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34749: ffffffff83944624     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34757: ffffffff83944628     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34762: ffffffff8394462c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34767: ffffffff83944630     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34777: ffffffff83944634     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34784: ffffffff83944638     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34789: ffffffff8394463c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34918: ffffffff83944640     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34952: ffffffff83944644     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_vmcor[...]
 34987: ffffffff83944648     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 34996: ffffffff8394464c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 35006: ffffffff83944650     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 35380: ffffffff839441a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_confi[...]
 35430: ffffffff83944800     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 35484: ffffffff83944654     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 35506: ffffffff83944658     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 35619: ffffffff83944804     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 35638: ffffffff83944808     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 35727: ffffffff8394480c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 35897: ffffffff839441a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_debug[...]
 36119: ffffffff839441a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_trace[...]
 36191: ffffffff83944b2c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pstor[...]
 36242: ffffffff83944810     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efiva[...]
 36262: ffffffff83944814     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ipc_init6
 36430: ffffffff83944100     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ipc_n[...]
 36514: ffffffff83944818     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ipc_s[...]
 36530: ffffffff8394481c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 36842: ffffffff83944b30     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 36918: ffffffff83944820     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_key_p[...]
 36960: ffffffff83944104     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 37229: ffffffff839441ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_secur[...]
 37290: ffffffff83944824     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_selin[...]
 37579: ffffffff83944828     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 37677: ffffffff8394482c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_selnl[...]
 37690: ffffffff83944830     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sel_n[...]
 37702: ffffffff83944834     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sel_n[...]
 37712: ffffffff83944838     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sel_n[...]
 37839: ffffffff8394483c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_aurul[...]
 37920: ffffffff839443bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sel_i[...]
 38125: ffffffff83944840     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 38264: ffffffff83944844     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_smack[...]
 38451: ffffffff8394465c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tomoy[...]
 38510: ffffffff83944660     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_aa_cr[...]
 38775: ffffffff83944848     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_appar[...]
 38991: ffffffff83944b34     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 39065: ffffffff83944664     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_safes[...]
 39082: ffffffff839441b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lockd[...]
 39371: ffffffff8394484c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crypt[...]
 39726: ffffffff839443c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dh_init4
 39756: ffffffff839443c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rsa_init4
 39865: ffffffff839442f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crypt[...]
 39887: ffffffff839443c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hmac_[...]
 39909: ffffffff839443cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crypt[...]
 39933: ffffffff839443d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_md5_m[...]
 39954: ffffffff839443d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sha1_[...]
 39974: ffffffff839443d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sha25[...]
 39995: ffffffff839443dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sha51[...]
 40006: ffffffff839443e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crypt[...]
 40029: ffffffff839443e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_aes_init4
 40045: ffffffff839443e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crct1[...]
 40056: ffffffff839443ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lz4_m[...]
 40109: ffffffff839443f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_drbg_init4
 40170: ffffffff83944850     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_jent_[...]
 40183: ffffffff839443f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_zstd_[...]
 40223: ffffffff83944854     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_asymm[...]
 40348: ffffffff83944858     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_x509_[...]
 40551: ffffffff839443f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_bio4
 41236: ffffffff839443fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blk_s[...]
 41252: ffffffff83944400     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blk_i[...]
 41324: ffffffff83944b38     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blk_t[...]
 41478: ffffffff83944404     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blk_m[...]
 41657: ffffffff83944408     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_genhd[...]
 41659: ffffffff8394485c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proc_[...]
 41991: ffffffff83944668     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blk_s[...]
 42009: ffffffff83944860     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bsg_init6
 42119: ffffffff8394440c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_blkcg[...]
 42173: ffffffff83944864     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_throt[...]
 42226: ffffffff83944868     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iolat[...]
 42255: ffffffff8394486c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ioc_init6
 42374: ffffffff83944870     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_deadl[...]
 42424: ffffffff83944874     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kyber[...]
 42524: ffffffff83944878     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bfq_init6
 43198: ffffffff83944410     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bio_c[...]
 43353: ffffffff839441b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_prand[...]
 43355: ffffffff83944b3c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_prand[...]
 44494: ffffffff8394487c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_btree[...]
 44598: ffffffff83944880     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_crc_t[...]
 45267: ffffffff83944884     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_percp[...]
 45276: ffffffff83944b40     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 45312: ffffffff839440e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dynam[...]
 45314: ffffffff8394466c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dynam[...]
 45680: ffffffff83944248     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mpi_init2
 45772: ffffffff83944888     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sg_po[...]
 45805: ffffffff83944414     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_irq_p[...]
 46477: ffffffff8394424c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kobje[...]
 46843: ffffffff839440e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_initi[...]
 47453: ffffffff8394488c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_phy_c[...]
 47594: ffffffff839441b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pinct[...]
 47731: ffffffff83944418     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sx150[...]
 47777: ffffffff8394441c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_byt_g[...]
 47910: ffffffff83944420     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_chv_p[...]
 48037: ffffffff83944424     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lp_gp[...]
 48160: ffffffff83944428     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bxt_p[...]
 48314: ffffffff83944890     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cnl_p[...]
 48386: ffffffff8394442c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cdf_p[...]
 48399: ffffffff83944430     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dnv_p[...]
 48424: ffffffff83944894     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ebg_p[...]
 48440: ffffffff83944434     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_glk_p[...]
 48514: ffffffff83944898     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_icl_p[...]
 48556: ffffffff8394489c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_jsl_p[...]
 48571: ffffffff839448a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lbg_p[...]
 48582: ffffffff83944438     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_spt_p[...]
 48648: ffffffff839448a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tgl_p[...]
 48911: ffffffff839441bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_gpiol[...]
 48913: ffffffff8394443c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_gpiol[...]
 49252: ffffffff83944bbc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 49254: ffffffff83944250     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 49302: ffffffff83944440     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_palma[...]
 49321: ffffffff83944444     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rc5t5[...]
 49333: ffffffff83944448     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tps65[...]
 49344: ffffffff8394444c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tps65[...]
 49412: ffffffff83944450     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pwm_d[...]
 49458: ffffffff83944454     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pwm_s[...]
 49501: ffffffff839448a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cryst[...]
 49663: ffffffff83944254     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pcibu[...]
 50103: ffffffff83944b44     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_r[...]
 50108: ffffffff83944108     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_r[...]
 50253: ffffffff83944258     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_d[...]
 50349: ffffffff83944b48     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_s[...]
 50638: ffffffff839448ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pcie_[...]
 50854: ffffffff839448b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_p[...]
 50888: ffffffff83944458     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_s[...]
 50913: ffffffff839442f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 50955: ffffffff839446b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_a[...]
 51229: ffffffff839448b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_h[...]
 51442: ffffffff839448b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_shpcd[...]
 51792: ffffffff839448bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_s[...]
 51952: ffffffff839448c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dw_pl[...]
 51961: ffffffff839448c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_meson[...]
 52210: ffffffff8394425c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_backl[...]
 52334: ffffffff8394445c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fbmem[...]
 52782: ffffffff839448c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_vesaf[...]
 52806: ffffffff839448cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efifb[...]
 52854: ffffffff839448d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 52949: ffffffff83944460     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_scan_[...]
 53019: ffffffff839446b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 53527: ffffffff83944464     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_init4
 54272: ffffffff83944670     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 54283: ffffffff839448d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ged_d[...]
 54499: ffffffff83944468     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_adxl_init4
 55949: ffffffff839448d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 55976: ffffffff839448dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 56015: ffffffff839448e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 56057: ffffffff839448e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 56223: ffffffff839448e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 56315: ffffffff839448ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hmat_init6
 56372: ffffffff839448f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 56442: ffffffff839448f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_acpi_[...]
 56454: ffffffff839448f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bgrt_init6
 56699: ffffffff839448fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_erst_init6
 56747: ffffffff83944b4c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bert_init7
 56762: ffffffff83944900     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ghes_init6
 56846: ffffffff83944904     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 56861: ffffffff83944908     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 56868: ffffffff8394490c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 56881: ffffffff83944910     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 56898: ffffffff83944914     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 56912: ffffffff83944918     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_chtdc[...]
 56933: ffffffff8394446c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pnp_init4
 57098: ffffffff83944674     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pnp_s[...]
 57110: ffffffff83944678     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pnpac[...]
 57343: ffffffff83944bc0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_clk_d[...]
 57405: ffffffff83944b50     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_clk_d[...]
 57852: ffffffff8394491c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_gpio_[...]
 57871: ffffffff83944920     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_plt_c[...]
 57884: ffffffff83944924     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fch_c[...]
 57897: ffffffff839442f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dma_c[...]
 57974: ffffffff839442fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dma_b[...]
 58190: ffffffff83944928     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dw_pc[...]
 58274: ffffffff839441c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_virti[...]
 58426: ffffffff83944b54     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_setup[...]
 58551: ffffffff83944224     0 NOTYPE  LOCAL  DEFAULT   41 __initcall___gnt[...]
 58639: ffffffff83944470     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ballo[...]
 58674: ffffffff83944474     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_s[...]
 59138: ffffffff8394492c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xenbu[...]
 59139: ffffffff83828068   105 FUNC    LOCAL  DEFAULT   37 xenbus_probe_initcall
 59140: ffffffff83944260     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xenbu[...]
 59194: ffffffff83944478     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xenbu[...]
 59224: ffffffff83944930     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xenbu[...]
 59243: ffffffff83944934     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xenbu[...]
 59255: ffffffff8394447c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xenbu[...]
 59257: ffffffff83944b58     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_boot_[...]
 59302: ffffffff83944300     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
 59317: ffffffff83944480     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_a[...]
 59336: ffffffff83944304     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_p[...]
 59396: ffffffff83944938     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hyper[...]
 59398: ffffffff8394493c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hyper[...]
 59451: ffffffff83944940     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_platf[...]
 59488: ffffffff83944944     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_l[...]
 59558: ffffffff83944948     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pvcal[...]
 59595: ffffffff83944484     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init4
 59769: ffffffff839441c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regul[...]
 59771: ffffffff83944bc4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regul[...]
 60333: ffffffff83944264     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tty_c[...]
 60726: ffffffff8394494c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_n_nul[...]
 60738: ffffffff83944950     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pty_init6
 60790: ffffffff83944954     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sysrq[...]
 61068: ffffffff83944bc8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_con_init
 61091: ffffffff83944268     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_vtcon[...]
 61294: ffffffff83944bcc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hvc_c[...]
 61361: ffffffff83944958     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_h[...]
 61363: ffffffff83944bd0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xen_c[...]
 61576: ffffffff83944bd4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_univ8[...]
 61590: ffffffff8394495c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_seria[...]
 61862: ffffffff83944960     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_seria[...]
 62001: ffffffff83944964     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lpss8[...]
 62022: ffffffff83944968     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mid82[...]
 62075: ffffffff83944bd8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kgdbo[...]
 62077: ffffffff8394496c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 62189: ffffffff8394426c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_serde[...]
 62249: ffffffff8394467c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_chr_d[...]
 62605: ffffffff83944488     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_misc_init4
 62623: ffffffff83944970     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hpet_init6
 62662: ffffffff83944974     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_nvram[...]
 62725: ffffffff83944978     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_agp_init6
 63162: ffffffff8394497c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_amd_i[...]
 63186: ffffffff83944b5c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dmar_[...]
 63563: ffffffff839446c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ir_de[...]
 63604: ffffffff8394448c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iommu[...]
 63753: ffffffff839441c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iommu[...]
 64031: ffffffff83944270     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iommu[...]
 64065: ffffffff83944308     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_iommu[...]
 64811: ffffffff83944980     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_drm_k[...]
 65578: ffffffff83944984     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_drm_c[...]
 67514: ffffffff83944274     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mipi_[...]
 67572: ffffffff83944490     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_vga_a[...]
 67718: ffffffff83944494     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cn_init4
 67727: ffffffff83944988     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cn_pr[...]
 67735: ffffffff839441cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_compo[...]
 67794: ffffffff83944278     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_devli[...]
 67805: ffffffff83944b60     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sync_[...]
 68231: ffffffff83944b64     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_defer[...]
 68276: ffffffff83a2c740     1 OBJECT  LOCAL  DEFAULT   60 initcalls_done
 68806: ffffffff8394498c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_topol[...]
 69005: ffffffff83944990     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cache[...]
 69090: ffffffff8394427c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_softw[...]
 69483: ffffffff8205ebb5   131 FUNC    LOCAL  DEFAULT    1 initcall_debug_s[...]
 69622: ffffffff83944280     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wakeu[...]
 69660: ffffffff83944284     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wakeu[...]
 69700: ffffffff839441d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_early[...]
 69702: ffffffff83944b68     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_late_[...]
 69715: ffffffff83944b6c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_genpd[...]
 69747: ffffffff83944b70     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_genpd[...]
 69941: ffffffff83944288     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_isa_b[...]
 69981: ffffffff83944680     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_firmw[...]
 70049: ffffffff8394428c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
 70312: ffffffff83944290     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regma[...]
 70313: ffffffff8383520e    13 FUNC    LOCAL  DEFAULT   37 regmap_initcall
 70736: ffffffff83944994     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_devco[...]
 70780: ffffffff83944998     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pvpan[...]
 70803: ffffffff83944498     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pm860[...]
 70912: ffffffff8394499c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_htcpl[...]
 71130: ffffffff8394449c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wm840[...]
 71255: ffffffff839444a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wm831[...]
 71266: ffffffff839444a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wm831[...]
 71346: ffffffff839444a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wm835[...]
 71354: ffffffff839444ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tps65[...]
 71376: ffffffff839444b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tps80[...]
 71420: ffffffff839449a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_twl_d[...]
 71507: ffffffff839449a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_twl40[...]
 71549: ffffffff839449a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_twl60[...]
 71632: ffffffff839444b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ezx_p[...]
 71684: ffffffff839444b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_da903[...]
 71780: ffffffff839444bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_da905[...]
 71790: ffffffff839444c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_da905[...]
 71813: ffffffff839444c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_lp878[...]
 71855: ffffffff839444c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_da905[...]
 71865: ffffffff839444cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_max77[...]
 71950: ffffffff839444d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_max89[...]
 71982: ffffffff839444d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_max89[...]
 72030: ffffffff839444d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_max89[...]
 72080: ffffffff839449ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_adp55[...]
 72129: ffffffff839444dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tps65[...]
 72163: ffffffff839444e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tps65[...]
 72177: ffffffff839444e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_aat28[...]
 72209: ffffffff839444e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_palma[...]
 72234: ffffffff839444ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rc5t5[...]
 72281: ffffffff83944294     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sysco[...]
 72293: ffffffff839444f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_as371[...]
 72308: ffffffff839449b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 72336: ffffffff839449b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cht_w[...]
 72386: ffffffff839444f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_libnv[...]
 73288: ffffffff839444f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dax_c[...]
 73447: ffffffff83944b74     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hmem_init7
 73507: ffffffff839444fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dma_b[...]
 73748: ffffffff83944500     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dma_h[...]
 73782: ffffffff839449b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_syste[...]
 73789: ffffffff839449bc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_add_d[...]
 73818: ffffffff839449c0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_udmab[...]
 73883: ffffffff83944504     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_scsi4
 74741: ffffffff839449c4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_sd6
 75012: ffffffff839449c8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_nvme_[...]
 75344: ffffffff839449cc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_nvme_init6
 75612: ffffffff83944508     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ata_init4
 76412: ffffffff839449d0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ahci_[...]
 76752: ffffffff83944298     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_spi_init2
 77086: ffffffff839449d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_net_o[...]
 77099: ffffffff839449d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_black[...]
 77252: ffffffff8394450c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_usb_c[...]
 77331: ffffffff83944510     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_usb_init4
 78721: ffffffff839449dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ehci_[...]
 78941: ffffffff839449e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ehci_[...]
 78983: ffffffff839449e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ohci_[...]
 79123: ffffffff839449e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ohci_[...]
 79161: ffffffff839449ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_uhci_[...]
 79286: ffffffff839449f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xhci_[...]
 80121: ffffffff839449f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_usb_s[...]
 80377: ffffffff839449f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_kgdbd[...]
 80412: ffffffff83944514     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xdbc_init4
 80529: ffffffff83944518     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_input[...]
 80752: ffffffff839449fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_input[...]
 80766: ffffffff83944a00     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_evdev[...]
 80851: ffffffff8394451c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rtc_init4
 81095: ffffffff83944a04     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cmos_init6
 81237: ffffffff8394429c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_i2c_init2
 81714: ffffffff83944520     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dw_i2[...]
 81736: ffffffff83944a08     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dw_i2[...]
 81800: ffffffff83944524     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_rc_co[...]
 81995: ffffffff83944528     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cec_d[...]
 82144: ffffffff8394452c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pps_init4
 82247: ffffffff83944530     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ptp_init4
 82301: ffffffff83944a0c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_mt632[...]
 82314: ffffffff83944a10     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_resta[...]
 82409: ffffffff83944534     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_power[...]
 82500: ffffffff83944b78     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_charg[...]
 82581: ffffffff83944538     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hwmon[...]
 82687: ffffffff839442a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_therm[...]
 83056: ffffffff839445ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_watch[...]
 83153: ffffffff83944a14     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_watch[...]
 83394: ffffffff8394453c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_edac_init4
 83777: ffffffff839441d4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_opp_d[...]
 83899: ffffffff839441d8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpufr[...]
 84145: ffffffff839441dc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpufr[...]
 84154: ffffffff839441e0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpufr[...]
 84163: ffffffff839441e4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpufr[...]
 84193: ffffffff839441e8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_CPU_F[...]
 84228: ffffffff839441ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_CPU_F[...]
 84310: ffffffff83944a18     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 84504: ffffffff839441f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cpuid[...]
 84595: ffffffff839442a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 84604: ffffffff839442a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_menu2
 84613: ffffffff839442ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_teo_g[...]
 84622: ffffffff839442b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 84740: ffffffff83944540     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_leds_init4
 84826: ffffffff83944a1c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ledtr[...]
 84837: ffffffff83944a20     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ledtr[...]
 84846: ffffffff83944a24     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ledtr[...]
 84859: ffffffff83944a28     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ledtr[...]
 84870: ffffffff83944544     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dmi_init4
 84950: ffffffff83944a2c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dmi_s[...]
 85021: ffffffff8394430c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dmi_i[...]
 85064: ffffffff83944b7c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_firmw[...]
 85095: ffffffff83944548     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efisu[...]
 85097: ffffffff839440ec     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efi_m[...]
 85099: ffffffff83944b80     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_regis[...]
 85222: ffffffff83944b84     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efi_s[...]
 85244: ffffffff839441f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_capsu[...]
 85260: ffffffff83944a30     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_esrt_[...]
 85305: ffffffff83944a34     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efiva[...]
 85433: ffffffff83944684     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_map_p[...]
 85442: ffffffff83944b88     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efi_r[...]
 85473: ffffffff839440f0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efi_e[...]
 85475: ffffffff83944b8c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_efi_e[...]
 85507: ffffffff83944688     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 85673: ffffffff83944a38     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hid_init6
 85906: ffffffff83944a3c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_hid_g[...]
 85916: ffffffff83944310     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ts_dm[...]
 86018: ffffffff83944b90     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_itmt_[...]
 86034: ffffffff83944a40     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pmc_c[...]
 86128: ffffffff83944a44     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pmc_c[...]
 86175: ffffffff8394454c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 86195: ffffffff83944a48     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_intel[...]
 86209: ffffffff83944a4c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pmc_a[...]
 86392: ffffffff839442b4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pcc_init2
 86561: ffffffff83944550     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_remot[...]
 86829: ffffffff83944554     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_devfr[...]
 86987: ffffffff83944558     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_devfr[...]
 87069: ffffffff83944a50     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_extco[...]
 87142: ffffffff8394468c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_power[...]
 87181: ffffffff839440f4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_idle_[...]
 87197: ffffffff8394455c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ras_init4
 87395: ffffffff83944b94     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cec_init7
 87508: ffffffff83944560     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_nvmem[...]
 87628: ffffffff83944a54     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_icc_init6
 87798: ffffffff839441f8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sock_init1
 88374: ffffffff839441fc     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_net_i[...]
 88385: ffffffff83944564     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_proto[...]
 88997: ffffffff83944200     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_net_d[...]
 89014: ffffffff8394410c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_net_n[...]
 89161: ffffffff83944204     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 89181: ffffffff83944690     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sysct[...]
 89786: ffffffff83944568     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_net_d[...]
 90305: ffffffff8394456c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_neigh[...]
 90965: ffffffff83944a58     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sock_[...]
 91051: ffffffff83944570     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fib_n[...]
 91598: ffffffff83944208     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_netpo[...]
 91659: ffffffff83944574     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fib_r[...]
 92444: ffffffff83944a5c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 92525: ffffffff83944578     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_init_[...]
 92604: ffffffff8394457c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_l[...]
 92630: ffffffff83944b98     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_s[...]
 93055: ffffffff83944580     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_devli[...]
 93359: ffffffff83944b9c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bpf_s[...]
 93448: ffffffff83944694     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_eth_o[...]
 93573: ffffffff83944ba0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sch_d[...]
 93632: ffffffff83944584     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pktsc[...]
 93723: ffffffff83944a60     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_black[...]
 93829: ffffffff83944588     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tc_fi[...]
 93995: ffffffff8394458c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tc_ac[...]
 94072: ffffffff83944a64     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_fq_co[...]
 94199: ffffffff8394420c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_netli[...]
 94284: ffffffff83944210     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_genl_init1
 94453: ffffffff83944590     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ethnl[...]
 95911: ffffffff83944ba4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tcp_c[...]
 96559: ffffffff83944698     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ipv4_[...]
 96561: ffffffff8394469c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_inet_init5
 97093: ffffffff83944a68     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_gre_o[...]
 97138: ffffffff83944594     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_nexth[...]
 97225: ffffffff83944a6c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_sysct[...]
 97486: ffffffff83944a70     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cubic[...]
 97539: ffffffff83944214     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_tcp_b[...]
 97554: ffffffff83944218     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_udp_b[...]
 97561: ffffffff83944598     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_cipso[...]
 98177: ffffffff83944a74     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xfrm_[...]
 98289: ffffffff839446a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_af_un[...]
 98435: ffffffff83944a78     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_inet6[...]
100102: ffffffff839446a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ipv6_[...]
100171: ffffffff83944a7c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_packe[...]
100278: ffffffff83944a80     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_strp_[...]
100329: ffffffff839446a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_vlan_[...]
100351: ffffffff8394459c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_wirel[...]
100440: ffffffff839445a0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_netlb[...]
100562: ffffffff83944a84     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_dcbnl[...]
100887: ffffffff839445a4     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_ncsi_[...]
100941: ffffffff839446ac     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_xsk_init5
101378: ffffffff839446b0     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pcibi[...]
101393: ffffffff83944314     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_a[...]
101413: ffffffff83944ba8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_m[...]
101552: ffffffff839445a8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pci_s[...]
101653: ffffffff839442b8     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_amd_p[...]
101663: ffffffff8394421c     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_bsp_p[...]
101665: ffffffff83944a88     0 NOTYPE  LOCAL  DEFAULT   41 __initcall_pm_ch[...]
105396: ffffffff839440f8     0 NOTYPE  GLOBAL DEFAULT   41 __initcall0_start
105644: ffffffff83944bc8     0 NOTYPE  GLOBAL DEFAULT   41 __initcall_end
106221: ffffffff839446bc     0 NOTYPE  GLOBAL DEFAULT   41 __initcallrootfs[...]
107574: ffffffff839442bc     0 NOTYPE  GLOBAL DEFAULT   41 __initcall3_start
107835: ffffffff83944318     0 NOTYPE  GLOBAL DEFAULT   41 __initcall4_start
108248: ffffffff83944bdc     0 NOTYPE  GLOBAL DEFAULT   41 __con_initcall_end
110902: ffffffff8397f000     1 OBJECT  GLOBAL DEFAULT   60 initcall_debug
111229: ffffffff839446c8     0 NOTYPE  GLOBAL DEFAULT   41 __initcall6_start
112250: ffffffff839445b0     0 NOTYPE  GLOBAL DEFAULT   41 __initcall5_start
114898: ffffffff83944090     0 NOTYPE  GLOBAL DEFAULT   41 __initcall_start
116407: ffffffff83944a90     0 NOTYPE  GLOBAL DEFAULT   41 __initcall7_start
119572: ffffffff83944110     0 NOTYPE  GLOBAL DEFAULT   41 __initcall1_start
120747: ffffffff81003be0   540 FUNC    GLOBAL DEFAULT    1 do_one_initcall
121425: ffffffff83944bc8     0 NOTYPE  GLOBAL DEFAULT   41 __con_initcall_start
121729: ffffffff83944228     0 NOTYPE  GLOBAL DEFAULT   41 __initcall2_start
```
