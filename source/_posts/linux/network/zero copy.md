---
title: zero copy
---

## 1. 传统 IO

为了更好的理解零拷贝解决的问题，我们首先了解一下传统 I/O 方式存在的问题：

在 Linux 系统中，传统的访问方式是通过 write() 和 read() 两个系统调用实现的，通过 read() 函数读取文件到到缓存区中，然后通过 write() 方法把缓存中的数据输出到网络端口：

```c
read(file_fd, tmp_buf, len);
write(socket_fd, tmp_buf, len);
```

整个过程中，涉及到了 2 次系统调用，2 次内核态和用户态之间的数据拷贝，2 次内核态和 DMA 之间的数据拷贝，代价非常大， 有没有什么方式可以尽量减少这些开销呢？

有下面几个思路：

1. 减少系统调用。也就是说，我尽可能一次性将要完成的 IO 操作，告诉内核，这样就不需要频繁地进行上下文切换
2. 直接从磁盘拷贝文件到 DMA。如果我们可以将所要进行的 IO 操作描述清楚，比如需要将某个文件通过某个 socket 发送出去，就没有必要先将文件读取到用户空间，再拷贝到内核中，来完成剩下的操作；相反，我们可以直接在内核态读取文件内容，并且拷贝到 DMA 中。

简要概括：Context switch is expensive, copy is expensive,  copy directly in kernel !

## 2. 零拷贝

### 2.1 direct IO

### 2.2 mmap + write

### 2.3 sendfile

### 2.4 splice

### 2.5 sockmap
