---
title: kernel && nic && napic 
categories: [linux, network, napi, nic]
date: 2022-2-23 16:19:45
---

## 1. kernel 与 nic 交互

接收数据帧的过程中，kernel 与网卡有两种交互方式：

1. 中断（interrupt）
2. 轮询（polling）

### 1.1 中断方式

如果要保证低延迟（laytency）则可以考虑中断的方式，保证一有数据就通过硬件中断通知 CPU，然后中断处理函数读取数据包，存放到上层的输入队列，再通知内核

### 1.2 轮询方式

采用 polling 的方式，定期检查网络设备中是否有数据，如果有数据，就连续将数据读出，从而提高吞吐量

### 1.3 中断轮询结合

既然这两者都有一定的局限，为何不将它们结合起来？这就是 NAPI（new api）的工作方式了。

NIC 接收到数据之后，通过中断通知 kernel，但是硬件中断处理函数并不分配 skb，也不读取数据包，而是将相应设备（napi_struct）添加到 poll 队列中，取消该网卡的硬件中断，触发下半部的软中断，由下半部的异步方式一次性读取完成设备上的所有数据或者达到配额上限，该步骤完成之后，重新启用网卡中断。

### 1.4 中断上半部 VS 中断下半部

大家知道，（硬件）中断是为了快速相应外部设备的一些事件，Linux 为了简化中断逻辑，减少竞争，规定中断不能被抢占（non-preemptive），也是不可重入的（nonreentrant），这就要求中断处理函数应该尽快完成。

那么一些比较繁重的任务（比如数据包的协议栈处理）如何完成呢？答案是下半部分。

中断被划分为上半部和下半部，上半部在中断上下文中快速执行，下半部则是以异步的方式完成特定的工作。Linux 提供的下半部解决方案有：

1. 软中断
2. tasklet
3. workqueue
4. kthread

前两者不依赖进程环境，可用于执行“不可休眠的任务”；后两个依赖于进程环境，执行期间可以休眠。

### 1.5 softirq internal

#### 1.5.1 什么是软中断

其实软中断这个名称有一点误导性，因为它和中断其实没有太大联系。只是采用了向量（数组）的方式来存放针对不同下标的处理函数，与硬件的处理比较类似，因而得名。

通过源码也能发现，结构非常简单，通过大小为 NR_SOFTIRQS(系统共计软中断数量)的数组来存放处理函数，注册软中断只需要设置对应项的 `softirq_action.action` 即可

```c
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};

static struct softirq_action softirq_vec[NR_SOFTIRQS];

void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```

网络子系统的初始化函数 `net_dev_init` 中注册了两个软中断处理函数 `net_tx_action` `net_rx_action`，一个对应发包（tx），一个对应收包（rx）。

```c
static int __init net_dev_init(void)
{
    //...
    open_softirq(NET_TX_SOFTIRQ, net_tx_action);
    open_softirq(NET_RX_SOFTIRQ, net_rx_action);
    //...
}
```

#### 1.5.2 软中断做了什么

软中断由 `ksoftirqd` 这个内核线程来处理，对应的入口函数为 `run_ksoftirqd`，系统初始化的时候，会在每一个核心上都启动一个该线程(kernel-softirqd) 

```c
static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
	cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
				  takeover_tasklets);
	BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

	return 0;
}
early_initcall(spawn_ksoftirqd);
```

入口函数 `run_ksoftirqd` 实际上只是对于 `__do_softirq` 的一层封装，简化之后对应下面的逻辑

```c
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
	// ...

	pending = local_softirq_pending();

restart:
	/* Reset the pending bitmask before enabling irqs */
	set_softirq_pending(0);

	local_irq_enable();

	h = softirq_vec;

	while ((softirq_bit = ffs(pending))) {

		h += softirq_bit - 1;
		h->action(h);
		h++;
		pending >>= softirq_bit;
	}

	//...
}
```

软中断通过 pending(u16) 来表示待处理的信号，借助 pending 的每一个 bit 来开启或关闭软中断，通过 `ffs(pending)` 可以获取等待处理的软中断信号。因此，在 while 循环中，就可以通过 bit 下标，获取到相应中断处理函数之后直接回调。对于接收数据包而言，就来到了 `net_rx_action` 函数中。

