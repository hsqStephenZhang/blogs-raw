---
title: future explained(3) 
date: 2021-11-24 14:01:29
categories: "rust future"
---



# future explained(3)  -- Pin 心路历程

withoutboats 是 Rust 异步的主要负责人，他在一次分享中详细描述了 Pin 的演变过程和背后的心路历程



## 一. Future 模式引入

在 Rust 之前的很多语言，绝大多数选择了回调机制实现的异步逻辑，比如C语言中的 libevent，再还有 nodejs 中的 callback。  
首先在 Rust 中，你很难写出 `nodejs` 那种回调模式的异步代码，下面是一小段典型的 `nodejs` 回调模型

```rust
fn get_foos(client: Client) -> impl Future<Output = Vec<Response>>{
    client.get_index("/foos").and_then(|index|{
        client.get_responses(&index).collect()
    })
}
```



在 Rust 中，且不说回调地狱问题，关键是这种写法 **根 本 编 译 不 过!**

这段代码，在 Rust 中存在很严重的所有权和生命周期问题，你会得到各种各样的编译错误。根据 rustc 的提示，缝缝补补，你或许会用下面这种方式去处理，也确实可以编译，但也仅此而已。且不说为了编译通过而付出的努力，单单是 Arc 和 Mutex 带来的额外开销，也是无法接受的。当创建大型程序时，简直无法想象。 

```rust
fn get_foos(client: Client) -> impl Future<Output = Vec<Response>>{
    let client: Arc<Mutex<Client>> = Arc::new(Mutex::new(Client));
    let client2 = client.clone();

    client.lock().unwrap().get_index("/foos").and_then(move |index|{
        client2.lock().unwrap().get_responses(&index).collect()
    })
}
```



因此，Rust 团队进行设计了一套基于 Future 的 API，提出了 async/await 这组关键字，将异步代码转换成同步模型，这不仅回避了上面的所有权问题，还更加符合人体工程学。 

这种写法非常接近 `blocking io`，程序员能轻易看出执行的逻辑，却能够以异步的方式运行，这主要归功于 Rust 编译器对于 async 代码块的高级抽象，async block 会被翻译成一个**状态机 GenFuture**（匿名结构体），并且其中任何跨越 await 代码块的引用，都会将被引用的变量和引用保存到当前为 async 代码块生成的状态机中，每一个 await 点都是状态的转变时机。  


这样的设计是十分优雅的，也非常有效，但是却带来了一个问题：**自引用** 。来看下面这个例子：  

```rust
async fn get_foos(client: Client) -> Vec<Response>{
    let index = client.get_index("/foos").await; // State { Client, &mut Client}
    client.get_responses(&index).collect().await
}
```



上面的代码中，`let index = client.get_index("/foos").await;` 存在一个跨越 await 关键字的对于 client 的引用，因此，必须要将 client 和 &mut client 都保存在状态机中。（这里不去考虑手动构造的问题，需要 unsafe 代码操作裸指针才可以，编译器实现起来会轻松很多）

如果按照最原始的 Future 接口( `fn poll(&mut self, ctx: &mut Context<'_>)`)，我们很轻易就能在安全代码中让 Rust 程序崩溃：

```rust
fn bar(){
    let mut future = get_foos(...);
    future.poll(...);
    let mut future2 = get_foos(...); // 为了防止 let mut future2 = future 只是一个别名，强制初始化 future2，从而下面的 future2 = future 成为一个 memcpy 操作
    future2 = future;       // 自引用结构已经失效，因为 future 和 future2 都是栈上的结构
    future2.poll(...); // 此时完全无法控制 poll 中是否会使用该失效的自引用 ，
                       // 产生了 undefined behavior
}
```

上面代码存在的问题是：当 poll 一个 Future 之后，我们移动了它原本的位置，导致 Future 中的**自引用结构失效**。 

这似乎有一点莫名其妙，深究一下根源，你会发现，这和堆、栈的特性有关。 

在栈上分配的变量，如果移动了所有权，其大概率（如果不是通过别名复用这块内存的话）也会重新分配一块内存，并且原先的内存会失效。参照下图，在 `state2 = state1;` 之后，会将 state1 中的内容完完整整拷贝到 state2 中，但是 State 中的 `&mut client`，还指向 state1 那块已经失效的内存，这才产生的 UB 行为

