---
title: 2.1 middleware custom
categories: [tower, "2.middleware"]
date: 2022-1-10 13:47:40
---

# 自定义中间件



先来回顾一下 Tower 当中的中间件模型：

通过 Service Trait 定义服务功能，通过 Layer Trait 搭建骨架，并通过 Stack 将所有的中间件粘合起来。并且， Service 和 Layer 通常是一一对应的关系。



本节就来带着大家动手实现实现一个日志中间件。



## 1. 日志中间件实现



以一个最简单的 Echo 业务 + 最简单的中间件 Log 为例，大致分为几个模块：

1. 核心业务 EchoService
2. LogService 
3. LogLayer



### 1.1 核心业务 EchoService



首先定义 Echo  服务的业务逻辑，Echo 所做的就是原样返回，再简单不过了，为了简化泛型的处理，Request，Response，Error 类型都定义为 String

```rust
// 这里导入的其实是后续所有代码所需的模块
use std::future::Future;
use std::pin::Pin;
use std::task::*;

struct EchoService;

impl Service<String> for EchoService {
    type Response = String;
    type Error = String;
    type Future = Pin<Box<dyn Future<Output=Result<Self::Response, Self::Error>>>>;

    fn poll_ready(&mut self, _: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: String) -> Self::Future {
        println!("request:{}", &req);
        Box::pin(async { Ok(req) })
    }
}
```



### 1.2 LogService 结构



接下来需要定义 Log 中间件功能，首先明确目标：分别记录 request 和 response 内容（为了避免 `Debug, Clone` 这样的约束，这里将其修改为输出一段文字）：

1. 在内层 Service 调用之前，输出 "pre-call"
2. 在内层 Service 返回结果之后，输出 "post-call"



之前一直强调，Layer 是**骨架**，Service 是**血肉**，这两者是紧密结合，相辅相成的关系。在这里，不妨将 Log 中间件的 Service 和 Layer 分为称为 LogService 和 LogLayer。



首先考虑 LogService 的功能，中间件是内层服务的包装（Decorator），自然需要将内层服务保存下来，否则就直接"失联"，很自然的，在 LogService 中定义一个 `inner` 的字段，保存内层服务的句柄。对于 LogService 而言，由于其功能比较单一，也就不需要添加更多的字段了。LogService 的结构可以确定下来：

```rust
struct LogService<S> {
    inner: S,
}

impl<S> LogService<S> {
    pub fn new(inner: S) -> Self {
        Self { inner }
    }
}
```



### 1.3 Pin<Box\<dyn Future\>>  VS 自定义 Future ？

已经确定了 LogService 的结构，接下来要做的就是为 LogService 实现 `Service Trait`

对于 poll_ready 而言，本身 LogService 是无状态的，按理来说可以返回 `Poll::Ready`，但是，我们无法确定 `inner` 是否为有状态的服务，所以直接转发（forward）内层 Service 的 poll_ready，即 `self.inner.poll_ready(cx)`。

接下来的关键点在于：**LogService::call 函数中，应该返回什么样的 Future，才能实现输出日志的功能？**

```rust
impl<S, Req> Service<Req> for LogService<S>
    where
        S: Service<Req>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = LogFuture<S::Future>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Req) -> Self::Future {
        todo!()
    }
}
```



大致的解决思路是：首先获取到 `inner.call(req)` 的结果，这是一个 Future，然后在该 Future 返回的前后，分别输出日志。如下所示

```rust
  let f = self.inner.call(req);
  println!("pre-handler");
  let r = f.await;
  println!("post-handler");
  return r;

```



但是上面这段代码还是一厢情愿了。`.await` 虽然是最直观的将异步转换成同步执行的模式，但其必须要在 `async` 代码块中使用，但是，目前如果不开启 GAT(generic assosiate type) 这个 feature，无法在 Trait 中定义 `async fn`。如何处理这种困境呢？



目前有两种可行的方案，各有利弊：

1. 返回 `Box::pin(async move { ... })`

这样就可以在 async 代码块中利用 await 直接 await inner Service 调用返回的 Future，将异步调用转换成同步的方式，是一种比较 high-level 的实现方式，利用 async/await 这组高级抽象减少了心智负担，但实际上我们也付出了代价。

