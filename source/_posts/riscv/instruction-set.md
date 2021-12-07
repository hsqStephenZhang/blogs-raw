---
title: RISC-V -- instruction set
---



# RISC-V  -- instruction set 



## 1. 什么是 RISC-V 



**定义**： **RISC-V**（发音为“risk-five”）是一个基于[精简指令集](https://zh.wikipedia.org/wiki/精简指令集)（RISC）原则的[开源](https://zh.wikipedia.org/wiki/开源标准)[指令集架构](https://zh.wikipedia.org/wiki/指令集架構)（ISA），简易解释为[开源软件](https://zh.wikipedia.org/wiki/開源軟體)运动相对应的一种“开源硬件”。该项目2010年始于[加州大学柏克莱分校](https://zh.wikipedia.org/wiki/加州大學柏克萊分校)，但许多贡献者是该大学以外的志愿者和行业工作者。



作为除了 x86 和 Arm 以外的第三大指令集，RISC-V 正在逐渐受到巨头们的青睐，并纷纷布局 RISC-V 生态，中科院计算所领导的团队也在大力开展 RISC-V 相关的工作。



RISC-V 有什么有点呢？一个字：**简单！**



RISC-V 指令使用模块化的设计，没有历史包袱，优雅简洁，十分便于学习，其 包括几个可以互相替换的基本指令集，以及额外可以选择的扩展指令集，大致包含了：

1. 基础整数指令集
2. 乘除指令
3. 浮点数指令
4. 原子指令
5. 压缩指令
6. 向量指令
7. 特权级指令



## 2. RISC-V 设计



### 2.1 寄存器集



RISC-V 包含了 32 个整数寄存器，以及一个用户可见程序计数器 PC。实现浮点数规范之后，还有 32 个浮点数寄存器（对于嵌入式系统，整数寄存器的数量可以为 16）。



和 MIPS 这种 RISC 架构类似，RISC-V 的第一个寄存器为 "**零寄存器**"（永远为 0，这可以带来很多好处，比如，`mov X, Y` 可以修改为 `add X, Y, 0 `）RISC-V 还提供了 "控制寄存器" 和 "状态寄存器" ，但是 User 模式无法访问。

RISC-V 没有指令可以同时操作多个寄存器，和 X86 这种功能强大但复杂的架构完全不同。



### 基本指令格式



基本指令包含下面几个部分：R I S U 

1. imm：立即数
2. rs1：源寄存器
3. rs2：源寄存器2
4. rd：目标寄存器
5. opcode：操作码





### 2.2 立即数





RISC-V 的立即数需要根据不同指令拼装（这一点对于手写 decoder 来说挺麻烦的）



### 整数计算









### 2.3 函数调用，分支，跳转



RISC-V 的函数通过 JAL（jump and link）将返回地址放在一个寄存器中，这样就可以省下一次堆栈的访问（push rip + jmp xxx） 



### 2.4 原子指令



RISC-V 支持多个 CPU 与 thread。其允许 CPU 执行时重排序：读取和写入顺序可以重排。但是有些读取可以被设置成“获取”运算，会在其后的访问之前被执行，有些写入可以被当作“释放”运算。基本指令中通过 `fence` 来保证存储器的访问顺序，





### 系统指令



RISC-V 规定了 8 条系统指令，简单的实现中，可以不定义这些软件，但是，如果是写一个 RISC-V 的系统模拟器，是一定要完成的。

scall 用于系统调用，sbreak 用于调试器调试，剩下的指令用于操作 csr（control and status registers）



| 指令                     | 含义                                        |
| ------------------------ | ------------------------------------------- |
| scall                    | system call                                 |
| sbreak                   | Breakpoint                                  |
| csrrw rd, csr, rs1       | read csr into rd, store rs1 into csr        |
| csrrc rd, csr, rs1       | read csr into rd, set bit of csr            |
| csrrs rd, csr, rs1       | read csr into rd, clear bit of csr          |
| csrrwi rd, csr, imm[4:0] | read csr into rd, store immediate into csr  |
| csrrci rd, csr, imm[4:0] | read csr into rd, set bit of csr with imm   |
| csrrsi rd, csr, imm[4:0] | read csr into rd, clear bit of csr with imm |



在大多数的系统当中，csr 只有在特权模式下才能访问，但是也存在一些只读寄存器，可以在用户模式下直接访问，比如：

1. cycle：从任意参照的时间流逝的时钟周期数
2. time：流逝的系统时间
3. instret：实时时钟
4. cycleh
5. timeh
6. instreth



## 3. 参考文件



[risc-v manual](http://crva.ict.ac.cn/documents/RISC-V-Reader-Chinese-v2p1.pdf.11.3)

[risc-v compression set manual](https://riscv.org/wp-content/uploads/2015/11/riscv-compressed-spec-v1.9.pdf)
