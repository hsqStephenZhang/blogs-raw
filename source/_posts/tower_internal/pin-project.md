---
title: pin projection
---

虽然在之前的文章中认识了 Pin，但是我们忽视了一个很大的问题：**Pin projection**

为什么这个问题很重要呢？

## 1. 什么是 Pin Projection

先来看一下 projection 到底是什么：当给出一个 Pin<&mut Self> 的时候，我们能否安全地获取到 &mut Field 呢？或者只能得到 Pin<&mut Field>。
通俗一点理解，就是给出一个结构体，映射到内部某一些字段上，应该是什么样的结构？。

我们可以引入更加具体的场景来进行探讨

### 1.1 Unpin or !Unpin

比如在下面的 Combinator 结构体内部保存了另一个 Future，问题来了，我们如何确保 `Combinator::inner` 函数中的 unsafe 代码正确性？

```rust
struct Combinator<F> {
    inner: F,
}

impl<F: Future> Combinator<F> {
    pub fn inner(self: Pin<&mut Self>) -> Pin<&mut F> {
        unsafe {
            let this = Pin::get_mut_unchecked(self);
            Pin::new_unchecked(&mut this.inner)
        }
    }
}

impl<F: Future> Future for Combinator<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut &Self>, ctx: &mut Context<'_>) -> Poll<Self::Output> {
        self.inner().poll(ctx)

        // 如果这样写，直接 GG
        // self.inner().poll(ctx); 
        // let this = self.as_mut();
        // mem::swap(&mut this.inner,&mut something_else());
        // self.inner().poll(ctx) // 💥
    }
}

```

上面这段代码完全没有问题，我们虽然可以获取到 inner，但还是会受到 Pin 的约束，是一个 `Pin<&mut F>` 的类型  
但是 unsafe 很多时候是和一些安全代码结合起来之后，才会导致 UB 行为。比如：

```rust
impl<F: Future> Unpin for Combinator<F> {} 
```

为一个结构体实现 Unpin 是安全的，但是在这里，就是彻彻底底的 UB 行为：
因为实现了 Unpin，所以可以通过 Pin<&mut Combinator> 获取到 &mut self。但假如 inner 是一个自引用结构的 Future，我们就能通过安全的方式，移动其内部结构 💥💥💥，导致原先正常的 unsafe code 变得 unsound

正确的实现是：

```rust
impl<F: Future + Unpin> Unpin for Combinator<F> {} 
```

### 1.2 Drop 函数

在 Drop 函数中也需要格外注意，甚至应该遵循下面的 Drop 模式：在真正的 Drop 之前，将 &mut self 封装为 Pin<&mut Self>，**并且确保在这一步不会移动内层的 Future**，然后再通过 inner_drop 函数真正 Drop，否则我们无法**约束 safe code 的行为**，完全有可能通过 &mut self 获取到 `&mut inner`, 直接移动 inner future，导致其析构函数失效。  
但这一点也是需要用户做出保证的，使用者也完全可以在调用 inner_drop 之前滥用 `&mut self` 赋予的权利，导致 UB

```rust
impl Drop for Type {
    fn drop(&mut self) {
        // `new_unchecked` is okay because we know this value is never used
        // again after being dropped.
        inner_drop(unsafe { Pin::new_unchecked(self)});
        fn inner_drop(this: Pin<&mut Type>) {
            // Actual drop code goes here.
        }
    }
}
```

## 2. pin-projection

如果要求开发人员对每一个需要 projection 的结构都写这些 boilerplate code，显然不符合人体工学，为了更加便捷地约束嵌入结构体内部的 Future，不让自引用结构失效，引入了 pin-project 这个大杀器。

### 2.1 使用方式

先来看一下 `pin_project!` 的用法，三部曲：
1. 需要进行 project 的结构体打上 `[pin_project]` 标签，需要进行 project 的字段打上 `#[pin]` 标签
2. 当获取了 Pin<&mut Self> 之后，可以调用 self.project() 方法，获取一个 Projection 结构，该结构对结构体的每一个字段都做了映射，打上 `#[pin]` 标记的字段获取到的是 Pin<&mut Field>，没打上标记的字段可以获取到 &mut Field
3. 如果需要对结构体实现 Drop trait，只能按照 pin_project 要求的规则来实现，必须在结构体标记上 PinnedDrop，并且实现 PinnedDrop Trait

```rust
use std::pin::Pin;

use pin_project::pin_project;

#[pin_project]
struct Struct0<T, U> {
    #[pin]
    pinned: T,
    unpinned: U,
}

impl<T, U> Struct0<T, U> {
    fn method(self: Pin<&mut Self>) {
        let this = self.project();
        let _pinned: Pin<&mut T> = this.pinned; // 获取到 Pin<&mut Field>
        let _unpinned: &mut U = this.unpinned; //  获取到 &mut Field
    }
}
```