## 2. 非 NAPI VS NAPI

NAPI 虽然有很多优势，但对于硬件有一定的要求：

1. 设备需要有足够的缓冲区，从而保存多个数据包
2. 需要可以禁用中断，而不影响其它的操作

因此，仍然有一些旧设备不支持 NAPI，但是 NAPI 框架使用了一些 trick，很好地兼容了旧逻辑，因此这里不妨先介绍一下 NAPI 工作模式。

### 2.1 NAPI 工作方式

NAPI 作用于驱动上，流程如下：
1. 网卡收到数据包，关闭其对应的中断，并且将该网络设备添加到 poll_list 中
2. 在软中断循环中，检测到需要进行 net_rx_action，并针对 poll_list 的每一个网卡，都进行 poll
3. 每一个网络设备轮询时的限额或者时间用完了之后，重启启用网卡硬件中断

接下来详细说明

#### 2.1.1 数据结构

网络数据接收的过程中，一个非常重要的数据结构是 per-cpu 变量 `softnet_data`。

```c
// net/dev/core.c
DEFINE_PER_CPU_ALIGNED(struct softnet_data, softnet_data); // per cpu
EXPORT_PER_CPU_SYMBOL(softnet_data);

// include/linux/netdevice.h
struct softnet_data {
	struct list_head	poll_list;
	struct sk_buff_head	process_queue;

	/* stats */
	unsigned int		processed;
	unsigned int		time_squeeze;
	unsigned int		received_rps;

	unsigned int		dropped;
	struct sk_buff_head	input_pkt_queue;
	struct napi_struct	backlog;

	// other stuff, need to be configure in kbuild
};
```

关键字段含义：

- poll_list：需要轮询的设备列表，对应 napi_struct
- process_queue/input_pkt_queue：用于 Non-NAPI 设备，通过 backlog 队列进行处理
- processed/time_squeeze/received_rps：统计信息

其中 Non-NAPI 设备的字段留在后面详解，这里主要关注 poll_list 这个通用字段。

poll_list 可以说是连接硬件中断和软中断的**桥梁**，它是一个链表，每当有一个网络设备需要轮询时，就添加到 `softnet_data.poll_list` 中，从而在 net_rx_action 当中异步轮询。

poll_list 保存的是 napi_struct 结构，第一眼就知道它绝对是 NAPI 的 C位

```c
struct napi_struct {
    struct list_head    poll_list;

    unsigned long       state;
    int         weight;
    int         (*poll)(struct napi_struct *, int);
    // GRO, netpoll，timer
    struct net_device   *dev; 
    struct sk_buff      *skb; 
    struct list_head    dev_list;  
    struct hlist_node   napi_hash_node;
    unsigned int        napi_id;   
};
```

重要字段含义：

- weight：轮询的限额
- poll：轮询回调函数

两个概念要提一下：预算（budget）和权重（weight）。预算代表了所有要轮询的设备在一次软件中断中总共能够处理的帧的输入；限制预算的原因是防止软中断占用CPU时间过长，从而影响其他进程和软中断（硬件中断不受影响）。现在除了总预算外，为了公平起见，对每个设备也需要分配一定的权重（weight），限制每个设备最大的帧读取量，weidght 越大说明该设备性能越高。

poll 函数是 NAPI 的核心，当设备与驱动匹配的时候，会调用驱动的 probe 函数，此时就可以通过 `netif_napi_add` 注册相应的 NAPI 设备，这样该网络设备接收到数据包并产生中断以后，就可以获取到对应的 napi_struct 并间接传递给 net_rx_action。

