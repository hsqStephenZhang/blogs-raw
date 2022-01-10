---
title: kprobe
categories: trace
date: 2022-1-9 13:47:40
---

## 1. kprobe 是什么

Kprobe是一种内核调测手段，它可以动态地跟踪内核的行为、收集debug信息和性能信息。 可以跟踪内核**几乎所有的代码地址**。这点非常神奇啊，几乎所有的代码，Wow，究竟如何办到这一点呢？我们可以从 debugger 角度出发，探究一下这个问题。

众所周知，debugger 是依赖于 ptrace 这个系统调用，ptrace 赋予了程序调试别的进程的能力，这一点是如何办到的呢？

不得不仔细观察 ptrace 的参数选项，我们发现，ptrace 有一个神奇的选项，叫做 `PTRACE_POKETEXT`，文档中说，这个选项可以 `Copy the word data to the address addr in the tracee's memory`，既然是修改内存中的内容，也就可以修改指令？

kprobe 是作用于指令级别的探针手段，因此和指令的执行分不开关系，必须要提前明确其中的细节：cpu取指令，执行指令，pc指向下一条指令地址，并且检查标志位（中断，eflags），判断是否会触发异常。而异常处理程序是由操作系统提前设置好的。



## 2. kprobe 实现原理

有的同学可能很诧异，what？代码段还能修改？不是只有可读可写的数据段之类的，才能修改其中的内容吗？Too young too simple

只要是内存中的东西，内核都有办法将其修改掉，只不过，这种手段并不会轻易暴露给用户，否则可不把系统玩崩了

### 2.1 register_kprobe

做了下面几件事：

1. 根据符号获取 kprobe 的地址，或者直接获取到传入参数中的 kprobe 地址
2. 准备一个可执行 mmap 区间中的 slot，并且将被 probe 的指令保存到 slot 中
3. 添加这个 kprobe 到全局的 kprobe 哈希表中
4. 通过 arm_probe 将对应位置上的指令的第一个字节替换为 0xcc

如此一来，执行到这个 kprobe 之后，就会触发 int3 trap，就会陷入内核设置的 int3 处理函数中，即 `do_int3`

### 2.2 kprobe_int3_handler

做了下面几件事：

1. 执行 pre_handler
2. 设置单步中断标志位
3. 将 pt_regs 中的 ip 指向保存的指令地址

```markdown
Tips:
1. 注意，此时已经将被修改指令的第一个字节修改为 0xcc(int3)，如果该指令的地址为 x,那么此时的 ip = x + 1
2. 因为单步中断是完全可控的，所以可以将 ip 指向我们保存在可执行段中的指令，之后再将 ip 重新指回去就好了
```

### 2.3 kprobe_debug_handler

重置该 kprobe，并且修正一些由于上一条指令执行导致的问题：

1. 函数调用

之所以要修正，主要是因为如果我们刚好将一条 `call %rip(xxx)` 执行的第一个字节修改为 `0xcc` 了，在单步执行之后，就会将返回地址压栈，压栈的地址又是通过当前的 EIP 来决定的，但此时的 EIP 已经不是正常执行流中的预期值了，这样，函数调用返回之后，return 直接爆炸

因此，这里有必要修正一下位于栈顶的 return_address

### 2.4 unregister_kprobe

做了几件事：

1. 弹出对应指令地址处，最上方的一个 kprobe 探针
2. 对于弹出的探针，将其禁止，也就是通过 `disarm_kprobe` 将原先指令还原
