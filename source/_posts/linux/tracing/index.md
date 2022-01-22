---
title: linux tracing
categories: [linux, trace]
date: 2021-11-23 23:21:13
---

## 1. Summerize

Linux 中的 tracing 机制非常丰富，比如最火热的 ebpf，还有老牌的 perf，systemtap，kprobe，tracepoint，uprobe，usdt，bcc，ftrace，这些花样繁多的手段，其实都可以完成一定程度上的 tracing 功能，但是也给我们带来了幸福的烦恼：选择困难！

看上去每一种方式都可以解决一定的问题，他们之间还有很多功能重叠的地方，如果不将各自的职责做一个详细的划分，就无法做到因地制宜地分析性能瓶颈。

首先要明确一点，tracing 机制和操作系统息息相关，Linux 内核中广发采用的事件通知机制，也在整个 tracing 系统中得到了广泛引用。

针对 tracing 这个场景，一般来说需要几个角色：

1. Data source mechanism         -- 数据源
2. Data-collecting mechanism    -- 数据收集机制
3. visualize tools                            -- 可视化机制

接下来就一一探究各个 tracing 功能分别对应哪个角色。

## 2. Data source

Data source 是真正产生 tracing 事件的底层机制。

不管 tracing 在外界看来有多透明，对于 CPU 而言，也仅仅是执行一段代码而已，或是调用了特定的函数，或有一些特殊的异常执行流，但不可能超出`instruction` 这个范畴。

如果你之前修改并编译过 kernel 代码，一定会感觉这个过程无比繁琐：定位一个问题，可能要反反复复很多次，最后甚至还无法定位问题所在。但很多人不知道的是， kernel 早就为提供了“**无需编译，直接插入逻辑，甚至是热更新**”的强大功能：

- 静态打桩 tracepoint
- 动态打桩 kprobe/kretprobe

用户态程序，实际上也在内核的掌控之中，因此也有相似的两种机制：

- 静态打桩 USDT
- 动态打桩 uprobe/uretprobe

更往底层来说，kernel还支持收集硬件的程序计数器 performance monitor counter (PMU)，提供更加精细的采样数据。

这些都是**事件源**，离开了事件源，其余的都无从谈起。

### 2.1 静态打桩

之所以说是静态，是因为打桩点已经编译到二进制可执行文件中。

```bash
Tips

打桩实现是基于 hooks 的思想，在代码中提前做了插桩，代码运行到跟踪点（桩）上注册了钩子函数（hook），那么就会执行钩子函数达到调试的目的

```

kernel 中的静态打桩点也被称为 tracepoint，有专门的开发人员维护，一般不会变动，非常稳定。内核可以通过 enable & disable 的方式打开或者关闭 tracepoint，在 off 状态下，几乎没有开销，只有一个分支判断加上几条指令的 penalty，近乎于 zero-cost，只有在打开的时候，用户提供给内核的函数才会被执行。

最常用的 tracepoint 就是系统调用了，syscall 都实现了对应的 tracepoint，开启这些探测点，就能知道系统调用的执行情况

类似tracepoint，用户程序也可以自己维护一套静态，稳定的打桩点，这被称为 user statically defined tracing (USDT)。USDT 其实是从Dtrace 中引入的设计思路，自从 Dtrace 出现之后，各种各样的应用都会在程序的关键位置加入 USDT，从而暴露 probe 的能力。

### 2.2 动态打桩

kprobe 是另外一种基于打桩机制实现的调试手法。kprobes的底层原理是：在程序运行的过程当中，动态修改内存中的指令，触发 int3 异常，再调用注册的 hook 函数执行，最后恢复原先的指令。因为可以在运行期间动态修改指令，赋予了kprobe极大的灵活性，几乎可以运用在任何地方。

但是，一定要铭记一点：计算机科学中**没有银弹**。kprobes 非常强大，也非常灵活，随之而来的代价是性能上的开销。因为每一次kprobe 会触发两次指令执行异常，并且，如果 kprobe attach 的函数发生了变动（比如函数名更改），该 probe 就会失效，没有tracepoint 那种稳定性保证。

uprobe 和 kprobe 原理类似，主要是用来追踪用户态程序，比如 malloc 这种容易发生问题的用户态底层组件。

至于 kretprobe 和  uretprobe，其实功能也比较类似，依赖于 kprobe(uprobe)，通过 trampoline 跳板代码，可让用户可以在某一个函数执行的首尾分别执行 pre-handler & post-hander，达到了对于某一个函数 probe 的效果，可以方便地统计函数执行时长等情况。

