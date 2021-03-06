---
title: future explained(0) 
date: 2021-11-24 14:01:29
categories: [rust, async]
---

generator（生成器）是 Rust 中 async/await 的基础，state machine 又是生成器的基础。

和闭包类似，不同的编译器会给 generator 生成不同的语义。但有一点是约定俗成的，也就是生成器最大的特点：程序的执行流程可以在生成器和调用者之间来回切换。

当我们需要暂时从生成器中返回的时候，可以使用 yield 关键字，当调用者希望再次进入生成器的时候，就调用 resume 方法，这时程序就从上次 yield 返回的那个点之后继续执行，还能获取到上下文信息，就好像能够按下了暂停按钮，又按下了继续按钮，程序执行完全没有感知。

其实不仅仅是 Rust，LLVM 作为一个广泛使用的后端，也支持了 yield 语法（因此 c++ 最新版本也可以实现 stackless 协程了）。接下来我们就分别探究一下二者实现的思路，以求对于 async/await 有一个更加深入的认识。

## 1. LLVM 实现

首先来看 LLVM 支持的方式：通过 `generator + yield` 两个关键字，可以将原先的一个普通函数，改写为生成器。

```cpp
#include <stdio.h>

generator int foo()
{
    yield 1;
    yield 2;
    yield 3;
}

int main()
{
    foreach (int i in foo())
        printf("Value: %d\n", i);

    return 0;
}
```

我们希望的执行流是：

1. 下一次重新执行该 generator，可以从上一次 yield 点开始执行
2. 可以保存栈的上下文

可以想象，我们需要保存很多**状态**，比如必须要 yield 的位置，这样才能在 resume 的时候执行正确位置的代码。Where? What?

1. 我们肯定不会将状态保存在**函数栈帧**中，因为一旦函数返回，该栈帧中的所有局部变量都失效了。也就是说，必须要将上下文（context）保存在函数外的某个结构体内（static 局部变量也不行，因为如果重复调用就炸了）。

2. 接下来考虑，需要保存什么样的状态，才能保证对于执行流和上下文没有感知？
   - 为了从 yield 下一条指令开始执行，肯定需要保存该指令的地址；
   - 为了保证栈帧中的局部变量不会失效，一定要将其保存存在调用栈之外的某个结构中（其实就是闭包）

实际上 LLVM 正是这样做的，通过 context 保存下一次 resume 需要跳转执行的位置，以及调用栈中的局部变量。每次调用这个 generator，都需要从外部传入相应的 context 上下文变量，将某些局部变量的读写都映射为针对 context 中相应字段的读写。

```
Tips:
  上下文变量还需要一个特殊的 setup 函数来处理，每一个 generator 都唯一对应 context，从而保证状态的确定性。
```

来简单看一下 LLVM 生成的 IR，围绕着 context 这个最关键的**Generator 状态**展开，关键位置都做了标记。

```llvm
// 上下文（状态）
%foo_context = type {
    i8*,      ; 0: block (pc 指针位置)    
    i32       ; 1: value (result) 
}

// foo 初始化，必须在调用 foo 之前设置
define void @foo_setup(%foo_context* %context) nounwind {
    ; set up 'block'
    %1 = getelementptr %foo_context* %context, i32 0, i32 0
    store i8* blockaddress(@foo_yield, %.yield1), i8** %1

    ret void
}

; The boolean returned indicates if a result was available or not.
; Once no more results are available, the caller is expected to not call
; the iterator again.
define i1 @foo_yield(%foo_context* %context) nounwind {
    ; 将 context.state 中的值作为下标，跳转到对应的位置
    %1 = getelementptr %foo_context* %context, i32 0, i32 0
    %2 = load i8** %1
    indirectbr i8* %2, [ label %.yield1, label %.yield2, label %.yield3, label %.done ]

.yield1:
    ; 将返回值保存在 foo_context.result 中 
    %3 = getelementptr %foo_context* %context, i32 0, i32 1
    store i32 1, i32* %3

    ; 将 'block' 块的 index 保存在 foo_context.state 中 
    ; make 'block' point to next block to execute
    %4 = getelementptr %foo_context* %context, i32 0, i32 0
    store i8* blockaddress(@foo_yield, %.yield2), i8** %4

    ; 返回
    ret i1 1

.yield2:
    ; 将返回值保存在 foo_context.result 中 
    %5 = getelementptr %foo_context* %context, i32 0, i32 1
    store i32 2, i32* %5

    ; 将 'block' 块的 index 保存在 foo_context.state 中 
    %6 = getelementptr %foo_context* %context, i32 0, i32 0
    store i8* blockaddress(@foo_yield, %.yield3), i8** %6

    ; 返回
    ret i1 1

.yield3:
    ; 将返回值保存在 foo_context.result 中 
    %7 = getelementptr %foo_context* %context, i32 0, i32 1
    store i32 3, i32* %7

    ; 将 'block' 块的 index 保存在 foo_context.state 中 
    %8 = getelementptr %foo_context* %context, i32 0, i32 0
    store i8* blockaddress(@foo_yield, %.done), i8** %8

    ; 返回
    ret i1 1

    ; 结束，返回默认值 0
.done:
    ret i1 0
}

define void @main() nounwind {
    ; 为 generator 分配 context 空间
    %context = alloca %foo_context
    call void @foo_setup(%foo_context* %context)
    br label %.head

.head:
    ; 每次调用，都需要传入 context 参数
    ; foreach (int i in foo())
    %1 = call i1 @foo_yield(%foo_context* %context)

    ; 省略 ...

.tail:
    ret void
}
```