如果想要为这种结构限制 Drop 的函数签名，还可以利用到 PinnedDrop 选项，用上它之后，我们只能在 PinnedDrop 函数中为结构体实现 Drop 逻辑，这个 trait 给出的是 `Pin<&mut Self>` 的签名，也就没有问题了。

```rust
use std::{fmt::Debug, pin::Pin};

use pin_project::{pin_project, pinned_drop};

#[pin_project(PinnedDrop)]
struct PrintOnDrop<T: Debug, U: Debug> {
    #[pin]
    pinned_field: T,
    unpin_field: U,
}

#[pinned_drop]
impl<T: Debug, U: Debug> PinnedDrop for PrintOnDrop<T, U> {
    fn drop(self: Pin<&mut Self>) {
        println!("Dropping pinned field: {:?}", self.pinned_field);
        println!("Dropping unpin field: {:?}", self.unpin_field);
    }
}

fn main() {
    let _x = PrintOnDrop { pinned_field: true, unpin_field: 40 };
}
```

### 2.2 正确性证明

为了证明 pin-project 的正确性，我们只需要确保其没有违背 projection 正确性的两点要求即可：
1. 只有内层所有 Future 是 Unpin，该结构才能是 Unpin（条件性 Unpin）；
2. Drop trait 的实现方法中，不会移动内部的自引用结构。

因为 pin-project 通过过程宏实现，不容易展开，这里拿另一个项目 pin-project-lite 进行说明，lite 版本是使用声明宏实现的，在 IDE 宏展开也更加轻松

对于上面的 Struct0 结构，实际上会生成下面的代码，这段代码的含义是，只有当 __Origin 结构是 Unpin 的时候，才会为 Struct0 打上 Unpin 标记，而 __Origin 是否 Unpin，只和我们打上 `#[pin]` 标签的 pinned 这个字段有关，也就确保了：只有当字段 pinned 的类型 T 是 Unpin 的时候，才会为 State0 这个结构体实现 Unpin trait

```rust

struct __Origin<'__pin, T, U>{
    __dummy_lifetime: ::pin_project_lite::__private::PhantomData<&'__pin ()>,
    pinned: T,
    unpinned: ::pin_project_lite::__private::AlwaysUnpin<U>,
}

impl<'__pin, T, U> ::pin_project_lite::__private::Unpin for Struct0<T, U>
    where __Origin<'__pin, T, U>: ::pin_project_lite::__private::Unpin {}
```


可能有同学会问，那么假如我们给 unpinned 字段赋值为一个 !Unpin 的结构，比如 aysnc {}，不就违反了这个约束吗？
确实如此，但我们之所以要使用 pin-project，完全是因为它带给了我们更好的编程体验，用来回避一些完全安全，放在以前需要手写 `unsafe { Pin::new_unchecked ...}` 的情形。  
试想一下，如果我们将 State0 的 unpinned 字段赋值为 async {}，虽然完全可行，但其实没有任何意义。当我们创建了一个 `impl Future<Output=()>`，并需要想要 Poll 这个 Future 时，仍然需要使用 Box::pin 或者 unsafe block，否则无法创建 Pin<&mut Future>；但如果是为 pinned 字段赋值为 `impl Future<Output=()>` 结构，通过 projection 只能获取到 `Pin<&mut impl Future<Output=()>>` 类型，Pin 封装的类型是 !Unpin，就能约束无法通过 Pin<&mut State0> 获取到 &mut T，**避免了对内部自引用结构的移动**

为了确保在 Drop 代码中，不会移动内部的自引用结构，pin-project-lite 做了一点小小的 trick，会为标记为 pin_project 的结构体实现 `MyStructMustNotImplDrop trait`，当用户自己实现 Drop trait 的时候，就会导致 `impl<T: Drop> MyStructMustNotImplDrop for T {}` 生效，从而和默认实现的发生冲突，编译不能通过，直接在根本上回避了 Drop 代码中获取到 &mut Self 的问题。
而如果用户想要实现 Drop 方法，也没有问题，通过实现 pin-project 提供的 PinnedDrop trait，就只能在 Drop 方法中获取到 Pin<&mut Self>，在 Self 不是 Unpin 的情形下，就避免了 safe code 中对于内部自引用结构的移动

```rust
struct MyStruct {}
trait MyStructMustNotImplDrop {}
impl<T: Drop> MyStructMustNotImplDrop for T {}
impl MyStructMustNotImplDrop for MyStruct {}
```

以上这些 pin-project 以及 pin-project-lite 的底层实现其实稍微有一些 tricky，但的确为我们保证了 projection Pin 结构的正确性，我们在使用的过程中，也只需要明白：
需要在自定义 Future(暂且称为 CustomFuture) 内部包含 Future ，并且在 CustomFuture::poll 函数中手动 poll 内部 Future 的时候，使用 pin-project 可以为我们**在保证正确性的前提下，简化大量的代码**。