# 通过任务(task)掌握异步编程

Rust 通过*任务*提供异步性, 任务是:

- 能够单独运行的工作碎块(也就是可以并发处理), 比较像是OS线程.
- 轻量级的, 这样就不需要一整个OS线程. 取而代之的是OS线程可以整合任意数量的独立
任务, 也就是我们常说的"M:N 线程", 或"用户空间线程"

**关键的想法(idea)是,  如果一个任务可能[阻塞](task-model/intro.html)线程以等待
外部事件发生, 那么它马上返还控制权给当前执行线程.**

为了知道这些想法是如何实现的, 在这个章节的教程里, 我们会用`futures`包来构建一个
*玩具*版的任务模型和执行器. 在本章的最后, 我们会将这些玩具与更加巧妙的抽象方式
联系起来, 并将其应用到真实的系统当中.

我们从定义一个简单的任务trait开始. 一个任务包含一些(很可能正在进行)的工作, 你
可以告诉任务尝试去调用`complete`方法来完成工作:

```rust
/// An independent, non-blocking computation
trait ToyTask {
    /// Attempt to finish executing the task, returning `Async::WillWake`
    /// if the task needs to wait for an event before it can complete.
    fn complete(&mut self, wake: &WakeHandle) -> Async<()>;
}
```

该任务会在那时尽可能地完成工作, 但是它也可以遇到了需要等待事件发生, 例如套接字中
可用的数据. 这时候任务不应该阻塞线程, 而是返回`Async::WillWake`:

```rust
enum Async<T> {
    /// Work completed with a result of type `T`.
    Done(T),

    /// Work was blocked, and the task is set to be woken when ready
    /// to continue.
    WillWake,
}
```

任务*返回*而不是阻塞给了底层线程去做其他有用的工作的机会(例如调用*不同*任务的
`complete`方法). 但是我们怎么只要什么时候要再尝试调用原本任务的`complete`方法呢?

如果你回过头看`complete`方法, 你会发现有个参数`wake`. 这个参数是`Wake`trait的
trait对象:

```rust
trait Wake: Send + Sync + 'static {
    fn wake(&self);
}

type WakeHandle = Arc<Wake>;
```

所以**无论什么时候你执行了一个任务, 你同时也会给他一个用于以后唤醒该
任务的句柄 (handle).**如果这个任务因为他在等待套接字上的数据而无法被处理, 它可以
把`wake`句柄告诉给套接字, 那么当数据可用的时候, `wake`就会被调用.

不过这有点抽象. 我们再具体过一遍整个故事吧, 也就是在本章我们要构建:

- 一个可以在单个OS线程上跑任意数量的任务的简单*任务执行器*
- 一个能够基于定时事件唤醒任务的简单*定时时间循环*
- 一个会将上面两者整合在一起的简单的实例

只要你理解了这些机制, 你就建立了理解这本书其他内容的基础.