<div align=center>
<img height=70% width=70% src="/images/pin1.png" />
<br/>
<br/>
</div>
解决的办法很简单：堆内存分配！ 

如果我们在栈上只保留一个指针，指向堆上分配的 State，那么，当所有权发生转移的时候，仅仅是之前这个指针失效了，堆上的**值引用结构**并没有移动，也没有被释放，仍然是有效的。

<div align=center>
<img height=70% width=70% src="/images/pin2.png" />
<br/>
<br/>
</div>

这种模式，就好像将 State 钉(Pin)在了堆空间中，无论栈上的指针怎么移动，堆上的内存都不会受到半点影响。

```rust
let state3 = state2;
let state4 = state3; // totally fine
...

```

## 二. Future 改进，Pin 引入

withoutboats 作为 Rust 异步机制的主要设计者，针对原始的 Future trait 提出了几个方案：

### 1. unsafe 标记

这个方案解决的思路是：为了使用 Future，你必须要确保 SAFETY：该 Future 一旦被 poll 之后，直到其被 Drop 都不会被移动。  
对于分配在堆上的 future 天生就满足这一点要求，所以这主要是针对在栈上的内存而言的约束。 

虽然这确实可以解决现阶段 Future 移动导致的问题，但这会带来一个不得不考虑的问题：本就不受大家欢迎的 unsafe 可能会遍布代码库！而且相当于把锅都甩给了用户，大大加重了开发人员心智负担，该方案作为针对 Future 接口的第一次改进，也就到此为止。

```rust
pub trait Future{
    type Output;

    unsafe fn poll(&mut self, ctx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

### 2. Pin 的原始版本

unsafe Future Trait 的方案被 pass 之后，Rust 团队开始寻求别的解决方式，并提出了 Pin 的概念。 

但 Pin 最一开始并不像现在这样简洁，其也是经历了很多设计上的改进。最一开始的设计其实长下面这样：

```rust
pub struct PinMut<'a, T>(&'a mut T);

impl<'a, T> PinMut<'a, T> {
    pub unsafe fn get_mut_unchecked(self) -> &'a mut T{
        self.0
    }
}

pub struct PinBox<T>(Box<T>);

impl<T> From<Box<T>> for PinBox<T> {
    fn from(b: Box<T>) -> PinBox<T> {
        PinBox(b)
    }
}

impl<T> PinBox<T> {
    pub fn as_mut<'a>(&'a mut self) -> &'a mut T{
        PinMut(&mut *self.0)
    }
}

pub trait Future{
    type Output;

