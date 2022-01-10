---
title: how perf works
categories: trace
date: 2022-1-9 13:47:40
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



## 2. perf 底层如何实现



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



接下来研究一下 intel_pmu_handle_irq 中究竟做了什么**黑科技**？RTFSC！



这个 handler 的处理其实是一个循环，因为每当我们处理好一次 pmu overflow 之后，可能在这段时间内，重新触发了 overflow，导致我们不得不处理，但也应该防止这个 loop 执行次数过多，也就引入了  loops 变量进行统计。

```c
static int intel_pmu_handle_irq(struct pt_regs *regs)
{
	struct cpu_hw_events *cpuc;
	int loops;
	u64 status;
	int handled;
	int pmu_enabled;

	cpuc = this_cpu_ptr(&cpu_hw_events);
	pmu_enabled = cpuc->enabled;
	loops = 0;
  
  /*
   	setup context here
  */
  
  // again loop
again:
	intel_pmu_lbr_read();
	intel_pmu_ack_status(status);
	if (++loops > 100) {
		static bool warned;

		if (!warned) {
			WARN(1, "perfevents: irq loop stuck!\n");
			perf_event_print_debug();
			warned = true;
		}
		intel_pmu_reset();
		goto done;
	}

	handled += handle_pmi_common(regs, status);

	status = intel_pmu_get_status();
	if (status)
		goto again;

done:
	cpuc->enabled = pmu_enabled;
	if (pmu_enabled)
		__intel_pmu_enable_all(0, true);
	intel_bts_enable_local();

	// return
}
```



如果将一些干扰因素抽离，就可以得到下面这个主干逻辑。就是一个不断处理 pmi 中断的过程。核心逻辑在 `handle_pmi_common` 中

```c
static int intel_pmu_handle_irq(struct pt_regs *regs)
{
 	// 1. set up context
  
  // 2. loop
  do {
    // core logic 
		handled += handle_pmi_common(regs, status);
  } while (intel_pmu_get_status()!=0)
}
```



可以看出，`handle_pmi_common` 主要分为三个步骤进行处理

1. processor trace 
2. perf metrics
3. perf event overflow 

```c
static int handle_pmi_common(struct pt_regs *regs, u64 status)
{
	// 上下文环境准备

	/*
	 * Intel PT(Processor Trace) 处理
	 */
	if (__test_and_clear_bit(GLOBAL_STATUS_TRACE_TOPAPMI_BIT, (unsigned long *)&status)) {
		handled++;
		if (unlikely(perf_guest_cbs && perf_guest_cbs->is_in_guest() &&
			perf_guest_cbs->handle_intel_pt_intr))
			perf_guest_cbs->handle_intel_pt_intr();
		else
			intel_pt_interrupt();
	}

	/*
	 * Intel Perf mertrics 处理
	 */
	if (__test_and_clear_bit(GLOBAL_STATUS_PERF_METRICS_OVF_BIT, (unsigned long *)&status)) {
		handled++;
		if (x86_pmu.update_topdown_event)
			x86_pmu.update_topdown_event(NULL);
	}

	status |= cpuc->intel_cp_status;

  // 针对每个 overflow bit，都要进行对应的处理，上报 perf event
	for_each_set_bit(bit, (unsigned long *)&status, X86_PMC_IDX_MAX) {
		struct perf_event *event = cpuc->events[bit];

		handled++;

		if (!test_bit(bit, cpuc->active_mask))
			continue;

		if (!intel_pmu_save_and_restart(event))
			continue;

		perf_sample_data_init(&data, 0, event->hw.last_period);

		if (has_branch_stack(event))
			data.br_stack = &cpuc->lbr_stack;

		if (perf_event_overflow(event, &data, regs))
			x86_pmu_stop(event, 0);
	}

	return handled;
}
```



```bash
Tips:
		Intel Processor Trace（PT）是Intel在其第五代 CPU 之后引入的一个硬件部件。该硬件部件的作用是记录程序执行中的分支信息，从而帮		助构建程序运行过程中的控制流图。在默认情况下 CPU 的 PT 部件是处于关闭状态，这意味着 CPU 不会记录程序的分支信息，因此也不会产生		任何开销。通过写 MSR 寄存器可以打开 PT 开关。在打开 PT 开关后，CPU开始记录分支指令信息，所记录的信息以压缩数据包的形式存储在内		 存中
```



硬件相关的机制我们可能不是很熟悉，但是 perf 采集的数据上报给用户的过程，这总得看一看呗。在针对 overflow bit 的处理逻辑末尾，都会调用 `perf_event_overflow` 函数，看到这个名字也不难猜出意思：perf 溢出事件。来看其核心处理逻辑。



`__perf_event_overflow ` 主要逻辑很简单，调用 event->overflow_handler，而这个 event 又是什么呢？这个 overflow_handler 在哪里设置的呢？嘿嘿，**`sys_perf_event_open`** 系统调用！

```c
static int __perf_event_overflow(struct perf_event *event,
				   int throttle, struct perf_sample_data *data,
				   struct pt_regs *regs)
{
	// 准备工作 ...

  // 调用 overflow_handler
	READ_ONCE(event->overflow_handler)(event, data, regs);

  // fasync 同步刷新，如果没有成功，需要提交到 work_queue 中 
	if (*perf_event_fasync(event) && event->pending_kill) {
		event->pending_wakeup = 1;
		irq_work_queue(&event->pending);
	}

	return ret;
}
```



深挖了这么多层，终于找到了和 perf 相关的系统调用了！在 perf_event_open 系统调用中，会通过 `perf_event_alloc` 分配一个 `perf event` 结构，并且设置对应的 overflow_handler，闭环啦！

```c
perf_event_alloc(...){
  ...
  if (overflow_handler) {
		event->overflow_handler	= overflow_handler;
		event->overflow_handler_context = context;
	} else if (is_write_backward(event)){
		event->overflow_handler = perf_event_output_backward;
		event->overflow_handler_context = NULL;
	} else {
		event->overflow_handler = perf_event_output_forward;
		event->overflow_handler_context = NULL;
	}  
  ...
}
```



Hold on，事情还没有结束，我们现在已经自底向上捋了一下 perf PMU 事件的调用流程，还需自上而下来总览一下整个 perf event 的实现原理。



## 3. perf_event_open 系统调用



Todo!()