上面的 generator 中，实际上没有保存什么局部变量，但是如果 generator 中存在跨越 yield 的局部变量，就需要将这些内容保存到 context 的某个字段中，这里不过多演示。

## 2. Rust 实现

虽然 Rust 底层依赖于 LLVM，LLVM 也提供了 generator 的能力，但 Rust 单独实现了一套类似的 generator。实际上 generator 也不算太难，简单来说，等同于一个闭包，只不过这个闭包不是一次性执行完成，而是分为多个阶段，每一个阶段都会对应一个状态，状态之间会进行转移。That's all！

下面主要会借助 Generator 和 Yield，探讨一下**目前** async/await 实现的底层原理。

### 2.1 Yield & Generator

首先来看一个 yield 关键字的 demo。

由于迭代器是一个实验性质的特性，因此要打开 `generators,generator_trait` 这两个 feature。 在我们的代码中，可以通过 yield + 闭包来声明 Generator。

```rust
#![feature(generators, generator_trait)]

use std::ops::{Generator, GeneratorState};
use std::pin::Pin;

fn main() {
  let mut g = || {
    yield 1;
    return "foo"
  };

  assert_eq!(Pin::new(&mut g).resume(()), GeneratorState::Yielded(1));
  assert_eq!(Pin::new(&mut g).resume(()), GeneratorState::Complete("foo"));
}
```

之所以用闭包实现，按照 LLVM 的实现机制以及上面的一小段简介，不难推断出原因：没错，需要保存上下文状态信息！那么，现在最重要的问题就是，这个上下文究竟是怎么保存的？编译器在背后究竟做了什么**黑魔法**？

就像 `Future` 的核心是 `Future trait`，`Generator` 的核心是 `Generator trait`，我们先来看一下这几个 trait 的定义。

```rust
#[derive(Clone, Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
#[lang = "generator_state"]
pub enum GeneratorState<Y, R> {
    Yielded(Y),
    Complete(R),
}

#[lang = "generator"]
#[fundamental]
pub trait Generator<R = ()> {
    type Yield;
    type Return;
    fn resume(self: Pin<&mut Self>, arg: R) -> GeneratorState<Self::Yield, Self::Return>;
}
```

简单和 `Future trait` 进行一下对比:

1. Future只有一个类型参数 Output，表示最终返回的结果，Poll 其实就相当于 Option；Generator 有两个类型参数 Yield 和 Return，可以在返回零次或多次 Yield 后最终返回Return，GeneratorState 其实就相当于 Result;
2. Future 的 poll 中有一个额外参数 ctx，一般是用来注册回调的；Generator 的 resume 中有一个额外参数 arg，也就是每次进入的时候都可以传一个参数，默认为 `()`，我理解是赋予了这个 Generator 扩展的能力。

`Generator trait` 在 99% 的情况下，不需要我们手动实现，而是编译器帮助我们生成：

参照 LLVM 的实现机制，对程序做控制流分析，将整个 generator 视为一个状态机，根据 `yield, return` 这两个关键字划分为多个子代码块。在 LLVM 中，实际上是通过一个指针保存了每次 resume 需要跳转的位置，但有一个更加优雅的方式：**enum + match**。

用一个 enum 保存每一个代码块执行中的状态，每次 yield 或者 return 之前，会将这段子代码块中的所有指令**全部**执行完成，然后修改 enum 中保存的状态，也就是**状态转移**，以及上下文中的局部变量，也都被捕获进来。

每一次 resume，其实就是需要匹配 enum 中的状态，去执行对应部分的代码，最后修改状态，是状态转移，直到返回 Complete。

按照这种思路，可以将上面 main 函数的闭包，翻译为下面的结构。

```rust
#![feature(arbitrary_self_types, generators, generator_trait)]

use std::ops::{Generator, GeneratorState};
use std::pin::Pin;

fn main() {
    let ret = "foo";
    let mut generator = {
        enum __Generator {
            Start(&'static str),
            Yield1(&'static str),
            Done,
        }

        impl Generator for __Generator {
            type Yield = i32;
            type Return = &'static str;

            fn resume(mut self: Pin<&mut Self>, resume: ()) -> GeneratorState<i32, &'static str> {
                use std::mem;
                match mem::replace(&mut *self, __Generator::Done) {
                    __Generator::Start(s) => {
                        *self = __Generator::Yield1(s);
                        GeneratorState::Yielded(1)
                    }

                    __Generator::Yield1(s) => {
                        *self = __Generator::Done;
                        GeneratorState::Complete(s)
                    }

                    __Generator::Done => {
                        panic!("generator resumed after completion")
                    }
                }
            }
        }

        __Generator::Start(ret)
    };

    Pin::new(&mut generator).resume(());
    Pin::new(&mut generator).resume(());
}
```

