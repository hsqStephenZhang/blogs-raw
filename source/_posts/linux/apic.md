---
title: x86 APIC
categories: [linux, interrupt]
date: 2022-1-21 10:54:45
---

## 1. 什么是 APIC

APIC 的全程是 Advanced Programmable Interrupt Controller，高级可编程中断控制器，我们今天主要讨论 x86 平台上的 APIC。

主要关注两点：

1. 怎么高级了
2. 什么是可编程

在这之前，首先认识一下 Interrupt 是什么

## 2. APIC details

### 2.1 中断，陷阱，异常

说起中断，大家经常将其和异常混淆，我觉得这里有必要解释一下。

常说的操作系统中的中断处理程序，其实应该叫做异常处理程序，而**异常**分为软件异常和硬件异常，前者是同步产生的，主要由 CPU 执行指令的过程中，触发的一些异常状况，比如除零异常，int 3 异常 ...，也可以统称为陷阱；后者则是异步产生的，也就是我们所说的 Interrupt，外部硬件给 CPU 发送一个信号，通过一些线路传出，交给 CPU 处理。

将异常再进行细分，还可以分为三类：

1. Fault
2. Trap
3. Abort

Fault 是可以在指令执行之前就检测出来的异常，如果检测并纠正成功，就能继续执行，比如 Divide Error，Invalid Opcode

Trap 必须要执行完指令才能触发，并且会立即上报 Trap，同样，也是可以像 Fault 一样继续执行的，比如 int 3，Breakpoint，Overflow

Abort 我们平时打交道比较少，只需要明白，这是最严重的一种情况，不会上报，也不允许程序继续执行，比如 Alignment Check，Machine Check

### 2.2 从 PIC 到 APIC

但是，上面所说的中断信号并不是直接传输到 CPU 的，老式机器上，有一个专用的 PIC 芯片，负责接收外围设备的 Interrupt request，然后交给 CPU，而现代负责这一功能的设备则是 APIC，advanced one。

一个 APIC，主要由两个部分组成：

1. Local APCI
2. IO APIC

Local APIC 是 per-cpu 的硬件，负责处理各自 CPU 上的一些本地 IO 中断，最典型的就是 APIC-timer

IO APIC 则为所有的核心同时提供外围设备的 IO 中断管理

![apic](/images/apic.png)

### 2.3 中断处理

中断发生之后，需要立刻交由操作系统得到处理，那什么才是一个中断的完成处理过程呢？应该分为下面几个步骤

1. 保存当前 cpu core 正在执行的程序状态
2. kernel 找到对应的处理程序，调用该程序
3. 中断处理程序执行结束，恢复原先程序的状态

所有中断对应程序的地址，就保存在 Interrupt Descriptor Table（IDT）中，并且，CPU 通过唯一的序号来标识每一种中断，这个序号就称为 vector number，中断向量号，IDT 也可以称为中断向量表

虽然名字中有 Descriptor 这个词，但是 IDT 的表项一般被称为 gate，包含了以下三种：

1. Interrupt Gates
2. Task Gates
3. Trap Gates

类似于 Global Descriptor Table，IDT 也对应了一个特殊的寄存器 IDTR，用来保存其地址，同时，也提供了两条指令 lidt/sidt，分别用来设置和读取 IDRT 的值，可以查看 `struct paravirt_patch_template pv_ops`，其中的 `.cpu.load_idt` 就设置的下面这个回调函数

```c
static __always_inline void native_load_idt(const struct desc_ptr *dtr)
{
    asm volatile("lidt %0"::"m" (*dtr));
}
```

同时，系统初始化的时候，也会通过 lidtl 指令来设置 IDTR 寄存器中（同时也表明 LDT 是可以为空的，不必要像 GDT 一样一次性设置好），这里也插一嘴，IDT 可能存在于地址空间的任意位置

```c
static void setup_idt(void)
{
    static const struct gdt_ptr null_idt = {0, 0};
    asm volatile("lidtl %0" : : "m" (null_idt));
}
```

LDTR 是一个 48 bits 的寄存器，对应下面的结构

```c
struct gdt_ptr {
        u16 len;  // 长度限制
        u32 ptr;  // base 地址，IDT 地址
} __attribute__((packed));
```

接下来看一下 IDT entry，也就是 **gate** 的结构

```c
struct idt_bits {
    u16     ist    : 3,
            zero    : 5,
            type    : 5,
            dpl    : 2,
            p    : 1;
} __attribute__((packed));

struct gate_struct {
    u16        offset_low;
    u16        segment;
    struct idt_bits    bits;
    u16        offset_middle;
#ifdef CONFIG_X86_64
    u32        offset_high;
    u32        reserved;
#endif
} __attribute__((packed));
```

这里的 ist 值得详细解释一下
