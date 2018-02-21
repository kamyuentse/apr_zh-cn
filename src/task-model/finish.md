# 整合执行器与事件循环

At this point, we've built a simple executor for running many tasks on a single
thread, and a simple event loop for dispatching timer events, again from a
single thread. Now let's plug them together to build an app that can support an
arbitrary number of tasks periodically "dinging", using only two OS threads.
我先, 我们已经做了一个简单的执行器, 以在单线程中执行多个任务, 以及一个简单的
事件循环以分发定时事件, 当然用得也是单线程. 现在, 我们将他们整合到一起来构建一个
app, 这个app能够支持任意数量的周期任务"dinging", 而只需要两个OS线程.

To do this, we'll create a task called `Periodic`:
为了达成这个目标, 我们需要创建一个`Periodic`任务:

```rust,no_run
struct Periodic {
    // a name for this task
    id: u64,

    // how often to "ding"
    period: Duration,

    // when the next "ding" is scheduled
    next: Instant,

    // a handle back to the timer event loop
    timer: Timer,
}

impl Periodic {
    fn new(id: u64, period: Duration, timer: Timer) -> Periodic {
        Periodic {
            id, period, timer, next: Instant::now() + period
        }
    }
}
```

The `period` field says how often the task should print a "ding" message. The
implementation is very straightforward; note that the task is intended to run
forever, continuously printing a message after each `period` has elapsed:
`period`字段告诉我们打印"ding"信息要有多频繁. 这个实现很直接: 告诉任务是要永远
执行, 并且持续地在每经过一个`period`时间后打印一次信息:

```rust
impl Task for Periodic {
    fn complete(&mut self, wake: &WakeHandle) -> Async<()> {
        // are we ready to ding yet?
        let now = Instant::now();
        if now >= self.next {
            self.next = now + self.period;
            println!("Task {} - ding", self.id);
        }

        // make sure we're registered to wake up at the next expected `ding`
        self.timer.register(self.next, wake);
        Async::WillWake
    }
}
```

And now, we hook it all together:
然后, 把以上的东西都放到一起:

```rust,no_run
fn main() {
    let timer = ToyTimer::new();
    let exec = ToyExec::new();

    for i in 1..10 {
        exec.spawn(Periodic::new(i, Duration::from_millis(i * 500), timer.clone()));
    }

    exec.run()
}
```

The program generates output like:
这个程序最后产生类似这样的输出:

```
Task 1 - ding
Task 2 - ding
Task 1 - ding
Task 3 - ding
Task 1 - ding
Task 4 - ding
Task 2 - ding
Task 1 - ding
Task 5 - ding
Task 1 - ding
Task 6 - ding
Task 2 - ding
Task 3 - ding
Task 1 - ding
Task 7 - ding
Task 1 - ding
...
```

Stepping back, what we've done here is a bit magical: the implementation of
`Task` for `Periodic` is written in a pretty straightforward way that says how a
*single task* should behave. But then we can interleave any number of such
tasks, using only two OS threads total! That's the power of asynchrony.
回过头看, 我们所做的一点魔法: `Periodic`的`Task`实现直接说明*单个任务*所应具有的
行为, 但之后我们把一堆任务交织(interleave)到仅仅两个OS线程! 这就是异步的力量!

## 练习: 多任务同时注册

The timer event loop contains an unfortunate explicit panic: "Attempted to add
to registrations for the same instant".
例子的定时器事件循环有个panic代码: "无法在同一时刻注册任务".

<!-- TODO: panic的翻译有待改进 -->
- Is it possible to encounter this panic in the above program?
- What would happen if we simply removed the panic?
- How can the code be improved to avoid this issue altogether?
- 可以在实例的程序中解决这个panic吗?
- 如果我们仅仅是移除了panic, 会发生什么?
- 如何改进代码来完全避免这个问题呢?

## 练习: 逐渐结束程序

Both the `Periodic` task and the `SingleThreadedExec` are designed to run
without ever stopping.
`Periodic`和`SingleThreadedExec`被设计成不会停止的.

- Modify `Periodic` so that each instance is set to ding only a fixed number of
  times, and then the task is shut down.
- Modify `SingleThreadedExec` to stop running when there are no more tasks.
- Test your solution!
- 修改`Periodic`, 使得每个实例能够被设置为只会ding固定此时, 然后对应的任务停止.
- 修改`SingleThreadedExec`, 使得当没有任务存在的时候, 他会停止执行.
- 测试你的方案!
