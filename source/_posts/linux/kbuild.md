---
title: Kbuild
categories: linux
date: 2022-1-20 23:26:40
---

## 1. 什么是 kbuild

Linux 是一个单体内核，也就意味着整个内核运行在同一个地址空间中，这当然有很多性能上的好处，也同时带来了一个挑战，必须要在编译期间选择想要的特性。

同时，Linux 不是一个纯粹的单体内核（monolithic kernel），因为它可以在运行时使用可加载的内核模块（loadable kernel modules）进行扩展。要加载模块，内核必须包含模块中使用的所有内核符号。如果这些符号在编译时未包含在内核中，则由于缺少依赖项，模块将不会被加载。

因此，能够选择要在 Linux 内核中编译（或不编译）什么代码非常重要。实现这一点的方法是使用**条件编译**。有大量配置选项可用于选择是否包含特定的功能，这被翻译成 C语言中的 define 条件编译参数。

因此，需要一种简单有效的方法来管理所有这些编译选项。负责这个的基础设施——构建内核映像及各个模块的工具——被称为 Kbuild，也就是内核编译系统

## 2. Kbuild Details

关于 Kbuild 的详细信息，可以去 [Kbuild kernel document](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt) [kernel document 中文翻译](https://blog.51cto.com/weiguozhihui/1591397) 查阅，这里简要说明一下，然后通过一些示例来看看 Kbuild 是如何具体应用的。

### 2.1 Kbuild main components

Kbuild system 有下面几个主要的组成部分：

1. Config symbols: 编译选项，用来决定是否需要编译某些文件，以及如何编译这些文件，以及是否要在最终生成的镜像中包含二进制对象文件 `xxx.o`

2. Kconfig files: 定义了所有 `CONFIG_XXX` 符号，以及它们的属性，比如类型，描述信息，依赖配置，通过 `make menuconfig` 可以根据这些信息生成树状的结构，从而可视化编译每一个符号的值

3. .config file: 保存了所有 `CONFIG_XXX` 符号的值，可以手动修改这些值，从而开启（关闭）某些特性，也可可以通过 `make menuconfig` `make xconfig` 来进行可视化编辑，然后保存为最新的 .config 文件，相信大家编译内核的时候，应该已经尝试过这项功能了

4. Makefiles: 普通的 GNU makefile

### 2.2 Kbuild 项目构成

一个完整的 Kbuild 项目，以 [github menuconfig demo](https://github.com/gxku/menuconfig.git) 为例，主要由 Kbuild ，Makefile，kconfig/xxx，src/xxx 组成

Kconfig 用来描述项目的依赖关系，而 kconfig 文件夹对于每个项目来说，都是相同的（可以从 Linux 源码的 kernel/scripts/kconfig 复制过来），主要是一些词法分析，解析 Kconfig 以及其它配置相关的工具。

先来看大家较为熟知的 `make menuconfig`，它会通过 ncurses 显示一个图形化菜单，供大家勾选一些编译选项，完成之后生成一个 `.config` 文件。那么，这个菜单是如何生成的呢？换句话说，菜单中的这么多项目，从何而来？

这就是 Kconfig 文件的作用了，稍微查看一下 Kconfig 的内容就能明白，其中的 `mainmenu "xxx"` `menuconfig TEST1` 等内容，就是之前显示的图形化菜单的结构化描述。实际上，如果查看 Makefile 可以发现，menuconfig 是通过执行 `kconfig/mconf Kconfig` 得到的，这也就是为什么要包含这样一个 `kconfig/` 文件夹的原因了。

完成了 `make menuconfig` 之后，生成了一个 `.config` 文件，仔细查看这个文件会发现，它包含了我们想要的所有编译选项，以 `CONFIG_XXX=?` 的格式存放，

```config
CONFIG_TEST1=y
CONFIG_LED_ON=y
```

并且，如果是 menuconfig 中没有配置的选项，对应的内容则为

```config
# CONFIG_LED_OFF is not set
```

接下来就是真正的编译了，在这之前，Kbuild 系统会通过 `kconfig/conf Kconfig` 递归读取所有的 Kconfig 配置（.config 文件），并且生成一个头文件 `include/generated/autoconf.h`，这个文件中包含了所有配置的符号，如 `CONFIG_FTRACE` `CONFIG_MODULES`，gcc 在编译的时候，都会将这个头文件先导入进来，这就完成了条件编译的目的。

通过 `cat include/generated/autoconf.h` 查看自动生成的头文件内容，如下所示

```c
/*
 *
 * Automatically generated file; DO NOT EDIT.
 * Main Menu
 *
 */
#define CONFIG_LED_ON 1
#define CONFIG_TEST2 1
#define CONFIG_NUM_PARAM2 10086
#define CONFIG_TEST1 1
#define CONFIG_OPTIONAL_PARAM "this shown if TEST1 is set"
#define CONFIG_NUM_PARAM 55
#define CONFIG_STRING_PARAM "this is default string."
```

然后就会真正执行 Makefile 中的命令，递归地进入到子目录中，执行下面的 Makefile 命令，子文件夹构建完成之后，再退回到上一级

### 2.3 Useful Tips

#### 2.3.1 obj && hostprogs

在内核模块的编译时候，经常会看到 `obj-m` `hostprogs-y` `always += xxx` 的 Makefile 指令，这些有什么作用呢？

之前提到过，内核是以 `单体+可插拔模块` 的方式编译的，一个功能，可以直接包含到最终生成的 vmlinux 中，也可以通过 loadable modules 的形式，按需加载，为了方便这种编译，Kbuild 就提供了两种模式：'y' 和 'm'

如果 `objs-y += demo.o`，那么 demo.c 编译得到的 demo.o 就会包含在最终生成的可执行文件中（还需要看上一级目录是否也 `objs-y += xxx` 了），如果 `objs-m += demo.o`，那么 demo.o 就会以内核模块的方式编译，自行加载

但在 Linux 源码中，我们经常可以看到 `objs-$(CONFIG_XXX) += xxx.o`，这就是根据特定功能的配置，来决定是直接编译到内核中，还是编译为模块，当 `CONFIG_XXX=y` 的时候，就是前者，`CONFIG_XXX=m` 的时候，就是后者，当然，如果不想要编译这个文件，就设置 `CONFIG_XXX=n`，或者什么都不做（对应了 .config 文件中的 `# CONFIG_LED_OFF is not set`）

至于 `hostprogs-y hostprogs-m` 也是类似的道理，只不过，hostprogs 是编译过程中需要执行的一些程序，比如 recordmcount 这种，对于编译的中间结果做一些 **black magic**

不论是 `obj-y` 还是 `hostprogs-y`，都有办法让它们强制编译，那就是加上一个 **always** 标签，比如

```Makefile
hostprogs-always-$(BUILD_C_XXX)     += recordmcount

always-y += test1
```

#### 2.3.2 flags && options

Kbuild 有一些默认的规则：

1. ccflags-y,是指定给 $(CC) 的编译选项；

2. asflags-y,是指定给 $(AS) 的编译选项；

3. ldflags-y,是指定给 $(LD) 的编译选项；

结合一些 `CONFIG_XXX`，就能灵活地设置编译器的参数

`cc-option` 也是用来判读提供给 $(CC) 的选项是否被支持，`cc-ldoption` 来判断提供给链接器的选项是否被支持 ...，这样就可以最大程度支持不同版本的编译器

## 2. 参考文档

最好的资料当然还是 kernel 文档了，推荐认真读两遍，会对 Kbuild 系统有更加深入的了解

[Kbuild kernel document](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