```c
void netif_napi_add(struct net_device *dev, struct napi_struct *napi,
		    int (*poll)(struct napi_struct *, int), int weight)
{
	if (WARN_ON(test_and_set_bit(NAPI_STATE_LISTED, &napi->state)))
		return;

	INIT_LIST_HEAD(&napi->poll_list);
	INIT_HLIST_NODE(&napi->napi_hash_node);
	hrtimer_init(&napi->timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_PINNED);
	napi->timer.function = napi_watchdog;
	init_gro_hash(napi);
	napi->skb = NULL;
	INIT_LIST_HEAD(&napi->rx_list);
	napi->rx_count = 0;
	napi->poll = poll; // IMPORTANT!
	
	// ... 
	set_bit(NAPI_STATE_SCHED, &napi->state);
	set_bit(NAPI_STATE_NPSVC, &napi->state);
	list_add_rcu(&napi->dev_list, &dev->napi_list);
	napi_hash_add(napi);
}
EXPORT_SYMBOL(netif_napi_add);
```

#### 2.1.2 从硬件中断到软中断

前面提到，硬件中断只需要做好一件事就好了：将网络设备的 napi_struct 添加到 `softnet_data.poll_list` 中，之后就交给软中断处理了。

收包的软中断处理函数为 `net_rx_action`

```c
static __latent_entropy void net_rx_action(struct softirq_action *h)
{
	struct softnet_data *sd = this_cpu_ptr(&softnet_data);
	unsigned long time_limit = jiffies +
		usecs_to_jiffies(netdev_budget_usecs);
	int budget = netdev_budget; // budget = 300
	LIST_HEAD(list);    // 待轮询的所有设备
	LIST_HEAD(repoll);  // 因自身权重用完而需要下一次轮询的设备

	local_irq_disable();
	list_splice_init(&sd->poll_list, &list);
	local_irq_enable();

    // 遍历每个设备的napi，直到所有设备都处理完，或者预算用完
	for (;;) {
		struct napi_struct *n;

		if (list_empty(&list)) {
			if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
				goto out;
			break;
		}

		// 取出链表中第一个待处理的设备
		n = list_first_entry(&list, struct napi_struct, poll_list);
		budget -= napi_poll(n, &repoll);

		// 预算（时间 or 次数）用完了
		if (unlikely(budget <= 0 ||
			     time_after_eq(jiffies, time_limit))) {
			sd->time_squeeze++;
			break;
		}
	}

	local_irq_disable();

    // 以下设备要重新被放入sd->poll_list
    // 1. 因为预算用完，list 中尚未处理的设备，
    // 2. 因设备本身权重用完而被放入 repoll 的设备
	list_splice_tail_init(&sd->poll_list, &list);
	list_splice_tail(&repoll, &list);
	list_splice(&list, &sd->poll_list);

	// 不论是总预算还是设备 weight 的关系，本次执行未能完成所有设备及其所有数据的接收，重新调度 NET_RX_SOFTIRQ（设置 pending 标志位）
	if (!list_empty(&sd->poll_list))
		__raise_softirq_irqoff(NET_RX_SOFTIRQ);

	net_rps_action_and_irq_enable(sd);
out:
	__kfree_skb_flush();
}
```

```c
static int napi_poll(struct napi_struct *n, struct list_head *repoll)
{
	void *have;
	int work, weight;

	list_del_init(&n->poll_list);

	have = netpoll_poll_lock(n);

	weight = n->weight;

	// 调用 napi_struct 的 poll 函数
	work = 0;
	if (test_bit(NAPI_STATE_SCHED, &n->state)) {
		work = n->poll(n, weight);
		trace_napi_poll(n, work, weight);
	}

	if (unlikely(work > weight))
		pr_err_once("NAPI poll function %pS returned %d, exceeding its budget of %d.\n",
			    n->poll, work, weight);

	if (likely(work < weight))
		goto out_unlock;

	/* Drivers must not modify the NAPI state if they
	 * consume the entire weight.  In such cases this code
	 * still "owns" the NAPI instance and therefore can
	 * move the instance around on the list at-will.
	 */
	if (unlikely(napi_disable_pending(n))) {
		napi_complete(n);
		goto out_unlock;
	}

	if (n->gro_bitmask) {
		/* flush too old packets
		 * If HZ < 1000, flush all packets.
		 */
		napi_gro_flush(n, HZ >= 1000);
	}

	gro_normal_list(n);

	/* Some drivers may have called napi_schedule
	 * prior to exhausting their budget.
	 */
	if (unlikely(!list_empty(&n->poll_list))) {
		pr_warn_once("%s: Budget exhausted after napi rescheduled\n",
			     n->dev ? n->dev->name : "backlog");
		goto out_unlock;
	}

	list_add_tail(&n->poll_list, repoll);

out_unlock:
	netpoll_poll_unlock(have);

	return work;
}
```

