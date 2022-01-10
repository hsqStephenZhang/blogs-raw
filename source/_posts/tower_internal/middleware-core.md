---
title: 1.3 tower 中间件模型
categories: [tower, "1.basic"]
date: 2022-1-10 13:47:40
---

## 1. 洋葱圈模型

主流的中间件，大多提供了下面这种洋葱圈模型，方便用户在一个调用中插入自定义逻辑预处理（pre-handle）及后处理（post-handle），配合handler实现一个完整的请求处理生命周期。

比较形象的图示： 


<div align=center>
<img src="./assets/middleware1.png" width="70%" />
<br/>
<br/>
</div>


上面这种表示比较抽象，进一步结合实际中间件中的 pre_handler 和 post_handler，可以表示为下面的结构：


<div align=center>
<img src="./assets/middleware.png" />
<br/>
<br/>
</div>


## 2. 深入中间件



如果深入中间件，就会发现，实现起来并不是一件多么困难的事情，我们所要做的就是将 Service 层层封装，最终转换成一个最外层的 Service 暴露出去，但是 Service 需要进行区分：
1. 中间件。对于中间层的 Service，本质是对内层的装饰，在 pre_handler 之后，调用内层的 Service，并且在调用完成之后，处理 post_handler 逻辑。  
2. 核心处理函数。最核心的 Service 需要处理业务处理逻辑，无内层 Service，处理结束之后，从这里开始逐层返回 Response。



实践起来，需要解决两个问题：如何包装？如何保存所有的中间件？

其实第一个问题，我们已经在 tower-core 一章中详细描述了，这里不再赘述，这里来详细分析一下如何保存中间件：
1. 数组。
2. 链表。



在 Rust 当中，将中间件保存在数组中，其实不是一件很直观的事情，因为数组要求所有中间件有相同的类型。但也不是没有办法：`Box<dyn Layer>`  动态派发，也就是将中间件保存为 `Vec<Box<dyn Layer>>`。但是这样做显然是有代价的，Box 的开销能避免就避免。

如果用链表来存储中间件，也存在这样的问题，仍然需要借助 Box 来完成动态派发，而且链表的局部性更差，性能也更拉胯。但优点是中间件的添加和删除迅速，灵活。



上面似乎是两瓶毒药，难道我们不得不选择一瓶喝下去吗？其实不然，Tower 选择了一个最 Rusty 的解决方案，虽然也有种种弊端，但算是符合 zero-cost 的哲学。



### 2.1 Tower 中间件如何组织



Tower 选择解决 Rust 的类型推导，来高效地解决这个问题。此话怎讲？



Rust 中泛型算是比较完善的了，其核心在于类型推导。大致可以认为，对于一个带泛型参数的类型而言，其泛型是根其构造时候的参数决定的。这一点很关键。



考虑下面的 Stack 结构，其有两个泛型参数：Inner 和 Outer，会在 `Stack::new` 方法中，根据外界传入的参数推导而来

```rust
#[derive(Clone)]
pub struct Stack<Inner, Outer> {
    inner: Inner,
    outer: Outer,
}

impl<Inner, Outer> Stack<Inner, Outer> {
    /// Create a new `Stack`.
    pub fn new(inner: Inner, outer: Outer) -> Self {
        Stack { inner, outer }
    }
}
```



那么，我们可以构造出各种 Stack 类型出来:

```rust
let a:Stack<i32, u32> = Stack::new(1, 2);

type A = Stack<i32, u32>;

let b:Stack<A, i8> = Stack::new(a.clone(), 3);

let c:Stack<A, A> = Stack::new(a.clone(), a.clone();
```



理论上这个 Stack 的泛型可以无限发散，但我们也只需要其中的一小部分就可以了，

```rust
let a:Stack<L1, ()> = Stack::new(layer1, ());   // typeof 是我自创的语法，大致意思可以参照C语言的 typeof 关键字
let b:Stack<L2, typeof(a)> = Stack::new(layer2, a);
let c:Stack<L3, typeof(b)> = Stack::new(layer3, b);
let d:Stack<L4, typeof(c)> = Stack::new(layer4, c);
```



我们可以无限给 Stack 叠加 Layer，而这里的 `:Stack<L1, ()>` 这种类型，完全不需要我们手写，**编译器能帮我们搞定！**

但同时我们也看到，每次在原有的 Stack 上叠加一层，都会创建一个全新的 Stack 类型，这就足以满足我们的需求了。



但身为框架的开发者，也不希望用户每次都要像上面一样，手动构造出最终的 Stack 类型，这时，Rust Builder 模式就发挥用武之地了。

Tower 提供了一个 ServiceBuilder，可以很轻松地在上面叠加中间件层，我们来看一下内部是如何实现的。



在 `ServiceBuilder::layer` 这个核心方法中，所做的就是不断修改 Stack 的结构。ServiceBuilder 同时提供了一些内置的中间件 Layer，也仅仅是针对 layer 函数的封装。

```rust
#[derive(Clone)]
pub struct ServiceBuilder<L> {
    layer: L,
}


impl<L> ServiceBuilder<L> {

   pub fn layer<T>(self, layer: T) -> ServiceBuilder<Stack<T, L>> {
      ServiceBuilder {
         layer: Stack::new(layer, self.layer),
      }
   }
   ... 
}
```



注意一点，layer 函数每次都会接收一个 `self`，而不是 `&mut self`。这也正应了上面所说的：Stack 类型前后不兼容，`self` 的类型已经在其创建的时候推导（并固定）下来了，**唯一**的方式就是每次创建一个新的 `Self` ，并借助**类型系统推导**出来。这一点虽然自己一个人很难想出来，但看到 ServiceBuilder 里面这样用，的确也是那么回事。





### 2.2  小结



基本原理已经介绍清楚了，也是时候来做一个小结。



为了防止动态派发带来的开销，以及提供更多的灵活性，Rust 选择了类型推导这条别的语言**从未设想的道路**。很精妙，很高效，那么代价是什么？

计算机世界中没有银弹，Tower 的 Stack Layer 组织模式，虽然实现了高性能，但同时也带来了 **类型上的开销**。是的，没错，类型也是有开销的，更多的是在程序员的心智负担上！如果你使用过 Tower，一定（或曾经）对这过程中繁杂的类型参数不胜其扰。一个很细微的错误，可能就会给你报很长的错误，大多是类型不匹配导致的，如果不太熟悉这套类型系统，甚至不知如何下手...... 

我有时候甚至在想，是否真的有必要追求那么极致的性能，如果就是用 `Box<dyn Layer>` 和 `Box<dyn Service>`，这个世界是不是会清净许多？

Anyway，既然用上了这一套，就先把它吃透，毕竟已经是 Rust 的中间件既定标准了，生态上也不用担心，有 Tokio，tonic 撑腰，前景还是很明朗的，况且，掌握了这些，再去写一个 golang 的中间件框架，应该是小意思吧 🐶



后续教程会详细说明，应该如何编写一个中间件，以及从 Tower 内置中间件中，我们可以学到的编码规范和技巧。

Tower 内置的中间件如下（打✔️的是会详细介绍的中间件）：

1. buffer(cache)  ✔️
2. rate limit    ✔️
3. concurrency limit ✔️
4. timeout ✔️
5. filter
6. retry
7. reconnect
8. route
9. discover