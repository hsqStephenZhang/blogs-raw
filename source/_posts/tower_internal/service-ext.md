---
title: 1.4 tower-ext
categories: [tower, "1.basic"]
date: 2022-1-10 13:47:40
---


## 1. ext 设计模式



xxx-ext 是 Rust 当中常用的一个设计模式，在满足某个 Trait 的前提下，引入很多的扩展功能（比如最常用的 Iterator，其实 `skip, enumerate, take` 都是针对 `Iterator trait` 的扩充）



Tower 也给 Service 引入了这个设计模式，十分好用。



源码如下，TL；DR

```rust
pub trait ServiceExt<Request>: tower_service::Service<Request> {
    #[deprecated(since = "0.3.1", note = "prefer `ready_and` which yields the service")]
    fn ready(&mut self) -> Ready<'_, Self, Request>
    where
        Self: Sized,
    {
        #[allow(deprecated)]
        Ready::new(self)
    }

    fn ready_and(&mut self) -> ReadyAnd<'_, Self, Request>
    where
        Self: Sized,
    {
        ReadyAnd::new(self)
    }

    fn ready_oneshot(self) -> ReadyOneshot<Self, Request>
    where
        Self: Sized,
    {
        ReadyOneshot::new(self)
    }

    fn oneshot(self, req: Request) -> Oneshot<Self, Request>
    where
        Self: Sized,
    {
        Oneshot::new(self, req)
    }

    #[cfg(feature = "call-all")]
    fn call_all<S>(self, reqs: S) -> CallAll<Self, S>
    where
        Self: Sized,
        Self::Error: Into<Error>,
        S: futures_core::Stream<Item = Request>,
    {
        CallAll::new(self, reqs)
    }
  
  	...
}

impl<T: ?Sized, Request> ServiceExt<Request> for T where T: tower_service::Service<Request> {}
```



`ServiceExt` 中的的几个函数，对于当前的 Service 做了不同的封装，下面会挑选几个进行细致的分析。



## 2. ReadyOneShot & Ready  实现



ReadyOneshot 结构中，保存了一个 `Option<Service>`，并且在 poll 方法中通过 `ready!` 匹配 `poll_ready` 的结果

之所以称为 ReadyOneShot，是因为 ReadyOneShort::poll 方法返回 Ready 之前会将 Option 中的 Service  取出来，如果再次 poll 这个 ReadyOneShot 结构，肯定会发生 panic（符合预期）

```rust
/// Oneshot
pub struct ReadyOneshot<T, Request> {
    inner: Option<T>,
    _p: PhantomData<fn() -> Request>,
}

// Safety: This is safe because `Services`'s are always `Unpin`.
impl<T, Request> Unpin for ReadyOneshot<T, Request> {}

impl<T, Request> ReadyOneshot<T, Request>
where
    T: Service<Request>,
{
    #[allow(missing_docs)]
    pub fn new(service: T) -> Self {
        Self {
            inner: Some(service),
            _p: PhantomData,
        }
    }
}

impl<T, Request> Future for ReadyOneshot<T, Request>
where
    T: Service<Request>,
{
    type Output = Result<T, T::Error>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        ready!(self
            .inner
            .as_mut()
            .expect("poll after Poll::Ready")
            .poll_ready(cx))?;

        Poll::Ready(Ok(self.inner.take().expect("poll after Poll::Ready")))
    }
}
```



Ready 是针对 ReadyOneshot 结构简单封装，主要是为了重复利用 Service，不需要拿到 Service 的所有权，只需 Service 的可变引用即可。Ready::poll 中所做的，也仅仅是 forward 内部 ReadyOneShot::poll 的结果。

```rust
/// Core Api，Ready
pub struct Ready<'a, T, Request>(ReadyOneshot<&'a mut T, Request>);

impl<'a, T, Request> Ready<'a, T, Request>
where
    T: Service<Request>,
{
    #[allow(missing_docs)]
    pub fn new(service: &'a mut T) -> Self {
        Self(ReadyOneshot::new(service))
    }
}

impl<'a, T, Request> Future for Ready<'a, T, Request>
where
    T: Service<Request>,
{
    type Output = Result<&'a mut T, T::Error>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        Pin::new(&mut self.0).poll(cx)
    }
}
```



你可能不禁要问了，ReadyOneShot 和 Ready 有什么好处吗？我真的需要它吗？

哎，别提，你还真离不开它！



由于 Service 将 poll_ready 函数单独抽离出来，导致我们在 call 之前，一定要先等待 poll_ready 返回，但手动去匹配 `Poll::Ready` 和 `Poll::Pending` 是一件很枯燥，但又不得不去做的事情。

得益于这套扩展，我们可以非常轻松地在代码中写出下面的链式调用 `service.ready().await.unwrap().call(req).await`，省去了大段的 **boilerplate**





## 3. 其他



再来看一下其它的拓展功能，其实在实际中用的不算太多了，但也是很有意思的功能。



比如 `call_all` 函数，会接收一个 Stream of Request，返回一个 Stream of Respone，相当于 JavaScript 里面的 `Promise.all`

而 and_then 描述的 service 执行依赖关系，相当于 JavaScript 里面的 `Promise.resolve`，这个名字听上去就很 Rusty，还有更多的组合式的扩展机制，比如 map_response, map_err, map_result，这里就不一一列举了。