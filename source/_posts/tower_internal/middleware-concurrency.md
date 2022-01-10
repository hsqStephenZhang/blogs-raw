---
title: 2.4 tower-concurrency limit
categories: [tower, "2.middleware"]
date: 2022-1-10 13:47:40
---

`Concurrency` 翻译过来是 `并发`，`Concurrency Limit` 也就是 `并发控制`，其核心限制条件是：**同一时刻最多接收 n 个请求**，和 RateLimit 有一定的相似之处，内层似乎都要使用一个计数器计数，但是具体如何实现，又有很大的不同。

## 1. 并发控制模型

可以将对于请求的响应能力看成是一种**资源**，资源的数量是有限的，先到先得，那得不到的请求就应该置之不理吗？当然不是，得不到的可以排队等待，当先来的请归还了借给它的资源，资源得到循环利用，后面排队的请求就能得到处理了。

细细一想，这种场景十分常见，不说和 Semaphore 完全相同，简直就是一模一样！并且tokio 早就为我们提供了完善的 Semaphore 轮子，省去了我们自己造的痛苦。

Semaphore 本质上是一个 Mutex + Atomic 计数器，如果需要抢占资源，首先尝试通过原子指令更新计数器，如果计数器的值大于 0，说明资源还有剩余，可以获取，如果计数器的值小于 0，说明资源无剩余，只能将自身封装为 Waiter 结构，添加到 WaitList 中去，自身陷入休眠，WaitList 是一个先进先出的队列，当原先占据的资源被释放，计数器的值就可以增加，当计数器的值大于 0 时，资源又有富余了，将队首的结点取出，唤醒该任务。
对于 Rust 来说，Semaphore 模型稍有不同，因为假如按照最朴素的思想，得不到 Semaphore 就去休眠，对线程就造成了阻塞，调度器岂不是很没面子？而 Rust 的 polling-based 模型可以很好解决这一点，核心思想是：如果可以获取 Permit，返回 Poll::Ready，否则在底层注册 Waker，通过 yield 返回到最上层的 runtime，runtime 便可以调度别的任务，等待事件通知，也就是相应的资源准备好了之后，再来重新 poll semaphore，这样就实现了高效的调度。

```rust
// 最上层 Semaphore， 通过 Arc 跨线程共享
pub struct PollSemaphore {
    semaphore: Arc<Semaphore>,
    permit_fut: Option<ReusableBoxFuture<Result<OwnedSemaphorePermit, AcquireError>>>,
}

// 获取到了 `permits` 个许可，同时保存 Semaphore。便于到时候归还 permit
#[must_use]
#[derive(Debug)]
pub struct OwnedSemaphorePermit {
    sem: Arc<Semaphore>,
    permits: u32,
}

// tokio::sync::Semaphore
#[derive(Debug)]
pub struct Semaphore {
    /// The low level semaphore
    ll_sem: ll::Semaphore,
}

// ll::Semaphore
pub(crate) struct Semaphore {
    waiters: Mutex<Waitlist>,
    /// The current number of available permits in the semaphore.
    permits: AtomicUsize,
}
```

向 Semaphore 申请资源的时候，不必 one-by-one 操作，而是可以一次申请多个，Semaphore 完全看请求者申请的数量和 Semaphore 现有的资源能否匹配，如果能满足，就将这些资源返回给请求者，否则对不起，只能返回一个 Pending 状态。
如果申请资源成功，返回一个 `OwnedSemaphorePermit` 结构，表示申请到了 `permits` 个资源，同时持有 `Arc<Semaphore>`，是需要在其 drop 时，将申请的所有 permitd 悉数返还给 Semaphore 

但对于我们这里用到的结构来说，每接收一个请求，意味系统再能够处理的请求就少了一份，只需要分配出去一个默认的 `OwnedSemaphorePermit{ sem, permits:1 }` 的结构即可
眼见为实：acquire_owned 函数中，通过 `ll_sem.acquire(1)` 获取了单个资源，等待该操作完成，就能返回一个许可

```rust
pub async fn acquire_owned(self: Arc<Self>) -> Result<OwnedSemaphorePermit, AcquireError> {
    self.ll_sem.acquire(1).await?;
    Ok(OwnedSemaphorePermit {
        sem: self,
        permits: 1,
    })
}
```

## 2. 如何利用 Semaphore 实现 ConcurrencyLimit 这个 Service

熟悉了 Semaphore 的作用和大致原理，接下来要思考的是，如何用好这个结构。
Semaphore 很容易创建， `semaphore: PollSemaphore::new(semaphore)` 一行代码即可搞定，关键是如何通过 Semaphore 控制并发？或者说，如果描述 `并发量` 这种资源？

我们先来看一个结构 TODO!

类似 RateLimit，必须要明确几个时间点：
1. 在 poll_ready 函数中，如果当前已经超出并发量，需要等待 `semaphore.poll_acquire()` 返回 Poll::Ready 之后，才表明我们做好了接收请求的准备
2. 一个请求的声明周期，在这里应该是从 `ConcurrencyLimit::call` 开始，直到内层服务 call 结果返回的 Future 得到最终结果为止，我们需要将 OwnedSemaphorePermit 和这个声明周期绑定，也就是确保开始时获取到一个 Permit，结束时返还该Permit，如何实现？

ConcurrencyLimit 给出了一个非常优雅的设计