    fn poll(self: PinMut<'_, Self>, ctx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

这里引入了 `PinBox` 和 `PinMut` 两个概念，都是对于指针的封装，最大的不同是，如果想要通过 `PinMut` 获取到背后的指针，必须要通过非安全方法完成，而对于 `PinBox` 而言，则是可以直接在安全代码中获取 Box 背后的对象指针。  

这种封装，带来了一个很大的好处：如果想要通过 PinMut 获取背后的引用，只能通过 get_mut_unchecked 这种 unsafe 的方式，那么，当我们将 Future 的 self 类型限定为 `PinMut<'_, Self>` 的时候，就会发现，并不是那么轻易就能获取到 `&mut self` 的哦，必须要确保这个操作的 **SAFETY**，也就不可能在安全代码中，直接获取到 &mut self，继而导致 future 中内容被轻易移动。并且，基于这两个抽象，正确性证明起来是比较容易的。    

```rust
let a = PinMut(&mut p);
// error to compile
mem::swap(a.get_mut_unchecked(),&mut something_else);
// ok to compile, but usage of unsafe is wrong
mem::swap(unsafe { a.get_mut_unchecked() },&mut something_else);

```

这段话中还引入了一点，如果能够在安全代码中，获取到自引用结构的可变引用，也会导致 Rust 给我们做出的安全规范轰然倒塌
结合之前的堆、栈分配的特点，实际使用中，会先将其分配在堆上，保证不会因为 `let mut future2 = future1;` 这种语义将其移动，其次，就是遵守 PinMut 给我们的约束，保证不会获取到可变引用而移动。 

相比如第一个 unsafe 提案，我们虽然免不了和 unsafe 打交道，但是已经可以得到一个非常干净的 Future 接口，调用 poll 也不需要再用 unsafe 来方式误用了，相当于是将 unsafe 逻辑转移到了内层实现中。

```rust
pub fn spawn<F: Future>(f: F){
    let mut fut = PinBox::from(Box::new(f));
    
    // safe the fut to somewhere

    // handin to the executor

    // construct PinMut, then poll
    fut.as_mut().poll(...);
} 
```

但即使是内层的 unsafe 逻辑，也会给开发人员带来很大的心智负担！ 

在上面的约束下，我们如果想要在 poll 方法中更新 self 的状态，或者调用 self 某一些 field 的方法，都要首先通过非安全代码，获取 &mut Self，相当于声明：我遵守 unsafe 赋予我的一切权力，我保证不会移动 self。

```rust
struct AsyncIoHandle{
    pub fn poll_read(&mut self,buf: &mut [u8]) ->Poll<usize>{
        ...
    }
}

struct IoFuture {
    buf: Vec<u8>,
    file: AsyncFileHandle
}

impl Future for IoFuture {
    tyep Output = usize;

    fn poll(self: PinMut<'_, Self>, x: &mut Context<'_>) -> Poll<Self::Output>{
        // wont't compile 
        // self.file.poll_read(&mut self.buf[..]) 

        unsafe {
            let this: &mut Self = self.get_mut_unchecked();
            this.file.poll(&mut this.buf[..]);
        }
    }
}
```

这种恼人的问题对于 **Leaf Future** 尤为严重。所谓的 **Leaf Future**，其实是面向底层 Io(reactor)、需要手动实现的 Future，**Non-Leaf Future** 就是通过 async await 自动生成的 Future，包含了下面的所有结点，non-leaf or leaf。更令人无奈的一点是，对于 99% 手动实现的 `Leaf Future`，并没有 `self referential` 的问题，仍需要使用 unsafe。  async/await 中的 `self referential` 算是解决，但又发力过猛，一棒子打死了更多的 **good future**。

总结一点：**目前为止，通过 PinMut 和 PinBox 的抽象，已经解决了 executor 中的问题(比如需要使用 unsafe)，提供了易用的 Future::poll 接口，但是对于 reactor 仍无太好的办法。PinMut 和 PinBox 的正确性是毋庸置疑的**  

### 3. Unpin 的引入

为了解决上面的问题，withoutboats 引入了 Unpin 的概念。  

Unpin 是一个 auto trait，Rust 默认为所有结构都添加了这个标记 Trait（像 Send 和 Sync），但是，如果一个结构包含了 !Unpin 的字段，那么其自身也是 !Unpin（ !Unpin 属性会传播），几乎唯一的 !Unpin 结构就是编译器为 async 代码块生成的 GenFuture。其余所有的基础结构，包括 i32, usize 都是 Unpin 的结构，比较特殊的是对于复杂的自引用结构的指针。  

对于内部包含了 Unpin 结构的 PinMut，只需要为其实现 DerefMut 方法，就可以直接获取到 &mut T，PinMut<'a,T> 和 &mut T，也就不需要烦人的 unsafe 了! 乌拉！   

```rust
pub auto trait Unpin {}

impl<'a, T: Unpin + ?Sized> DerefMut for PinMut<'a,T>{
    type Target = T;

    fn deref_mut(&mut Self) -> &mut Self::Target{
        ...
    }
}

```

3. 最终方案 Pin

最终引入标准库的方案并不是上面的 PinMut + PinBox，而是更加简洁的一个统一接口： **Pin**  

Pin 也是一个智能指针，内层包装了一个指针，针对内部不同的指针类型，实现了不同的 trait。 

标准库中有下面几个关键的函数，都对它们做了注解：  

```rust
pub struct Pin<P>{
    pointer: P
}

// 直接通过裸指针创建 Pin 是不安全的，使用者必须要确保，这个指针是有效的!
impl Deref<P: Deref> Pin<P>{
    pub unsafe fn new_unchecked(pointer: P) -> Pin<P> {
        Pin { pointer }
    }
}

// 同理，将 Pin 转换成内部的裸指针，也是不安全的!
impl<'a, P> Pin<'a mut P>{
    pub unsafe fn get_unchecked_mut(self) -> 'a mut P {
        self.pointer
    }
}

// Target::Unpin 的指针 P，可以安全获取 &mut P::Target
impl<P: DerefMut<Target: Unpin>> for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    } 
}

// 如果要从 Pin<P> 创建出 Pin<&'a mut P::Target>，还是可以办到的
// 因为 Pin 始终有这个指针的所有权，所以不需要担心会出现 UB 行为
impl<P: DereMut> Pin<P> {
    pub fn as_mut<'a>(&mut self) -> Pin<&'a mut P::Target> {
        unsafe { Pin::new_unchecked(&mut *self.pointer) }
    }
} 

// 从 Box 创建 Pin 本身就是安全的，因为 Box 实现了 Unpin
impl<P> From<Box<P>> for Pin<Box<P>> {
    fn from(b: Box<P>) -> Pin<Box<P>>{
         unsafe { Pin::new_unchecked(b) }
    }
}
```

这段代码可能有一点绕，但是核心关注点就两个

1. 对于实现了 Deref 类型的 P，是否可以安全地获取到内层的这个 pointer
2. 对于实现了 Deref 类型的 P，是否可以安全地获取内层 pointer 指针指向的对象

始终要牢记 Pin 的使命：需要确保 Rust 中 async/await 代码中的自引用结构有效，不能在 safe code 中 crash Rust 的强安全保证。**这段代码其实是有问题的，后面会提到**。

目前的 Pin 的设计，最核心的 API 是 get_unchecked_mut，这里的 **SAFETY** 保障是：如果 `<P as DerefMut>::Target ` 是 Unpin，那么无法通过安全手段得到其可变引用，否则就可能移动其内容，导致自引用失效（99%的Unpin都发生在自引用结构上）

结合上面的 Unpin 概念，我们可以针对基本类型和自引用结构分别探讨一下合理性。

基本类型是 Unpin 的，因此可以通过 `Pin::new` 创建，也可以通过 `b.get_mut()` 获取可变引用，跟普通使用指针没有任何区别

```rust
let mut a = 1;
let b = Pin::new(&mut a);
let c = b.get_mut();
```

接来下考虑 Pin 在 async/await 这种自引用结构上的运用（前提是已经分配在堆上了）。为什么对于一个 `Box<impl Future<...>>`，我们不应该安全的获取代码块的可变引用？

换句话说，如果我们可以安全地获取，可能会怎样违背 Rust 的安全准则？  

```rust
let a = Box::new(async {}); // a is pined on the heap, but <A as DerefMut>::Target is still Unpin
let mut b = unsafe { Pin::new_unchecked(a) };
let c = b.as_mut();
// let d = c.get_unchecked_mut();
let d = unsafe { c.get_unchecked_mut() };

```

如果可以安全获取到 &mut GenFuture，一旦在两次 poll 之间，被不知情的用户通过 `mem::swap()` 等手段将其移动出去，就会导致自引用结构失效 💥 。因此，`Pin::get_mut_unchecked()` 需要通过 unsafe 来让用户自己做出保证。

前面我们也提到，如果想要通过 `Pin::new()` 这种方式安全地构造 Pin，需要保障 `<P as DerefMut>::Target: Unpin`，对于 Future 来说，也就只能用 Box 包装两层，第一次是对内部 Future 的 Box，第二次是对指向 `async {}` 的指针的 Box，然后我们也可以通过 `b.as_mut().get_mut()` 的方式安全地获取到 `&mut Box<...>`，WOW！我们可以移动里面的结构了耶！那么 Rust 崩了吗？就这？    


等等，别高兴太早，注意到 Future::poll 的方法签名，要求被 poll 的 future 必须要被 Pin 保护起来，我们这里再怎么移动 Box 里面的 Future，都不可能影响 Future::poll。一个 Box 压根就无法被 poll

```rust
let a = Box::new(Box::new(async {}));
let mut b = Pin::new(a);
let c = b.as_mut().get_mut();
```


## 三. 小结

使用 Pin 主要是为了避免自引用 Future 的移动问题，核心会关注两点：
1. 是否分配在堆上
2. 在给出了 Pin<&mut Self> 之后，是否能安全地获取到 &mut Self

前者面向 executor，后者面向 reactor  

## 3.1 第一个问题，Heap or Stack?  

```rust
// 1. Pin to the heap 

// Box::pin(future) : Pin<Box<T>>
// for example
let a: Pin<Box<dyn Future>> = Box::pin(future);

// 2. Pin to the stack

let f: impl Future = async {};
// pin-utils crate
pin_utils:pin_mut!(f);
f: Pin<&mut impl Future<..>>;
```

对于一个分配在堆上的 Future 来说，当然可以随意"移动"，这里的移动指的是，`Pin<Box<dyn Future>>` 这个结构的所有权不管如何转移，该 Future 的内存一直(Pin)在堆上，直到 Box 被 Drop 才会被释放，比如下面的例子中，a 首先被 poll，不管之后所有权如何转移，改变的也只是栈上的指针位置，该 `Box<dyn Future>` 的内容仍然在堆上，因而自引用也是有效的

```rust
async fn bar(){}

let a = Box::pin(aysnc{ bar().await; 1});
a.poll(...);
let b = a;
let c = b;
// 仍然是有效的
c.poll(...);
```

对于一个分配在栈上的 Future 来说，如果我们想要手动 Poll 这个 Future，也需要将其 Pin 住，才能满足 `poll(self: Pin<&mut Self>, cx: &mut Context<'_>)` 的方法签名，有两种方式可以实现：
1. 通过 unsafe 代码，`Pin::new_unchecked(&mut a)`
2. 通过 pin-utils 这个 crate 提供的 pin_mut! 宏来解决  

```rust
async fn bar(){}

// 1. unsafe 实现，需要手动保证 a 不被 move
let mut a = bar();
let f = unsafe { Pin::new_unchecked(&mut a) };
f.poll(...);
let c = a;
f.poll(..); // BOM!

// 2. pin_mut 宏实现，更加 clever 的方式，回避了 a 被 move 的可能
let a = bar(); // a 是分配在栈上的
pin_mut!(a);  // a now is Pin<&mut impl Future<Output=()>>
a.poll(...)

```

显然，对于栈上的 GenFuture 来说，仅仅是 `let c = a` 这种转移所有权的方式，也会让 Pin<&mut ...> 失效，栈上的 Future 一旦已经 poll，根本无法在线程间转移，更别提调度了，只有在没有堆内存分配的嵌入式系统中，才会实现特殊的 runtime，保证一定在特定的 Stack 上去 poll 这个已经 Pin 住的 Future。因为一旦离开了这个执行栈，这个 Future 也就失效了。而第二种方式中的 pin_mut!，由于 Rust 卫生宏的特性，将变量 a 覆盖了，就无法通过除非安全代码之外的方式获取到 `&mut a`，从而保证安全性。

对比了 `Box::pin 和 pin_mut!`，我们可以得出下面的结论：
- 当需要返回一个 Box，或者要将一个 Future 保存在结构体中，需要用 `Box::pin`
- 当目的是在某个函数中使用 Future，更应该用上 pin_mut!，可以节省 heap allocation 的开销

这里必须要补充一个知识点，也就是 Future 的两种执行方式:
1. runtime::spawn(future)   -- 顶层 Future
2. future.await                       -- 交给 parent Future 执行

第一种方式中，runtime 会首先对顶层的 Future 做一次堆分配，我们就叫它 root Future，无论 root Future 是否是 Unpin 的，`Box<dyn Future>` 一定实现了 Unpin，就可以交给 executor 安全地执行 poll 了

第二种方式中，该 Future 会被 parent Future(async) **感知**，在被 grand parent 感知...，到最后也一定是通过runtime::spawn 来执行的。如果将其视为一颗 Future 树，内部所有的子孙 Future 都会复用 root Future 分配的空间，也就是 spawn 中一次性分配在堆上的空间。

### 3.1 第二个问题，获取 &mut Self

针对 async/await 得到的 `Pin<&mut Future>`，Pin 可以确保不会在安全代码中得到 `&mut Future`，从而被滥用；针对手动实现、没有自引用结构的、面向底层 reactor 的 Future，则可以没有任何代价地获取到 `&mut Future`。

针对 executor 和 reactor，有两种不同的 Future，async/await 是 high-level，面向 executor 的 Future，手动实现的是 low-level，面向系统的 Future。只有前者会导致自引用问题，但是 Rust 团队花了这么多精力去修补完善，这才拿出了这个比较好用的 API，如果不了解这些，完全没有办法想象，居然有这么多考量的因素。