# 一个玩具事件循环

Asynchronous programming is most often used for I/O, but there are many other
sources of events. In this section, we'll build a tiny *event loop* that allows
you to register tasks to be woken up at a specific time.
异步编程经常被用于I/O, 但其实有很多其他类型的事件源. 在这一小节, 我们将会构建
一个简单的*事件循环*, 它将允许你注册一些可以在将来被唤醒的任务.

To do this, we'll spin up a dedicated timer event thread, whose sole job is to
wake up tasks at appropriate times. Consumers of the timer just send this thread
a message saying when they'd like to be woken, and how to wake them:
为了完成这件事, 我们需要一个专门的定时器事件线程, 它的工作就是在正确的时间唤醒
任务. 这个定时器的消费者们只需要告诉定时器什么时候应该要唤醒它们, 以及如何唤醒: 

```rust,no_run
use std::collections::BTreeMap;
use std::sync::{Arc, mpsc};
use std::thread;
use std::time::{Duration, Instant};

/// A handle to a timer, used for registering wakeups
#[derive(Clone)]
struct ToyTimer {
    tx: mpsc::Sender<Registration>,
}

/// A wakeup request
struct Registration {
    at: Instant,
    wake: Arc<Wake>,
}

/// State for the worker thread that processes timer events
struct Worker {
    rx: mpsc::Receiver<Registration>,
    active: BTreeMap<Instant, Arc<Wake>>
}

impl ToyTimer {
    fn new() -> ToyTimer {
        let (tx, rx) = mpsc::channel();
        let worker = Worker { rx, active: BTreeMap::new() };
        thread::spawn(|| worker.work());
        ToyTimer { tx }
    }

    // Register a new wakeup with this timer
    fn register(&self, at: Instant, wake: Arc<Wake>) {
        self.tx.send(Registration { at, wake }).unwrap();
    }
}

```

The event loop lives in the `Worker::work` method. The basic approach is
extremely simple: we keep a `BTreeMap` with the currently-registered wakeups,
and use a channel to make new registrations. The one bit of cleverness: if it's
not yet time to fire off any wakeups, but we do have some scheduled, we can use
`recv_timeout` on the channel to wait for *either* a new registration to come
in, *or* for it to be time to fire the first existing one off:
这个时间循环在`Worker::work`方法里面. 基本的目标特简单: 我们维护一个记录目前
已注册的唤醒句柄(wakeups)的`BTreeMap`, 并且用一个通道(channel)来进行注册. 这样做
的好处是:如果还没有到触发(fire)任何句柄的时刻, 但是我们已经有了一些注册好的句柄,
我们可以用通道上的`recv_timeout`方法来等待*要么*一个进来的新注册事件, *或者*等待
触发第一个已有句柄的时刻:

```rust,no_run
impl Worker {
    fn enroll(&mut self, item: Registration) {
        if self.active.insert(item.at, item.wake).is_some() {
            // this simple setup doesn't support multiple registrations for
            // the same instant; we'll revisit that in the next section.
            panic!("Attempted to add to registrations for the same instant")
        }
    }

    fn fire(&mut self, key: Instant) {
        self.active.remove(&key).unwrap().wake();
    }

    fn work(mut self) {
        loop {
            if let Some(first) = self.active.keys().next().cloned() {
                let now = Instant::now();
                if first <= now {
                    self.fire(first);
                } else {
                    // we're not ready to fire off `first` yet, so wait until we are
                    // (or until we get a new registration, which might be for an
                    // earlier time).
                    if let Ok(new_registration) = self.rx.recv_timeout(first - now) {
                        self.enroll(new_registration);
                    }
                }
            } else {
                // no existing registrations, so unconditionally block until
                // we receive one.
                let new_registration = self.rx.recv().unwrap();
                self.enroll(new_registration)
            }
        }
    }
}
```

That's it! This kind of "event loop" pattern, where a dedicated thread is
continually processing events and registrations (or else blocking until they are
available) is foundational to async programming. Fortunately, to *do* async
programming in Rust, you can use off-the-shelf event loops for events of
interest, as we'll see in the next chapter.
完成了! 这种"事件循环"模式, 一个专用线程在不停地处理事件与注册(或阻塞直到有
新事件触发或注册), 是异步编程的基础. 幸运的是, 在Rust里*实现*异步编程, 你可以用
现成的(off-the-shelf)事件循环来驱动你感兴趣的事件, 而我们将会在下一章中看到.

But before we go there, let's plug together the pieces we've built into a tiny,
working app.
但在那之前, 我们先把前面的执行器, 调度器整合成一个简单的app吧.
