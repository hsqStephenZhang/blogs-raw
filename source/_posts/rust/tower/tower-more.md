---
title: tower more
date: 2021-11-24 14:01:29
---



# Tower more



这一部分，主要是探讨 Tower 中的一些不足之处，以及我们团队在使用 Tower 之后的一些体验。



## 1. 问题



### 1.1 `call `是否应该使用 `Pin<&mut Self>` ？



关于这个问题，已经有了一些[github 讨论](
https://github.com/tower-rs/tower/issues/319)。但是到目前为止，Tower 官方还是决定采用一个普通的 `&mut self`，这意味着 Service 必须是`Unpin`。



这个问题的出发点在于，如果在 Service 内部保存一个 Future，并且在 poll_ready 中使用，这就会和 call 函数的签名有一定的冲突，因为 `Service::poll_ready` 接收一个 `&mut self` ，而非 `Pin<&mut Self>`。

为了在 `poll_ready` 中保证安全，必须防止在 `poll_ready` 中调用  `Pin::new_unchecked`（因为我们无法承诺不移动这个 service），而 `Pin::new()` 要求 future 实现 Unpin 的，对于 async/await 生成的自引用 Future 来说，只有用将其分配在堆上，才能确保 Pin，因此带来了额外的开销。该讨论旨在用一个更好的方式来加以约束。



### 1.2 GAT （generic associate type）支持



GAT，正如其名字所言，允许在 trait 的关联类型上定义泛型参数，声明周期参数。这个功能很有用，尤其是当你熟悉如今的妥协方式时。

在 Tower 定义的 `Service trait` 中，要求响应返回的 Future 必须是`'static'`的，因为目前不支持 GAT 功能，也就无法将 `Self::Future`  和 `call` 的声明周期关联起来，因此会强制要求返回的 Future 具有 `'static' `  生命周期。 因为写 `Box<dyn Future>` 实际上会被 desugar 成`Box<dyn Future + 'static>`，因此在`fn call(&'_ mut self, ...) `中的匿名  `lifetime` 并不满足这个要求。在未来，Rust 编译器团队计划增加一个名为泛型关联类型（GAT）的功能，这将解决这个问题。泛型关联类型允许我们将响应的 future 定义为 `type Future<'a>`，`call`定义为`fn call<'a>(&'a mut self, ...) -> Self::Future<'a>`，但现在响应返回的 Future 必须是`'static`的。



### 1.3  filter or middleware 模式 



Tower 目前采用的 middleware 模式，**严重依赖于类型推导**，使得**运行期增加或删除中间件**非常困难，因为通过 `ServiceBuilder` 叠加 layer 之后，其泛型参数已经固定了。

你可能会好奇，运行期去增加删除中间件有什么作用？事实上，在有些场景下，还是很有必要的。

比如使用中间件实现 proxy 组件的时候，如果要进行动态更新，就依赖于运行期的增删能力，而 Tower 目前根本无法实现。有解决办法吗？有，只不过有一定的代价，也就是前文中说过的，用`Vec<Box<dyn Service>>` 或者 `LinkedList<Box<dyn Service>>` 来保存中间件 ，通过牺牲一定的性能代价，换来更多的灵活性，并且省去了很多泛型的烦恼。



同时，我们也考虑，在某些场景下，是否可以使用更加简洁的 Filter 模式去拓展功能，而不是大而全的 Middleware 模式，关于这两者的区别，可以参考 [StackOverflow 讨论](https://stackoverflow.com/questions/54863655/whats-the-difference-between-interceptor-vs-middleware-vs-filter-in-nest-js)