如果仔细阅读之前小节对于 [Pin](./why_pin_withoutboats.md) 的描述，会认识到：一个 Future 其实是一颗树状结构，只需要一次内存分配，就能保证内部所有子 Future 也都能在状态机状态变迁之后获取到相应的堆内存，而通过 enum 保存状态，这种结构的内存利用非常高效。但是在 Service trait 中，由于 GAT(generic assosiate type) 能力的不足，只能通过 Box::pin 的方式，返回一个 `Pin<Box<dyn Future<Output=Result<Self::Response, Self::Error>>>>`，而无法将 Service::Future 和 async 代码块结构的 `impl Future<Output=...>` 类型关联起来，因此需要通过`动态分发`解决这个问题，但这也造成了额外且不太必要（有GAT时）的堆内存开销，在高性能场景下，可能不是一个很好的选择。

```rust
{ 
    ...
    type Future = Pin<Box<dyn Future<Output=Result<Self::Response, Self::Error>>>>;


```



2. 手动实现 Future

如果说 `Box::pin(async move { ... })` 是一种比较 **high-level**  的解决方案，这里的第二个方案就是 **low-level** 的模式：

缺少了 async/await 这组高级抽象带来的便捷之后，只能将 `inner.call(req)` 保存到一个 LogFuture 中，并手动实现 poll，虽然比较麻烦，但也有其好处：不需要对 Futuere 的一次额外堆内存分配，性能会有所提升



事情进行到这里，逻辑上说得通：为了性能，麻烦就麻烦一点吧！只不过，又遇到了一点小问题：Pin-project。

将  `inner.call(req)` 保存到 LogFuture 中以后，这个 LogService 到底有没有实现 `Unpin trait`？之所以关心这一点，是因为在 `Future::poll` 中，我们得到的是一个 `Pin<&mut Self>`，但只有获取到 `inner: Pin<&mut I>` 才能进行真正的 poll，如何确保正确性？unsafe ？pin-project，Yes！

在 [why_pin](./why_pin_withoutboats.md) 一文中提到，`pin-project` 这个宏确保了 从 `Pin<&mut Self>` 投影到 `Pin<&mut I>` 是100%安全的，投影之后便可以对其进行 poll，也就可以在 `inner.poll(cx) `前后插入 LogService 的 hook 逻辑

```rust
use pin_project_lite::pin_project;
pin_project! {
    struct LogFuture<I> {
        #[pin]
        pub inner: I,
    }
}

impl<I, T, E> Future for LogFuture<I>
    where
        Resp: Future<Output=Result<T, E>>,
{
    type Output = Result<T, E>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        let inner = this.inner;
        println!("pre handler, called every poll");
        let r = match inner.poll(cx) {
            core::task::Poll::Ready(v) => {
                println!("post handler after get the resp"); // hook 逻辑
                Poll::Ready(v)
            }
            core::task::Poll::Pending => Poll::Pending,
        };
        r
    }
}
```



对于到 `LogService::call` 中，我们只需要将 `self.inner.call(req)` 的结果保存到 LogFuture 中，返回该 Future 即可，并且，在返回之前，插入 `pre-handler` 逻辑都是可行的（poll 函数中的 pre-handler 可能会调用多次）

```rust
{
    ...
    type Future = LogFuture<S::Future>;

    fn call(&mut self, req: Req) -> Self::Future {
        println!("pre handler before pass the req");
        let f = self.inner.call(req);
        LogFuture::new(f)
    }
}
```



### 1.4 LogLayer



完成了 LogService ，接下来让我们定义 LogLayer，其需要实现 Layer trait，接收 inner service，并创建 LogService。并且由于 LogSevice 不需要接收别的参数，这里的 LogLayer 也就不需要保存更多的信息了。



```rust
struct LogLayer;

impl<S> Layer<S> for LogLayer {
    type Service = LogService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        LogService::new(inner)
    }
}
```





## 2. 总结



通过编写一个简单的 Log 中间件，我们可以学习到下面几种规范（by Tower）：

1. 自定义 Service，保存内层 Service，并实现 Service trait
3. Service::call 方法的实现中，返回一个 Future，这里有两种选择：
    a. 返回 **Box::pin(async { ... })**
    b. 返回 **自定义Future**，保存 inner service 调用返回的 Future，并且手动实现 poll 逻辑，借助 pin_project! 实现 projection
3. 自定义 Layer，并实现 Layer trait，在 Layer::layer 中创建对应的 Wrapper Service