## 3. Collecing mechanisms

数据源的基本原理和大致划分已经看完了，并没有我们想象中那么复杂，无非划分为 动态/静态，用户态/内核态。接下来的数据收集机制才是 tracing system 的重中之重。

数据收集机制，顾名思义，是需要借助底层的 Data source，收集我们感兴趣的数据，才能进一步展示。

有了数据源做支撑，这里需要做的事情是：

1. 注册。告诉数据源，我们感兴趣的事件，请处理并将发送回来
2. 收集。收集，通过某种交互方式，将内核收集到的内容传递到用户手中。

接下来会对主流的 collecting mechanisms 做一个大致的对比

### 3.1 ftrace

ftrace 的原理是使用 gcc 编译器的 `-pg` 选项，达到在内核代码中插桩的目的。该选项会自动在函数的入口处加上对 mcount 的调用指令。这样每次进入一个函数时都可以调用对应的钩子函数来打印函数信息。事实上，对于 mcount 的函数调用，在 99%的情况下，都是无用的，也会对性能有极大的损耗，因此，默认情况下，在函数开头其实是插入了几条 nop 指令，运行期间，动态修改这个指令，就能达到 trace function call 的目的。

ftrace 通过 tracefs 文件系统和用户进行交互，tracefs 下面有很多特殊的文件，它们都注册了特定的 file_operations，写入操作大致相当于 **读取指令 + 解析指令 + 执行**，TODO!(读取操作...)。

### 3.2 perf events

通过 perf_event_open 系统调用，在用户空间创建了一块ringbuf，内核将收集到的数据写入这块共享内存中，就完成了user-kernel的数据传输

### 3.3 systemp Tap

systemtap 在 kprobe 的基础上，加上脚本解析和内核模块编译运行单元，使开发人员在应用层即可实现 hook 内核，大大简化了开发流程。 工作原理是通过将脚本语句翻译成 C语句，编译成内核模块。 模块加载之后，将所有探测的事件以钩子的方式挂到内核上，当任何处理器上的某个事件发生时，相应钩子上的句柄就会被执行。

### 3.4 eBPF

eBPF 给Linux注入了超能力！内核为eBPF提供了一个虚拟机，可以运行特定格式的代码，并且有各种各样的 hook 点，而且可以借助kprobe，tracepoint，uprobe，USDT 中的任何一种底层机制来收集想要的数据，修改程序中的指令路径，简直无所不能。这一切功能，甚至建立在无需改动内核代码的基础之上，和 Linux 你中有我，我中有你（不是）。

## 4. Front-end

### 4.1 perf

perf trace 是跟踪syscall的非常棒的工具，perf record | report | bench | top | probe 也是非常好用的指令，实用且易于上手。

### 4.2 trace-cmd & kernelshark

ftrace作为老牌tracing工具，在前端方面肯定是少不了的。比如trace-cmd，kernelshark，perf都集成了ftrace的功能。

trace-cmd 非常值得一提，我经常用它来看内核中函数的执行路径，比如执行 show 指令，就可以清晰地看到当前kernel正在执行的函数调用关系，再去对照源码查看，会节省不少工夫。

```C
$ trace-cmd show | head -20

## tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 6)               |        __fget_light() {
 6)   0.804 us    |          __fget_files();
 6)   2.708 us    |        }
 6)   3.650 us    |      } /* __fdget */
 6)   0.547 us    |      eventfd_poll();
 6)   0.535 us    |      fput();
 6)               |      __fdget() {
 6)               |        __fget_light() {
 6)   0.946 us    |          __fget_files();
 6)   1.895 us    |        }
 6)   2.849 us    |      }
 6)               |      sock_poll() {
 6)   0.651 us    |        unix_poll();
 6)   1.905 us    |      }
 6)   0.475 us    |      fput();
 6)               |      __fdget() {
```

但是不管再怎么fancy，trace-cmd 也无法离开ftrace单独使用，其主要依赖的就是通过 debugfs和内核中的ftrace进行交互。

kernelshark 我没有使用过，不过看上去功能也非常丰富，可以更好地可视化展现出来，而不像trace-cmd那样仅仅输出在命令行中。

### 4.3 bcc & bpftrace

eBPF在 collecting mechanisim 这个层面已经做到了顶级，其前端工具也没有令我们失望。iovisor这个项目也做了很多贡献，通过bcc，bpftrace两这个工具，利用脚本**定制化**输出的方式和内容，极大提高了开发者体验，也有很多开箱即用的功能脚本，并且仍在不断扩充当中。
