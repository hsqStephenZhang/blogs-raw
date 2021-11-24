---
title: ftrace
---



# ftrace



先说一个非常重要的参考资料，也就是 ftrace 作者本人的 pdf：[ftrace kernel hook](https://blog.linuxplumbersconf.org/2014/ocw/system/presentations/1773/original/ftrace-kernel-hooks-2014.pdf)

接下来，主要跟着作者的思路，融入自己的一些思考，给大家解读一下 ftrace 究竟做了什么？



## 1. 什么是 ftrace 



ftrace，顾名思义，就是 `function trace`，针对函数级别的追踪调试机制。

我们平时是如何使用这个强大功能的呢？



可以通过 tracefs 和内核进行交互，比如向 current_tracer 文件中写入 `function`，内核就会追踪所有的函数调用

```bash
$ cd /sys/kernel/debug/tracing
$ echo function > current_tracer
$ cat trace
# tracer: function
#
# entries-in-buffer/entries-written: 205022/119956607 #P:4
#
# _-----=> irqs-off
# / _----=> need-resched
# | / _---=> hardirq/softirq
# || / _--=> preempt-depth
# ||| / delay
# TASK-PID CPU# |||| TIMESTAMP FUNCTION
# | | | |||| | |
 <idle>-0 [002] dN.1 1781.978299: rcu_eqs_exit <-rcu_idle_exit
 <idle>-0 [002] dN.1 1781.978300: rcu_eqs_exit_common <-rcu_eqs_exit
 <idle>-0 [002] .N.1 1781.978301: arch_cpu_idle_exit <-cpu_startup_entry
 <idle>-0 [002] .N.1 1781.978301: tick_nohz_idle_exit <-cpu_startup_entry
 <idle>-0 [002] dN.1 1781.978301: ktime_get <-tick_nohz_idle_exit
 <idle>-0 [002] dN.1 1781.978302: update_ts_time_stats <-tick_nohz_idle_exit
 <idle>-0 [002] dN.1 1781.978302: nr_iowait_cpu <-update_ts_time_stats
 <idle>-0 [002] dN.1 1781.978303: tick_do_update_jiffies64 <-tick_nohz_idle_exit
 <idle>-0 [002] dN.1 1781.978303: update_cpu_load_nohz <-tick_nohz_idle_exit
 <idle>-0 [002] dN.1 1781.978303: calc_load_exit_idle <-tick_nohz_idle_exit 
```



向 current_tracer 文件中写入 `function_graph`，内核就会追踪所有的函数调用，并且将其调用链，函数执行时间都展示出来（这个功能在阅读内核代码的时候极其好用）

```bash
$ cd /sys/kernel/debug/tracing
$ echo function_graph > current_tracer
$ cat trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 2)   0.666 us    |                        load_new_mm_cr3();
 2)   0.170 us    |                        switch_ldt();
 2)   1.500 us    |                      } /* switch_mm_irqs_off */
 2)               |                      switch_mm_irqs_off() {
 2)   0.273 us    |                        load_new_mm_cr3();
 2)   0.157 us    |                        switch_ldt();
 2)   1.071 us    |                      }
 2)   0.166 us    |                      flush_tlb_mm_range();
 2)   0.152 us    |                      _raw_spin_unlock();
 2)   4.541 us    |                    } /* __text_poke */
 2)               |                    __text_poke() {
 2)               |                      __get_locked_pte() {
 2)   0.155 us    |                        _raw_spin_lock();
 2)   0.471 us    |                      }
 2)               |                      switch_mm_irqs_off() {
 2)   0.380 us    |                        load_new_mm_cr3();
 2)   0.169 us    |                        switch_ldt();
 2)   1.155 us    |                      }
 2)               |                      switch_mm_irqs_off() {
 2)   0.277 us    |                        load_new_mm_cr3();
```



如果通过 `echo '*schedule*' > set_ftrace_filter`  来设置过滤器，就会发现， `cat trace` 得到的结果中，果真就是我们想要留下来的满足 `*schedule*` 这个 pattern 的函数。



那么问题来了，ftrace 究竟是如何生效的？我们真的可以为所欲为吗？接下来深入探究一下。

 

## 2. ftrace 如何工作？



最根本的来说，ftrace 的实现需要基于编译器的一个功能： mcount。需要编译器在每个函数执行开始前插入一个函数，通过这种方式来 trace 每个函数的执行。



### 2.1 编译器的超能力 



要验证这一点也很容易，只需要通过 gdb 反汇编内核中的函数，就会发现，的的确确在这些函数的开头，都插入了一条 `call __fentry__` 指令！

```bash

$ gdb -batch -ex 'file vmlinux' -ex 'disassemble schedule'  // disassemble 别的函数也是这种结果

Dump of assembler code for function schedule:
   0xffffffff81829730 <+0>:     callq  0xffffffff81062c80 <__fentry__>
   0xffffffff81829735 <+5>:     push   %rbx
   0xffffffff81829736 <+6>:     mov    %gs:0x17bc0,%rbx
   0xffffffff8182973f <+15>:    mov    0x18(%rbx),%eax
   0xffffffff81829742 <+18>:    test   %eax,%eax
   0xffffffff81829744 <+20>:    je     0xffffffff81829763 <schedule+51>
   0xffffffff81829746 <+22>:    mov    0x2c(%rbx),%eax
   0xffffffff81829749 <+25>:    test   $0x30,%al
   0xffffffff8182974b <+27>:    je     0xffffffff81829759 <schedule+41>
   0xffffffff8182974d <+29>:    mov    %rbx,%rdi
   0xffffffff81829750 <+32>:    test   $0x20,%al
   0xffffffff81829752 <+34>:    je     0xffffffff818297b6 <schedule+134>
   0xffffffff81829754 <+36>:    callq  0xffffffff810a78a0 <wq_worker_sleeping>
   0xffffffff81829759 <+41>:    cmpq   $0x0,0x898(%rbx)
   0xffffffff81829761 <+49>:    je     0xffffffff81829790 <schedule+96>
   0xffffffff81829763 <+51>:    xor    %edi,%edi
   0xffffffff81829765 <+53>:    callq  0xffffffff81828e60 <__schedule>
   0xffffffff8182976a <+58>:    mov    %gs:0x17bc0,%rax
   0xffffffff81829773 <+67>:    mov    (%rax),%rax
   0xffffffff81829776 <+70>:    test   $0x8,%al
   0xffffffff81829778 <+72>:    jne    0xffffffff81829763 <schedule+51>
   0xffffffff8182977a <+74>:    mov    0x2c(%rbx),%eax
   0xffffffff8182977d <+77>:    test   $0x30,%al
   0xffffffff8182977f <+79>:    je     0xffffffff8182978e <schedule+94>
   0xffffffff81829781 <+81>:    mov    %rbx,%rdi
   0xffffffff81829784 <+84>:    test   $0x20,%al
   0xffffffff81829786 <+86>:    je     0xffffffff818297b0 <schedule+128>
   0xffffffff81829788 <+88>:    pop    %rbx
   0xffffffff81829789 <+89>:    jmpq   0xffffffff810a7870 <wq_worker_running>
   0xffffffff8182978e <+94>:    pop    %rbx
   0xffffffff8182978f <+95>:    retq   
   0xffffffff81829790 <+96>:    mov    0x8b0(%rbx),%rdi
   0xffffffff81829797 <+103>:   test   %rdi,%rdi
   0xffffffff8182979a <+106>:   je     0xffffffff81829763 <schedule+51>
   0xffffffff8182979c <+108>:   mov    (%rdi),%rax
   0xffffffff8182979f <+111>:   cmp    %rax,%rdi
   0xffffffff818297a2 <+114>:   je     0xffffffff818297bd <schedule+141>
   0xffffffff818297a4 <+116>:   mov    $0x1,%esi
   0xffffffff818297a9 <+121>:   callq  0xffffffff813c32c0 <blk_flush_plug_list>
   0xffffffff818297ae <+126>:   jmp    0xffffffff81829763 <schedule+51>
   0xffffffff818297b0 <+128>:   pop    %rbx
   0xffffffff818297b1 <+129>:   jmpq   0xffffffff81340280 <io_wq_worker_running>
   0xffffffff818297b6 <+134>:   callq  0xffffffff813402c0 <io_wq_worker_sleeping>
   0xffffffff818297bb <+139>:   jmp    0xffffffff81829759 <schedule+41>
   0xffffffff818297bd <+141>:   mov    0x10(%rdi),%rdx
   0xffffffff818297c1 <+145>:   lea    0x10(%rdi),%rax
   0xffffffff818297c5 <+149>:   cmp    %rax,%rdx
   0xffffffff818297c8 <+152>:   jne    0xffffffff818297a4 <schedule+116>
   0xffffffff818297ca <+154>:   jmp    0xffffffff81829763 <schedule+51>
End of assembler dump.
```



也就是说，在调用 schedule 函数之前，真正执行函数本身代码前会调用 `__fentry__` 函数。`__fentry__` 函数的定义是:

```c
#ifdef CONFIG_DYNAMIC_FTRACE

SYM_FUNC_START(__fentry__)
	retq
SYM_FUNC_END(__fentry__)
EXPORT_SYMBOL(__fentry__)
```



这个函数没有什么特别之处，直接 `retq` 返回。但正因为有了这个额外的函数调用，让我们能完成很多相当 hack 的事情。



```
Tips:
		如何让编译器加上这个 __fentry__ 的函数调用呢？这其实是因为 gcc 支持一个特殊的选项 `-pg -mrecord-mcount -mfentry`
		查看内核的 MakeFile，就会看到其编译的时候正是用到了这几个参数，这样就会插入函数探针:
		
    $ cat Makefile| rg fentry            // 查看源码中的 MakeFile
      ifeq ($(call cc-option-yn, -mfentry),y)
        CC_FLAGS_FTRACE     += -mfentry

```



### 2.2 then ?



Ok，那我们只需要针对这个 `__fentry__` 函数做一些 hack 操作，就大功告成？非也非也，还早着呢？

如果每一个函数开头，都需要进行这个额外的操作，开销累积起来是相当可观的。据统计显示，仅仅在 `__fentry__` 中使用 retq，就会增加 13% 的开销，显然无法接收！

因此内核开发者想要某种方式，零成本完成这件事。也就有下面几个目标：

1. 在启动内核的时候，将所有的 `call __fentry__` 转换成 `nop` 指令；
2. 需要确定所有 `__fentry__` 的位置，最好能在编译期间完成这件事



```
Tips:
		所谓的零成本，其实在 rust 和 cpp 中都有展现，其含义是：如果你不需要这个功能，那么你不需要为其发出代价，如果你使用了它，那么你不可能比语言本身提供的机制做得更好！
```



### 2.3 recordmcount



hold on! 到底说的是 mcount 还是 fentry ？这一点其实对于我们而言并不是很重要，但还是很有必要说明一下其中的缘由：

你几乎可以将 mcount 和 fentry 理解成一样的机制，mcount是最原先的版本，fentry是改进版。它们的区别在于，mcount 无法记录函数参数，反映到函数上就是，每次调用 mcount 之前，都需要将寄存器中的内容保存到堆栈上，如下所示

```asm
<posix_cpu_timer_set>:
     55							 push %rbp
     48 89 e5				 mov %rsp,%rbp
     41 57					 push %r15
     41 56 					 push %r14
     41 55 					 push %r13
     41 54 					 push %r12
     53 						 push %rbx
     48 83 ec 30 		 sub $0x30,%rsp
     e8 1a 81 0b 00  callq ffffffff810f7430 <mcount>
     48 8b 47 70 		 mov 0x70(%rdi),%rax
     49 89 ff 			 mov %rdi,%r15 
```

fentry 则不需要这个操作，直接调用 fentry 就好了。这样对比下来，如果要像之前那样动态修改 mcount 为 nop，这些寄存器压栈的开销仍不可避免，fentry 则由编译器为我们保证了这一点。因此，在 gcc 4.6 引入了 fentry 之后，就将 mcount 升级为 fentry 了，只需要用到 `gcc -pg -mfentry` 的编译选项即可。

```asm
<posix_cpu_timer_set>:
     e8 eb 3c 0b 00  callq ffffffff810f0af0 <__fentry__>
     41 57 					 push %r15
     41 56					 push %r14
     49 89 ff 			 mov %rdi,%r15
     41 55 					 push %r13
     41 54					 push %r12
     49 89 d5 			 mov %rdx,%r13
     55 						 push %rbp
     53 						 push %rbx
```



接下来鉴于









