---
title: how perf works
---



# perf 



## 1. perf 是什么



perf  这个工具大家可能都用过，非常强大，但是也不免让人好奇，底层是如何实现的呢？

官方的定义为：Perf 是访问 processor monitor unit (PMU) 以及记录和展示 software events（例如页面错误）的接口。 它支持系统范围、线程级别和 KVM 虚拟化的各种监控。

但这种解释还是无法让人完全理解，别着急，听我慢慢道来。



首先明确一点：perf 是用来衡量 **计算机/系统性能**的工具。这一点应该没有什么疑问，那么，可以从哪些角度来衡量呢？大致上需要两个维度：

1. 硬件。比如 cache 丢失，流水线停顿，页面交换等
2. 软件。内核产生的事件，分布在各个功能模块中，统计和操作系统相关性能事件。比如系统调用次数、上下文切换次数、任务迁移次数、缺页例外次数等。



后者其实比较好理解，因为只需要内核代码当中添加一些 hack，比如添加 **打点、metrix** 这些工作就好了，但是前者对我们来说，着实有些陌生。

换一个角度考虑，既然内核可以为我们打点，记录一些程序执行情况，那么 CPU 是否也能记录一些更加 low-level 的事件呢？比如 cache miss, branch predict fail, TLB miss，这些硬件上的事件，都在 CPU 掌控之中，CPU 玩去哪有能力统计出来，并汇报给它的上层软件：操作系统。



## 2. perf 如何实现



1. 计数。 计算事件发生的次数。
2. 基于事件的采样。 这是一种不太准确的计数方式：每当发生特定阈值数量的事件时，就会记录一个样本，产生对应的时间。
3. 基于时间的采样。 这是一种不太精确的计数方式：以定义的频率记录样本。
4. 基于指令的采样（仅限 AMD64）。 处理器遵循出现在给定时间间隔内的指令，并对它们产生的事件进行采样。 这允许跟进单个指令并查看哪些指令对性能至关重要。





其实 perf 底层依赖于 PMU 这样一个硬件。PMU 的全称是 Performance Monitoring Unit，也就是性能监控单元。PMU 会初始化一个 硬件性能计数器 PMC（Performance Monitoring Counter），PMC随着指定硬件事件的发生而自动累加。在 PMC 溢出时，PMU 触发一个 PMI（Performance Monitoring Interrupt）中断。内核在 PMI  中断的处理函数中保存下面几个内容：

1.  PMC 的计数值
2. 触发中断时的指令地址
3. 当前时间戳以及当前进程的PID，TID，comm 等信息。



我们把这些信息统称为一个采样（sample）。内核会将收集到的 sample 放入用于跟用户空间通信的 Ring Buffer。用户空间里的 perf  分析程序采用 mmap 机制从 ring buffer 中读入采样，并对其解析。



PMU 是体系相关的一个结构，对于 x86 & intel 而言，核心代码在 arch/x86/events/intel 目录中。

针对 PMI 中断的处理函数便是 intel_pmu_handle_irq

```c
static __initconst const struct x86_pmu intel_pmu = {
	.name			= "Intel",
	.handle_irq		= intel_pmu_handle_irq,
	.disable_all		= intel_pmu_disable_all,
	.enable_all		= intel_pmu_enable_all,
	.enable			= intel_pmu_enable_event,
	.disable		= intel_pmu_disable_event,
   /* other callbacks */
}
```

