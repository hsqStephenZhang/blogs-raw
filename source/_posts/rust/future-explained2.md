---
title: future explained(2)
date: 2021-11-24 14:01:29
categories: "rust future"
---



# Future explained(2)  -- self referential



前面提到了，future 其实分为 Non-Leaf-Future 和 Leaf-Future，前者由 async/await 生成，后者由开发者手动对接系统 Io。



那么为什么会出现 self referential 的问题呢？

其实主要的原因出现在 async/await 生成的 Non-Leaf-Future 上！



试想下面的这段代码



```rust
async fn test(){
   let mut a = [1024; u8];
   let f1 = read_to_string(&mut a).await;
}
```



如果将其翻译为状态机，大致会是下面这个结构：

```rust
enum TestFuture{
    Start(Start),
    State1(State1),
    Done,
}

struct Start;

struct State1<'self>{
    a: [1024; u8],
    ref_a: &'self mut [1024; u8],
}

struct Done;
```



由于需要用一个闭包来保存上下文中的变量，这里用到了 &mut a，跨越了 await 点，所以既需要将原先的这个数组 a 保存下来，也需要将其引用保存下来。



```
Tips:
		跨越 await 点的引用，既需要保存被引用的结构（owner），也需要保存该引用。这是因为，我们需要始终保证引用指向的内容是有效的！
```



可惜的是，上面这段代码只能是我们的一厢情愿！实际中，我们根本无法构造出 State1 这种结构！

这是因为 Rust 严格的所有权机制导致的，我们已经有了 TestFuture 结构的可变引用，就无法再将一个可变引用 ref_a 指向其自身的一个结构。那么 Rust 编译器是怎么做的呢？



嘿嘿，Rust 代码都是 Rustc 编译生成的，编译器自然有办法：所谓的引用，也不过是受到特殊约束的指针罢了，在确保安全的情况下，使用一个裸指针代替，效果完全相同！

也就是说，编译器实际上为我们生成了下面的结构：

```rust
struct State1{
    a: [1024; u8],
    ref_a: *const [1024; u8],
}

```



而该结构体，通过一个 new 函数是无法直接构造的，只有在确定了其**在栈上的位置**之后，才能通过 init 方法初始化其 ref_a 字段。

```rust
#[derive(Debug)]
struct State1 {
    a: [1024; u8],
    b: *const [1024; u8],
}

impl State1 {
    fn new() -> Self {
        State1 {
            a: [1024; u8],
            ref_a: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const [1024; u8] = &self.a;
        self.ref_a = self_ref;
    }
}
```



上面强调了一点：**在栈上的位置**。因为如果 ref_a 指向的是栈上的内存，一旦 State1 在栈上移动，比如拷贝到另一块区域，这个指针就会失效。相反，如果其分配在堆上，就不容易出现这个问题，因为只要这个堆内存没有释放，就不会被移动！



