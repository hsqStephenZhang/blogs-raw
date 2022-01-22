---
title: 2.3 tower-timeout limit
categories: [rust, tower, middlewares]
date: 2022-1-10 13:47:40
---

Timeout 这个中间件的责任也非常明确：如果某个服务调用时间过长，超出了规定的超时时间，直接返回错误

## 1. 模型抽象

为此，Timeout 肯定需要保留下面两个结构

1. inner Service
2. 超时时间 duration

```rust
#[derive(Debug, Clone)]
pub struct Timeout<T> {
    inner: T,
    timeout: Duration,
}
```

参考 RateLimit，在这里需要明确定时器的工作起止时机。对于 Timeout 来说，我们关注的是内部服务调用的时长，因此，只有通过 `inner.call(req)` 发生了调用，才会开始计时，调用内部服务时候，每次返回 Pending 状态之后，再来检查定时器是否超时，做出响应的处理

仔细一看，这和我们之前自定义的 LogService 有一定相似之处：都需要在内部 Service 返回一定状态之后，添加 post-hook 逻辑，只不过 LogService 是在返回 Poll::Ready 之后输出 Response 内容，这里是在返回 Pending 之后，再去检查定时器。

对 Timeout 的场景再做一层抽象，其实和 `select!` 非常类似：按顺序循环等待多个任务执行，如果前面的 future 返回正确结果，直接返回，否则 falldown，检查优先级较低的 future 的结果。但这也要求在 async 代码块中执行这个操作。

## 2. 为 Timeout 实现 Service trait

接下来就试着将刚刚分析的模型用代码表述出来，大体是下面的结构：

1. poll_ready 函数中，仅仅是转发 inner service 的 poll_ready 结果，这没有什么好说的
2. call 函数中，一旦我们通过 `inner.call(reqeust)` 调用内层服务，马上设置 sleep 这个定时器

最关键的问题是，返回什么样的 Future？

```rust
impl<S, Request> Service<Request> for Timeout<S>
where
    S: Service<Request>,
    S::Error: Into<crate::BoxError>,
{
    type Response = S::Response;
    type Error = todo!();
    type Future = todo!();

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        match self.inner.poll_ready(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(r) => Poll::Ready(r.map_err(Into::into)),
        }
    }

    fn call(&mut self, request: Request) -> Self::Future {
        let response = self.inner.call(request);
        let sleep = tokio::time::sleep(self.timeout);

        todo!()
    }
}
```

和 LogService 类似，我们在这里也有两个选择：

1. 手动实现 Future，包装 response
2. 返回 `Box::pin(async { todo!()})`

### 2.1 高级 Future

先来看后者，通过 `Box::pin` 封装一个 async 代码块，虽然带来了额外的开销，但是有一个最大的好处：**可以用上 async/await 这组高级抽象**。这也带来了一连串的效应：可以使用 `select!` 宏来简化我们的实现，更加优雅，大体实现如下所示

```rust
fn call(&mut self, request: Request) -> Self::Future {
    let response = self.inner.call(request);
    let sleep = tokio::time::sleep(self.timeout);

    Box::pin(async move{
        tokio::select! {
            v = response => {
                Poll::Ready(v.map_err(Into::into))
            }
            _ = sleep => {
                Poll::Ready(Err(Elapsed(()).into()))
            }
        };
        Poll::Pending
    })
}
```

虽然 async 可以简化 Future 的编写，但也带来了额外的开销，真正 zero-cost 的众人就轮到了 **手动实现 Future** 手上

### 2.2 手动实现 ResponseFuture

在 LogFuture 的实现中也提到了 pin_project! 的必要性：因为我们在 ResponseFuture::poll 函数中只能获取到 Poll<&mut Self>，但是我们真正想要做的是 poll 内部的 future，这就要求从 Pin<&mut Self> 投影到 Pin<&mut T>，同时也应当投影到 Pin<&mut Sleep>，而 pin_proejct! 在内部也为我们保证安全性，可以放心大胆使用。

```rust
pin_project! {
    #[derive(Debug)]
    pub struct ResponseFuture<T> {
        #[pin]
        response: T,
        #[pin]
        sleep: Sleep,
    }
}

impl<T> ResponseFuture<T> {
    pub(crate) fn new(response: T, sleep: Sleep) -> Self {
        ResponseFuture { response, sleep }
    }
}
```

为 ResponseFuture 实现的 Future trait 也非常简单，我们要做的仅仅是通过投影，分别获取到 Pin<&mut InnerService> 和 Pin<&mut Sleep>，参考 select!，轮流去查看这两个 Future 是否已经 Ready 即可，分为三种情况：

1. response 为 Ready。返回该结果
2. response 为 Pending，sleep 也为 Pending。此时内部服务仍在调用过程中，且没有超时
3. response 为 Pending，sleep 已经返回 Ready。说明内部服务调用已经超时了，返回 Error

```rust
impl<F, T, E> Future for ResponseFuture<F>
where
    F: Future<Output = Result<T, E>>,
    E: Into<crate::BoxError>,
{
    type Output = Result<T, crate::BoxError>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();

        // First, try polling the future
        match this.response.poll(cx) {
            Poll::Ready(v) => return Poll::Ready(v.map_err(Into::into)),
            Poll::Pending => {}
        }

        // Now check the sleep
        match this.sleep.poll(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(_) => Poll::Ready(Err(Elapsed(()).into())),
        }
    }
}
```

最后千万别忘记了给 Timeout 创建一个对应的 Layer，否则无法在 ServiceBuilder 中创建这个服务
TimeoutLayer 只需要**透传**给 Timeout 一个 timeout 参数即可，实现起来也非常容易。  
至此，超时控制中间件圆满完成。

```rust
#[derive(Debug, Clone)]
pub struct TimeoutLayer {
    timeout: Duration,
}

impl TimeoutLayer {
    /// Create a timeout from a duration
    pub fn new(timeout: Duration) -> Self {
        TimeoutLayer { timeout }
    }
}

impl<S> Layer<S> for TimeoutLayer {
    type Service = Timeout<S>;

    fn layer(&self, service: S) -> Self::Service {
        Timeout::new(service, self.timeout)
    }
}
```

## 3. 总结

Timeout 中间件也遵循了中间件编写的几个准则:

1. 首先考虑中间件的功能，为 Timeout 添加对应的字段
2. 考虑 Timeout::call 函数中应当返回什么样的 Future，是 high-level 的 `async {..}`，但是有额外的开销，还是 low-level 的手动实现 Future，麻烦但 zero-cost，需要进行权衡
3. 最终通过 TimeoutLayer，为 Timeout 添加创建流程，相对来说是一个 boilerplate