`enum __Generator`   就是生成的匿名结构，编译器会为其实现  Generate trait，就是上面分析所说的状态机核心逻辑。

总的来说，就是通过 **闭包 + 状态机**，实现了整个 Generator。

有了 Generator 的基础，再去看 async/await 的实现，就会轻松不少。我们只需要去关心，究竟是如何将 async/await 转换成闭包，并且用 **状态匹配 + 状态转移** 的方式去驱动执行。

```
Tips:
  需要提示一点，generator 在 Rust 中仍然是实验性的功能，只不过目前的 async/await 底层是借助了这一点来实现，未来不保证会不会修  改为别的方案，因为自引用结构的确有一些 annoying
```

### 2.2 HIR & MIR

Rust 针对 async/await 的处理，主要分为 HIR 和 MIR 两个阶段。

```bash
Tips:
    Rust 面向 LLVM 编程，但是 rustc 编译器也做了非常多的优化，将其划分为 HIR，MIR 
    
    1. HIR. 全称是 "high-level (H) intermediate representation (IR)"，可以将其理解为 AST 的另一中表示方式，主要是将语法糖  做了转换。

    2. MIR. The "mid-level (M) intermediate representation (IR)"。HIR 的简化版本，将高级的表达形式，转换成更加低级的结   构。rust 中的生命周期在这里会被抹除。

    3. LIR. Rust 中相当于 LLVM-IR。"low-level (L) intermediate representation (IR)" 是一种非常接近机器码的表示方式，会  对 MIR 做进一步的精简。
```

#### 2.2.1 HIR

在 HIR 中，其实是针对 `async fn test(f: F) where F:Future { f.await }` 做了脱糖处理。

比如，在 HIR 的表达式解析模块中，会将 async 代码块构造为一个实现了 future trait 的 generator：

```rust
std::future::from_generator(static move? |_task_context| -> <ret_ty> {
    <body>
})
```

会将 `<expr>.await` desugar 变为一个 `loop + match`。如果返回 Pending，下一句将会执行 `yield`：

```rust
match <expr> {
    mut pinned => loop {
        match unsafe { ::std::future::Future::poll(
            <::std::pin::Pin>::new_unchecked(&mut pinned),
            ::std::future::get_context(task_context),
        ) } {
            ::std::task::Poll::Ready(result) => break result,
            ::std::task::Poll::Pending => {}
        }
        task_context = yield ();
    }
}
```

也就是说，会将 async 代码块翻译为 `from_generator` 生成的匿名结构体，这个结构体由编译器实现 Future trait。

你可能会好奇，GeneratorState 和 Poll 并**不兼容**啊，前者返回 GeneratorState，后者返回 Poll，这是为什么呢？别担心，在 GeFuture::poll 函数中，特别做了处理：

GenFuture 包装了一个 Generator 的时候，GenFuture::poll 其实会调用内部的 generator 的 resume 方法，返回的 Yield 对应 Pending，Complete 对应 Ready。

```rust
struct GenFuture<T: Generator<Yield = ()>>(T);

impl<T: Generator<Yield = ()>> !Unpin for GenFuture<T> {}

#[unstable(feature = "gen_future", issue = "50547")]
impl<T: Generator<Yield = ()>> Future for GenFuture<T> {
    type Output = T::Return;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // Safe because we're !Unpin + !Drop mapping to a ?Unpin value
        let gen = unsafe { Pin::map_unchecked_mut(self, |s| &mut s.0) };
        set_task_context(cx, || match gen.resume() {
            GeneratorState::Yielded(()) => Poll::Pending,
            GeneratorState::Complete(x) => Poll::Ready(x),
        })
    }
}

pub fn from_generator<T: Generator<Yield = ()>>(x: T) -> impl Future<Output = T::Return> {
    GenFuture(x)
}
```

这一部分源码在 [rust expr code](https://github.com/rust-lang/rust/blob/master/compiler/rustc_ast_lowering/src/expr.rs)，[GenFuture code](https://doc.bccnsoft.com/docs/rust-1.36.0-docs-html/src/std/future.rs.html#20-22) 中可以找到，

到这一步为止，HIR 对于 async 代码块的处理就完成了，接下来是最重要的 MIR 处理。

#### 2.2.2 MIR

MIR 中有一个 [transform](https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_transform/src/generator.rs) 模块专门做了这件事情。

其实主要的处理逻辑我们已经看过了，就是将 yield 转换成匿名 enum，并插入多个 yield point即可。这里并不过多展开。