```rust
#[derive(Debug)]
pub struct ConcurrencyLimit<T> {
    inner: T,
    semaphore: PollSemaphore,
    permit: Option<OwnedSemaphorePermit>,
}

impl<S, Request> Service<Request> for ConcurrencyLimit<S>
where
    S: Service<Request>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = ResponseFuture<S::Future>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // 如果 permit 是 None，意味着之前交出的许可已经被消耗了
        if self.permit.is_none() {
            self.permit = ready!(self.semaphore.poll_acquire(cx));
            debug_assert!(
                self.permit.is_some(),
                "ConcurrencyLimit semaphore is never closed, so `poll_acquire` \
                 should never fail",
            );
        }

        // 一旦获得了许可，就可以询问内部 Service 是否 ready
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> Self::Future {
        // 这个 permit 不一定是自己申请的
        let permit = self
            .permit
            .take()
            .expect("max requests in-flight; poll_ready must be called first");

        // Call the inner service
        let future = self.inner.call(request);

        ResponseFuture::new(future, permit)
    }
}
```

但是这种结构似乎有一点问题？因为当我们连续两次调用一个 ConcurrencyLimit::call 函数的时候，第二次一定会在 `self.permit.take()` 处 panic，
这样说，难道我们轻易就能写出下面这段不安全的代码？

```rust
let mut service = ServiceBuilder::new()
    .concurrency_limit(3)
    .layer(LogLayer)
    .service(EchoService);

let mut s1 = service.clone();
let s1 = s1.ready().await.unwrap();

let mut s2 = service.clone();
let s2 = s2.ready().await.unwrap();

let r1 = s1.call("123".into()).await.unwrap();
let r2 = s2.call("123".into()).await.unwrap();
assert_eq!(r1, String::from("123"));
assert_eq!(r2, String::from("123"));
```

其实不然，你会发现上面这段代码编译不会通过，因为 ServiceExt::ready 函数中，返回的 Ready 结构包含了对于 self 的引用，是有生命周期约束的，上面这段代码显然就因为 `mutable more than once at a time` 而无法编译。

为了解决这个问题，必须这样使用 Service：在调用 ServiceExt 提供的 ready 函数之前，一定需要先 clone 出一个相同的 service
正如下面代码所示，我们创建了 service 之后，分别 clone 得到 s1,s2，然后分别提供服务

```rust
// ...

let mut s1 = service.clone();
let mut s2 = service.clone();

let s1 = s1.ready().await.unwrap();
let s2 = s2.ready().await.unwrap();

let r1 = s1.call("123".into()).await.unwrap();
assert_eq!(r1, String::from("123"));

let r2 = s2.call("123".into()).await.unwrap();
assert_eq!(r2, String::from("123"));
```

实际上，这正是我们在多线程多任务场景下希望使用 service 的方式：我们希望对于每一个到来的连接，都通过 clone 创建相同的服务，并且利用 tokio::spawn 提交任务，交给 runtime 调度执行

### ConcurrencyLimit 真正做了什么

可能稍微有一点绕，但是我在试图证明当前的 ConcurrencyLimit 正确性：
1. 无法同时有两个 service.call(req)，rust 的所有权机制防止了这一点，因此 `self.permit.take()` 不会 panic
2. 真正引用中使用 service 对 req 做出响应的时候，会将 service clone 一份，移动到 async Future 中，这就要求 ConcurrencyLimit 实现了 Clone
    - 在 ConcurrencyLimit 的 Clone 实现中，实际上每一个 clone 出去的结构，都共享了同一个底层的 Semaphore，并且会将 Option<OwnedSemaphorePermit> 结构设置为 None
        ```rust
        impl<T: Clone> Clone for ConcurrencyLimit<T> {
            fn clone(&self) -> Self {
                Self {
                    inner: self.inner.clone(),
                    semaphore: self.semaphore.clone(),
                    permit: None,
                }
            }
        }
        ```
    - Clone 得到的 `ConcurrencyLimit.permit = None`，并且，每一个 Clone 得到的 Service 在真正被 call 前，而已必须要经过 poll_ready 来确认是否准备就绪，并发控制因此而来：这些共享底层信号量的任务，将一起受到 Semaphore的限制。相当于通过 tokio::spawn 提交了 x 个任务，最多只能有 n 个被执行，n 是并发限制量


### 收尾工作

确定了 ConcurrencyLimit 的数据结构之后，在相应的 Layer 中要完成的事情就很简单了：确定 ConcurrencyLimit::new 中所需的参数，并在 Layer::layer 函数中完成相应创建

```rust
#[derive(Debug, Clone)]
pub struct ConcurrencyLimitLayer {
    max: usize,
}

impl ConcurrencyLimitLayer {
    /// Create a new concurrency limit layer.
    pub fn new(max: usize) -> Self {
        ConcurrencyLimitLayer { max }
    }
}

impl<S> Layer<S> for ConcurrencyLimitLayer {
    type Service = ConcurrencyLimit<S>;

    fn layer(&self, service: S) -> Self::Service {
        ConcurrencyLimit::new(service, self.max)
    }
}
```

## 3. 总结

上一小节提到了 RateLimit，是针对单个任务而言，限制一段时间内接收的最多请求，本节的 ConcurrencyLimit，是针对多任务而言，限制每一个时刻整个系统中最多接收的请求数。

要注意的一点就是 Service 的 Clone 问题。针对 Concurrency 而言，要通过 clone 出多任务，扩展系统的并发量，而 RateLimit 完全没有必要实现 Clone trait，即使实现了 Clone trait，其含义也是模糊不清的，这一点使用的时候一定要想好
