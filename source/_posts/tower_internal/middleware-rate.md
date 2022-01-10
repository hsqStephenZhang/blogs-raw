---
title: tower-rate limit
categories: tower
date: 2022-1-10 13:47:40
---

Tower 自带的流量控制中间件，核心功能为：**在 x 时间内，最多接收 n 个请求，如果超出最大流量，休眠直到下一个刷新点**

## 1. 核心数据结构

为了描述这个核心功能，我们定义了下面的核心数据结构:
1. rate: 描述一段时间内，最多允许多少请求通过
2. sleep: 定时器，表示休眠的时间

而这里的 RateLimit 就是我们想要对外提供的 Service，通过相对应的 RateLayer 创建，结构非常清晰。  
接下来的主要问题是，我们应该如何为 RateLimit 实现 `Service trait`？

```rust
use tokio::time::{Instant, Sleep};
use std::time::Duration;


/* Rate */
#[derive(Debug, Copy, Clone)]
pub struct Rate {
    num: u64,
    per: Duration,
}

/* Service */
#[derive(Debug)]
pub struct RateLimit<T> {
    inner: T,
    rate: Rate,
    sleep: Pin<Box<Sleep>>,
}

/* Layer */
#[derive(Debug, Clone)]
pub struct RateLimitLayer {
    rate: Rate,
}

impl RateLimitLayer {
    /// Create new rate limit layer.
    pub fn new(num: u64, per: Duration) -> Self {
        let rate = Rate::new(num, per);
        RateLimitLayer { rate }
    }
}

impl<S> Layer<S> for RateLimitLayer {
    type Service = RateLimit<S>;

    fn layer(&self, service: S) -> Self::Service {
        RateLimit::new(service, self.rate)
    }
}

```

## 2. 为 RateLimit 实现 Service trait

我们的想法很简单：通过一个计数器来计算请求了多少次，并且设置定时器，每隔一段时间就进行刷新，如果在这段时间内，统计出来请求数目高于我们规定的最大限额，就应该拒绝服务，否则，返回 Ready 状态

在这个设计中，我们也发现了， RateLimit 组件其实是一个 Stateful 的有状态组件，因此之前的设计其实存在一些缺陷，需要保存 RateLimit 的状态
接下来就创建该结构，并且给 RateLimit 添加一个字段 state

```rust
#[derive(Debug)]
enum State {
    // The service has hit its limit
    Limited,
    Ready { until: Instant, rem: u64 },
}

#[derive(Debug)]
pub struct RateLimit<T> {
    inner: T,
    rate: Rate,
    state: State,
    sleep: Pin<Box<Sleep>>,
}
```

这里还有一点需要明确，当我们因为请求数目过多，需要拒绝服务的时候，理应在 poll_ready 阶段直接返回 Err，这样就可以使得用户尽早感知，避免通过 RateLimit::call 创建 Future，带来不必要的开销。但是，对于 **请求量** 的统计，只能在 call 函数中实现，因为用户完全有可能在 poll_ready 之后没有调用这个 Service。  

有了上面这两点，就很容易设计了：
1. poll_ready 函数中，判断 Self.state，如果是 Ready，返回内部服务的状态，否则如果是 Limited 状态，说明我们已经进入了休眠状态，需要查看是否到了下一次刷新时间点，如果已经到点，我们就可以将原先的计数刷新，开始新一轮的**流量控制**，也只有在刷新之后，此时才会去判断内层服务是否 ready 
2. call 函数中，我们需要完成对请求数目的统计，首先统计当前请求的时间，如果已经开始下一轮流量控制，需要重置计数。**当剩余能够对外提供服务的数量** 这个计数递减到 0 的时候，状态将从 Ready 转变为 Limited，同时设置定时器，表示直到下次请刷新之前，都会拒绝服务

RateLimit 的代码实现也非常清晰，总的来说是一个良好设计的中间件，并且也能够在实际服务中派上用场

```rust
impl<S, Request> Service<Request> for RateLimit<S>
where
    S: Service<Request>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        match self.state {
            State::Ready { .. } => return Poll::Ready(ready!(self.inner.poll_ready(cx))),
            State::Limited => {
                if let Poll::Pending = Pin::new(&mut self.sleep).poll(cx) {
                    tracing::trace!("rate limit exceeded; sleeping.");
                    return Poll::Pending;
                }
            }
        }

        self.state = State::Ready {
            until: Instant::now() + self.rate.per(),
            rem: self.rate.num(),
        };

        Poll::Ready(ready!(self.inner.poll_ready(cx)))
    }

    fn call(&mut self, request: Request) -> Self::Future {
        match self.state {
            State::Ready { mut until, mut rem } => {
                let now = Instant::now();

                // If the period has elapsed, reset it.
                if now >= until {
                    until = now + self.rate.per();
                    rem = self.rate.num();
                }

                if rem > 1 {
                    rem -= 1;
                    self.state = State::Ready { until, rem };
                } else {
                    // The service is disabled until further notice
                    // Reset the sleep future in place, so that we don't have to
                    // deallocate the existing box and allocate a new one.
                    self.sleep.as_mut().reset(until);
                    self.state = State::Limited;
                }

                // Call the inner future
                self.inner.call(request)
            }
            State::Limited => panic!("service not ready; poll_ready must be called first"),
        }
    }
}
```

## 2. 总结

和上一小节实现的 LogService 相比，大体流程没有变化：
1. 自定义 RateLimit，实现 Service
2. 自定义 RateLimitLayer，实现 Layer trait，并且在 layer 函数中创建 RateLimit 这个 Wrapper Service

不太一样的是，这里在 RateLimit 中维护了一个状态 State，实际上这种模式在中间件当中是十分常见的，对于这种 State，我们定义一个良好的结构，用来描述 RateLimit 可能处于的状态，目标就完成了一大半，接下来就与实现 Future trait 类似，手动完成状态的转换，驱动中间件不断运转

这里一直没有提到，RateLimit 有一个非常致命的问题：无法 Clone！之所以这是一个很大的问题，是因为我们在处理请求的时候，为了横向扩展，会希望通过 tokio::spawn 处理每一个请求，也就要求 service 是可以 Clone 的，这样才能将其移动到 aysnc {..} 代码块中，整个服务的运转流程如下所示

```rust
let mut service = ServiceBuilder::new()
    .layer(l1)
    .layer(l2)
    .layer(l3)
    .service(coreService);

loop{
    let conn = get_connection(..).await;
    let s = service.clone();

    // 每一个 connection，都为其 clone 一个 "一模一样" 的 service
    tokio::spawn(async move {
        /* s and conn were moved here */
        handle_conn(...).await;
        ...
    })
}
```

RateLimit 无法通过 Clone 创建，其实是有一定道理的：单线程环境中，完全没有必要将 Service Clone 出来再使用，只有在多线程多任务的模型中，才需要对每一个请求都 Clone 一个相应的 Service。但 RateLimit 的定义是：**对单个服务一段时间内请求量的控制**，这完全是针对单线程做出的限制。
后面小节中，我们也可以看到，对于多线程服务模型中的请求量，我们也可以做一定的限制，那就需要换一个名词了：**并发控制**



