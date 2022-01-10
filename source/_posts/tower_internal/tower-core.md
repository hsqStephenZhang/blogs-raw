---
title: tower 核心抽象
categories: tower
date: 2022-1-10 13:47:40
---


## 1. Tower 的起源



说起 Tower，就不得不提到 Rust 另一个重要的框架，Tokio，这两个单词连起来读，就是 Tokyo Tower（东京塔），这也从另一个方面说明，Tower 是 Tokio 官方支持的中间件框架，Axum ，Tower-web，Tonic 这些项目，都广泛使用了 Tower 这层抽象。

那么，我们不禁要问，Tower 到底是什么？官方给出的答案是：

**Tower 是一个模块化，可重用的中间件框架**



所谓的中间件，之后会详细解释，在这里，可以简单将其理解为从用户到核心服务之间的中间层。有句话说得好，计算机任何问题都可以通过添加一个间接的中间层来解决，这里也是如此。中间件可以带来很多好处，比如数据管理、应用服务、消息传递、身份验证、日志，但是作为一个**框架**，Tower 的核心任务是**如何定义一个良好的抽象，让上层开发人员能够方便地编写高质量中间件**！



所以，我们首先要来了解一下 Tower 的几个核心抽象。



## 2. Service



对于一个应用来说，可以简单将其概括为 **请求 + 相应**，类似于 `Ping-Pong` 模型，而网络又为应用赋予了异步属性，结合 Rust 的 Future 抽象，我们不难得出下面这种抽象：

`async fn handler(req:Request) -> Response {..}` 



为了更好地描述这种**服务**，更加自然的方式是将其定义为 trait，不妨称这个 trait 为 Service，而和这个 Service 想关联的下面几个类型：

1. Request 
2. Response
3. Error
4. Future



因而 `Service trait` 有如下的结构

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future;
  
    fn poll_ready(&mut self) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```



核心函数自然是 call，用来返回一个 Future，提供给 Runtime 执行。

但是这里还涉及到了 poll_ready 函数，用来表示服务是否可用，这一点在**限制流量或者并发度**的时候非常好用，如果服务本身不关心是否可用，就可以直接返回 Poll::Ready，开销很小，但是极大地提高了 Service 的灵活度。



## 3. 组合 



### 3.1 还缺少什么



 通过 `Service` 这个抽象，已经可以很好地描述一个小功能了，比如 EchoService（相信不必我多解释了），但始终不要忘记，我们是一个中间件框架，还需要将中间件组合在一起才行，这样才能提供更加强大的拓展能力。



事实上，对于中间件而言，我们可以沿用 `Service` 这种抽象，理由是：即使是添加了 n 个中间件，我们描述的**仍然是**一个异步的请求-相应服务，只不过我们想要在中间的某些环节上加以拓展，更加专业的说法是，添加 hook 逻辑，比如在调用核心服务之前和之后输出日志就是一个很基本的需求。



仔细思索，这不就是装饰器（Decorator）模式？接收一个函数，返回一个新的函数。只不过将普通的函数装饰转换成了对 `Service trait` 的装饰。



接下来的核心问题是，应该如何**一层层**把最核心的服务包装起来？

针对这个问题，其实有很多实现方式，node 的 koa，golang 的 gin 都有很好的实现方式，我们不妨先看一下 Tower 里面的使用方式再做评判。



### 3.2 Tower Layer



Tower 使用 Layer 这个 trait 来粘合 Service。

上面我们提到，中间件实际上就是对内层服务的包装，那么，肯定要先有内层的 Service，才谈得上进一步包装。因此，Tower 抽象出了 `Layer trait`，用来表示：接收内层的 Service 类型，返回一个包装之后的 Service。



因而 `Service trait` 有如下的结构，通过 layer 函数来描述这件事，包装之后的 Service 是什么类型，已经定义在关联类型中了

```rust
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```



用上面的 Layer trait，来实现一个 Constant 的粘合层，所谓 Constant，意味着接收一个 Service，原封不动返回这个 Service。显然，返回的 Service 类型和 inner Service 相同，因此，在关联类型中，直接声明 `type Service = S;` 就好了。

```rust
struct Constant;

impl<S> Layer<S> for Constant{
    type Service = S;

    fn layer(&self, inner: S) -> Self::Service {
        inner
    }
}

```



### 3.3 换个方式理解 Layer



Layer 这个东西很神奇，它只是规定了如何去做，具体生效，还需要传入一个目标 Service 才行。



可以换一个方式理解，Layer 相当于**骨架**，是一副干巴巴的躯壳，本身不具有对外服务的能力，只有当我们传入一个核心的函数，才会**注入血肉**，得到一个中间件加持的究极无敌 Service。（或者理解为 Java 的 Factory 和 Bean 的关系）



同时可以非常方便地引入 Builder 建造者模式，通过 ServiceBuilder，我们将 Layer **按顺序**叠加，最终传入**血肉**，也就是核心的 service

```rust
let wrapped_svc= ServiceBuilder::new()
                  .buffer(100)
                  .concurrency_limit(10)
                  .layer(l4)
                  .layer(l5)
                  .service(svc);
```



通过 ServiceBuilder 操作，我们就完成了对于 svc 的中间件包装，之后的使用就很简单了：

1. poll_ready 等待服务准备好
2. 传入 request，调用 `wrapped_svc.call(request)` 来获取一个 Future
3. 将该 Future 交给 Tokio 处理
4. 循环 1-3 



## 4. Tower 和应用程序



已经解释完了 Tower 的两个核心 trait：Service 和 Layer，之后的章节中，会详细介绍各个关键中间件的使用方式，以及如何利用 Tower 的抽象，来搭建一个应用。

不知道大家注意到没有，Tower 自始至终，都在描述 `async fn handler(req)->resp` 这件事，但是没有提到 req 是怎么来的，相信聪明的你已经想到了，Tokio，或者更加准确的说，还需要一个连接器。

在实际应用中，需要在建立连接之后，才能获取到 request 内容，才能启动整个中间件链条，完成 `Ping-Pong` 的任务。大致逻辑长下面这样：

```rust
let wrapped_svc= ServiceBuilder::new()
                  .buffer(100)
                  .concurrency_limit(10)
                  .layer(l4)
                  .layer(l5)
                  .service(svc);

loop{
  
  let req = make_connection();
  let s = wrapped_svc.clone();
  
  /*
  	todo!(handle connection,get the request)
  */
  
  tokio::spawn(async move {
    	// s has been moved into this closure
      let f = s.ready().await.call(req);
			f.await;
  });
}
```

