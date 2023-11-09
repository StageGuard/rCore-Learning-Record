> 原文链接：https://github.com/verhovsky/books-futures-explained

## Rust Fat Pointer

Rust 的一些指针，例如 `&[T]` 或者 `&dyn Trait` 他们是胖指针，胖指针包含了指向对象的指针 `*mut T / *const T` 和它对应的 `vtables` 指针。虚表里包含了函数方法指针和 Drop 指针。

胖指针在内存的结构是这样的：

```rust
#[repr(C)]
struct FatPointer<'a> {
    data: &'a mut T,
    vtable: *const usize,
}
```

所以我们可以直接用 `mem:transmute` 来把 `FatPointer` 转换成真正的胖指针。

## generator 和 yield

在 Rust 的闭包调用 `yield value;` 会将当前闭包转换成 Generator，Generator 的调用者可以通过迭代器来持续接受 yield 发送的数据。

在 Generator 中，如果在 yield 前后发生了借用关系，可能会出现问题，yield 之前和之后的代码有不同的生命周期，导致借用失效。
但是如果是 static Generator 就可以用。


## Pin

首先，Pin 有 `Pin` struct 和 `Unpin` trait，`Pin` 是已经固定的对象的 wrapper。

`Pin` 是为了约束实现 `!UnPin` trait 的对象，pin 可以固定该对象在栈上的地址，尤其是有自引用的对象，pin 后可以保证自引用是始终有效的。

pin 一个栈对象需要 unsafe，而 pin 堆对象不需要（通过 `Box::pin`），因为分配在堆上的对象总是有稳定的内存地址。

## Rust Future

Future 是一种未来将要完成的操作，他的执行受到 Reactor 和 Executor 的调度，这就是异步执行的运行时。

Future 分为 `non-leaf-future` 和 `leaf-future`。

### `non-leaf-future` 和 `leaf-future` 的区分

`non-leaf-future` 是我们使用 `async {}` 创建的块，其中可以包含其他 `non-leaf-future` 和 `leaf-future`。

`leaf-future` 是运行时创建的实现了 `Future` trait 的对象，比如 `tokio::net::TcpStream`。

要想实现自己的 Future，首先需要 Executor 和 Reactor。

### Waker

对于阻塞线程的等待，我们需要一个工具来让线程正确阻塞并在其他线程唤醒。所以我们用 `Condvar` 来实现，封装成 Parker。

为了避免虚假唤醒，还需要把 `Condvar.wait` 放到 while 循环里。

我们实现自己的 Waker，需要构造一个 `RawWaker` 来允许自己定义 `wake`, `clone`, `drop`, `wake by ref` 行为，在 `wake` 和 `wake by ref` 里调用我们自己的 Parker.unpark 来解除 Executor 线程阻塞。

### Reactor

Reactor 负责接收事件并调度执行，也就是说我们的 `leaf-future` 需要在这里面执行。

Executor 通过 `Future.poll` 轮询结果时：

* 对于初次轮询，调用 `Reactor.register` 注册事件的执行，通过 mpsc channel 把要执行的事件发送到 Reactor 线程里。
* 对于轮询但是未唤醒，Executor 会继续等待，但是会更新一下 Waker。如果不更新，Reactor 可能会唤醒旧的 Waker，旧的 Waker 可能已经被 drop 或者失效了。
* 对于已唤醒，也就是 Reactor 定义的执行完成，Executor 会取出结果并返回。

在 Reactor 的线程中，接受 mpsc channel 的 Event 并执行，执行结束后再调用 `waker.wake` 来继续 Executor 轮询执行。

> mpsc 意思是 mutiple producer single consumer，多生产者，单一消费者