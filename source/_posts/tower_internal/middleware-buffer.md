---
title: 2.5 middleware-buffer
categories: [tower, "2.middleware"]
date: 2022-1-10 13:47:40
---

不知道大家有没有发现，Tower Service trait 中，要求 `ready + call` 都接收一个 `&mut self`，这也就意味着，我们无法将其很好地扩展到多线程上！

你可能会觉得，针对每一个 service，调用一下 `service.clone()`，再传入不就好了吗？但假如一个 Service 包装了很多中间件，clone 的开销也是不可小觑的。

Tower 提供了这样一个中间件：将一个 `multiproducer, single consumer channel` 和某个服务绑定，客户端可以通过任意 Clone 得到的 Buffer Service 句柄发送请求，由 `single consumer`，也就是 Worker 依次处理请求，为我们探测内层 Service 的状态，等到，同时将 Response 通过某种方式发送回请求者。

## 1. channel + semaphore 模型

进一步分析 Buffer 的定义：需要一个 mspc channel 来存放所有的请求，有多个 service 句柄可以发送请求，单个 worker 处理请求，但同时我们又希望对于该 channel 容量加以控制，更专业的术语叫做 `backpressure`，即需要根据**有限的服务处理能力**限制请求数量。
不难想到很多异步 runtime 都提供的 bounded mspc channel 非常适合描述这种模型，并且 tokio 等 runtime 都已经提供了完善的实现，可不幸的是，`Tokio bounded mspc channel` 并没有提供 polling-based 的接口，因此，我们不得不自己造一个 bounded mspc channel 的轮子。

幸运的是，我们也不是从零起步，完全可以借助已有的一些设施：`unbounded channel + semaphore`
Semaphore 这个之前已经在 [ConcurrencyLimit](./middleware-concurrency.md) 中使用过的数据结构，没错，又是将服务处理能力抽象为 Semaphore 的计数，每次需要向 Semaphore 获取许可 permit，才能接着往下进行，并且当请求处理结束之后，需要将许可返还给 Semaphore，这样才能循环利用。
有了 Semaphore，就可以对 unbounded channel 容量做出限制，正好符合我们的需求。

因此 Buffer 定义了下面的结构：
1. tx 是 mpsc channel 的发送端
2. semaphore 是所有 service 句柄共享的信号量
3. permit 是当前 service 获取的服务调用许可
4. handle 是对于服务内部错误的处理

```rust
#[derive(Debug)]
pub struct Buffer<T, Request>
where
    T: Service<Request>,
{
    tx: mpsc::UnboundedSender<Message<Request, T::Future>>,
    semaphore: PollSemaphore,
    permit: Option<OwnedSemaphorePermit>,
    handle: Handle,
}

#[derive(Debug)]
pub(crate) struct Handle {
    inner: Arc<Mutex<Option<ServiceError>>>,
}
```

Buffer 保存了 channel 的发送端，每一个 clone 出来的 buffer，都可以发送请求，与此同时还需要有一个对应的接收端，从 channel 中不断取出请求，这样才能驱动 `producer-consumer` 模型运转，不难想象我们其实在 Buffer::new 中处理了这件事

```rust
impl<T, Request> Buffer<T, Request>
where
    T: Service<Request>,
    T::Error: Into<crate::BoxError>,
{
    pub fn new(service: T, bound: usize) -> Self
    where
        T: Send + 'static,
        T::Future: Send,
        T::Error: Send + Sync,
        Request: Send + 'static,
    {
        let (service, worker) = Self::pair(service, bound);
        tokio::spawn(worker);
        service
    }

    pub fn pair(service: T, bound: usize) -> (Buffer<T, Request>, Worker<T, Request>)
    where
        T: Send + 'static,
        T::Error: Send + Sync,
        Request: Send + 'static,
    {
        let (tx, rx) = mpsc::unbounded_channel();
        let semaphore = Arc::new(Semaphore::new(bound));
        let (handle, worker) = Worker::new(service, rx, &semaphore);
        let buffer = Buffer {
            tx,
            handle,
            semaphore: PollSemaphore::new(semaphore),
            permit: None,
        };
        (buffer, worker)
    }

    fn get_worker_error(&self) -> crate::BoxError {
        self.handle.get_error_on_closed()
    }
}
```


## 2. 总结

在 Buffer 这个中间件中，我们不仅学习到 mspc 的请求处理模式还需要注意一个坑

如果我们定义了下面的中间件，简单地将内部的服务包装一层，返回一个 Box::pin 类型的 Future，看上去一切 Ok，但是，当内层服务包含了 Buffer 的时候，会出现一个巨大的问题！

```rust
impl<S> Service<hyper::Request<Body>> for MyMiddleware<S>
where
    S: Service<hyper::Request<Body>, Response = hyper::Response<BoxBody>> + Clone + Send + 'static,
    S::Future: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = futures::future::BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: hyper::Request<Body>) -> Self::Future {
        let inner = self.inner.clone();
        Box::pin(async move {
            let response = inner.call(req).await?;
            Ok(response)
        })
    }
}
```

具体什么问题呢？经过试验，出现了这个错误信息：
`panicked at 'buffer full; poll_ready must be called first'`

what？我们的调用方式明明是 `svc.ready().await.unwrap().call(req).await;`，按道理已经调用过 `poll_ready()` 了
进一步查看 Buffer 的代码中，是这样确保 poll_ready 和 call 之间的调用次序的：在 poll_ready 中获取 permit，在 call 中取出 permit，如果 call 中取出的 permit 为 None，说明错误调用了 poll_ready 和call，问题的关键在于 **permit**。

```rust
// poll_ready:
let permit =
    ready!(self.semaphore.poll_acquire(cx)).ok_or_else(|| self.get_worker_error())?;

// call:
let _permit = self
    .permit
    .take()
    .expect("buffer full; poll_ready must be called first");
```

定位到这一点，接下来的问题就很好找了，在 MyMiddleware::call 中，实际上 Clone 得到的服务，里面已经没有 permit 了，或者说，每当一个 Buffer Clone 出去，该 clone 的服务都无 permit，这一点通过源码也很容易定位，其注释也强调了这一点：Clone 得到的 service，必须要 poll_ready 返回 Ready 才能调用。

```rust
impl<T, Request> Clone for Buffer<T, Request>
where
    T: Service<Request>,
{
    fn clone(&self) -> Self {
        Self {
            tx: self.tx.clone(),
            handle: self.handle.clone(),
            semaphore: self.semaphore.clone(),
            // The new clone hasn't acquired a permit yet. It will when it's
            // next polled ready.
            permit: None,
        }
    }
}
```

我们解决的办法很简单，因为只有已经 ready 的服务才能调用，所以，我们 Clone 得到一个未 ready 的服务，然后将其和当前服务置换，这样闭包捕获的 inner 就是已经获取了 Semaphore permit 的服务了，再调用就不会有问题。

```rust
fn call(&mut self, req: hyper::Request<Body>) -> Self::Future {
    let clone = self.inner.clone();
    let mut inner = std::mem::replace(&mut self.inner, clone);

    Box::pin(async move {
        // Do extra async work here...
        let response = inner.call(req).await?;

        Ok(response)
    })
}
```