### 2.1 非 NAPI 工作方式

了解了 NAPI 这种中断和轮询相结合的收包方式之后？再来回顾一下经典的 backlog 队列模式，就能领会到 Linux 向下兼容的艺术。

对于不支持 NAPI 工作方式的网卡，在网卡的中断处理函数中会直接调用 netif_rx，其在中断上下文中执行，执行期间会关闭 CPU 的中断，工作完成再重新打开。
netif_rx 函数的工作最终主要由 enqueue_to_backlog 完成。

```text
hw interrupt handler
    --> netif_rx
        --> netif_rx_internal
            --> enqueue_to_backlog
				--> ____napi_schedule
					--> raise net_rx_action (async)
```

```c
static int enqueue_to_backlog(struct sk_buff *skb, int cpu,
			      unsigned int *qtail)
{
	struct softnet_data *sd;
	unsigned long flags;
	unsigned int qlen;

	sd = &per_cpu(softnet_data, cpu);

	local_irq_save(flags);

	rps_lock(sd);
	if (!netif_running(skb->dev))
		goto drop;
	qlen = skb_queue_len(&sd->input_pkt_queue);
	if (qlen <= netdev_max_backlog && !skb_flow_limit(skb, qlen)) {
		if (qlen) {
enqueue:
			__skb_queue_tail(&sd->input_pkt_queue, skb);
			input_queue_tail_incr_save(sd, qtail);
			rps_unlock(sd);
			local_irq_restore(flags);
			return NET_RX_SUCCESS;
		}

		/* Schedule NAPI for backlog device
		 * We can use non atomic operation since we own the queue lock
		 */
		if (!__test_and_set_bit(NAPI_STATE_SCHED, &sd->backlog.state)) {
			if (!rps_ipi_queued(sd))
				____napi_schedule(sd, &sd->backlog);
		}
		goto enqueue;
	}

drop:
	// handle exceptions
}
```

来看看 softnet_data 结构中与 non-napi 设备相关的字段，

- sd.input_pkt_queue
该队列仅用于 non-napi 设备。non-napi 设备会在驱动程序中（往往是中断处理函数）分配 skb，从寄存器读取数据到 skb，然后调用 netif_rx 把 skb 放到 sd->input_pkt_queue 中。注意，所有 non-napi 设备共享输入队列，即 per-cpu 的sd->input_pkt_queue。

- sd.process_queue
刚才已经提到，non-napi 设备把 skb 放入 sd.input_pkt_queue，然后在下半部处理中使用 sd.backlog.poll 即proces_backlog 函数来处理 skb。改函数模拟了 NAPI 驱动的 poll 函数行为。作为对比，NAPI 设备的 poll 从设备私有队列读取。Non-NAPI 的 process_backlog 函数则从 non-napi 设备共享的输入队列 input_pkt_queue 中读取配额的数据。然后放入 sd.process_queue。

- sd.backlog
用于 non-napi 设备，为了和 NAPI 接收架构兼容，所有的 non-napi 设备使用一个虚拟的 napi_struct，即这里的 sd.backlog。NAPI 设备把自己的 napi_struct 放入sd->poll_list，而所有的 non-napi 设备在 netif_rx 的时候把sd->backlog 放入 sd->poll_list。在中断下半部处理函数中，会通过 napi_poll 调用 backlog.poll 进行处理。
通过这种**共享的虚拟 napi_struct**方式，使得 non-napi 设备很好的融入了 NAPI 框架，使得 non-napi 和 NAPI 设备对下半部（net_rx_action）是透明的。不得不说，这是一个值得学习的精巧设计，让我们知道如何向前兼容旧的机制，如何让下层的变化对上层透明。

## 3. NAPI 示例



