---
title: notify chain
categories: [linux, trace]
date: 2022-1-9 13:47:40
---

Linux 中广泛采用了事件通知机制，即 **注册 + 回调**，`notify chain` 就是很典型的例子。

## 1. notify chain 是什么

首先要明白，notify chain 是什么？

将其比作 **发布-订阅模式**，可能更加容易理解：

某个读者订阅了某个杂志，当杂志发版之后，就会收到对应的通知；一旦不再想看这个杂志，取消订阅即可。

Linux 中为了及时相应某一些到来的事件，采取了通知链机制，可以区分为三个角色：

1. 订阅者。可以使用通知者提供的 API，注册感兴趣的事件，以及对应的回调函数
2. 通知者。通常是某个中断处理程序，或者是某个关键函数；
3. 事件。当某个事件发生时，通知者这段代码得到执行，就可以查找是否有订阅者对当前事件感兴趣，如果已经注册，那么调用对应的回调函数。

## 2. show me the code

我们可以从一个具体的地方开始看起：do_int3

熟悉 debugger 原理的同学会知道，单步中断用到了 `int 3` 这个特殊的指令，该指令会触发 CPU 异常，CPU 查找异常处理函数向量表之后，就会调用到之前注册的  `do_int3` 函数上。

`do_int3`  这个函数可以说完成了相当多的功能，比如 kprobe，ftrace，都在其中加入了自己的处理逻辑，我们这里暂时不去理会。

关注函数返回之前的  **notify_die** 函数调用，这个名字似曾相识，没错，马上就会进入通知链处理了！

```c
static bool do_int3(struct pt_regs *regs)
{
 int res;

#ifdef CONFIG_KGDB_LOW_LEVEL_TRAP
 if (kgdb_ll_trap(DIE_INT3, "int3", regs, 0, X86_TRAP_BP,
    SIGTRAP) == NOTIFY_STOP)
  return true;
#endif /* CONFIG_KGDB_LOW_LEVEL_TRAP */

#ifdef CONFIG_KPROBES
 if (kprobe_int3_handler(regs))
  return true;
#endif
 res = notify_die(DIE_INT3, "int3", regs, 0, X86_TRAP_BP, SIGTRAP);

 return res == NOTIFY_STOP;
}
```

noitfy_die 函数中，会将 `寄存器 + 异常名称 + 错误码 + 错误信号` 封装到 die_args 结构体中 ，并且调用 `atomic_notifier_call_chain` 函数，传入该结构体

```c
int notrace notify_die(enum die_val val, const char *str,
        struct pt_regs *regs, long err, int trap, int sig)
{
 struct die_args args = {
  .regs = regs,
  .str = str,
  .err = err,
  .trapnr = trap,
  .signr = sig,
 };
 RCU_LOCKDEP_WARN(!rcu_is_watching(),
      "notify_die called but RCU thinks we're quiescent");
 return atomic_notifier_call_chain(&die_chain, val, &args);
}
NOKPROBE_SYMBOL(notify_die);
```

`atomic_notifier_call_chain` 其实只是针对  `notifier_call_chain` 的一层封装，目的主要是为了 RCU lock 的加锁和解锁，核心处理逻辑在下面这段代码中，主要是遍历 notifiter_block 这个单向链表，分别调用这个注册链上的所有回调函数。

```c
static int notifier_call_chain(struct notifier_block **nl,
          unsigned long val, void *v,
          int nr_to_call, int *nr_calls)
{
 int ret = NOTIFY_DONE; 
 struct notifier_block *nb, *next_nb;

 nb = rcu_dereference_raw(*nl);

 while (nb && nr_to_call) {
  next_nb = rcu_dereference_raw(nb->next);

  ret = nb->notifier_call(nb, val, v);

  if (nr_calls)
   (*nr_calls)++;

  if (ret & NOTIFY_STOP_MASK)
   break;
  nb = next_nb;
  nr_to_call--;
 }
 return ret;
}
NOKPROBE_SYMBOL(notifier_call_chain);

```

这里的 `notifier_block` 这个链表中的内容，也不是凭空而来，一定需要通过某种机制向该链表添加结点，或者用更加专业一点的词汇：register

上面的 atomic notifier chain，也对应了这样一个 注册函数 `atomic_notifier_chain_register`，本质上是调用 `notifier_chain_register` 这个核心逻辑，注册到通知链中，这种 wrapper function，大部分情况是通过 spinlock 或者 rcu 机制来确保并发安全。

```c
int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
  struct notifier_block *n)
{
 unsigned long flags;
 int ret;

 spin_lock_irqsave(&nh->lock, flags);
 ret = notifier_chain_register(&nh->head, n);
 spin_unlock_irqrestore(&nh->lock, flags);
 return ret;
}
EXPORT_SYMBOL_GPL(atomic_notifier_chain_register);
```

`notifier_chain_register` 这个核心逻辑中，关键是根据 notifier_block 的优先级，将其插入到链表的合适位置中去，逻辑很清晰。

```c
static int notifier_chain_register(struct notifier_block **nl,
  struct notifier_block *n)
{
 while ((*nl) != NULL) {
  if (unlikely((*nl) == n)) {
   WARN(1, "double register detected");
   return 0;
  }
  if (n->priority > (*nl)->priority)
   break;
  nl = &((*nl)->next);
 }
 n->next = *nl;
 rcu_assign_pointer(*nl, n);
 return 0;
}
```

## 3. what's more?

上面拿 `int 3` 异常中的 notify_die 机制来作为示例，演示了一下 notify chain 是如何起作用的，但其实除了这个 `atomic_notifier_chain`，还有 `blocking_notifier_chain`，`raw_notifier_chain`，`srcu_notifier_chain`，本质上都是相通的。
