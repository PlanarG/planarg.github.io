---
title: Tacos lab1 directions
date: 2024-03-18 10:47:11
tags: [Operating System]
categories: [Operating System]
---

## 一点碎碎念

操作系统的 `lab` 相比于 ics 的 `lab` 难度明显上了一个档次，里面可能踩的坑实在是太多了！！而且对于新编写出的 `rust` 版本的操作系统，没有前人的足迹可以追寻，在写的时候也可能出现各种各样莫名其妙的问题。这篇文章权当一个小小的指北吧。我只会列出每个任务的设计思路。

首先我想列出在写这个 `lab` 之前，我们应该达成的几点“共识”（要不然这 `lab` 没法写了）

- 多线程中常用 `Mutex<>` 来确保对关键数据访问的独一性。它有一个优点是不需要 `Mutex` 本身可变就可以修改里面的数据，同时可以很安全地被多个线程访问。

- `std` 是没法用的，但是 `rust` 很贴心地帮我们在 `alloc` 里面实现了很多有用的工具，比如 `Vec, BinaryHeap, BTreeMap` 等等。需要注意的是 `rust` 中没有与 `multiset` 类似的数据结构。 

- 在这个 `lab` 中，我们只需要考虑单核 `cpu` 的情况。虽然整个系统看上去很多很多部分都是并行的，但实际上这些代码在某个时刻只会有一行正在被执行。

- 这个 `lab` 的核心是维护不同线程的切换，线程的切换涉及以下几种情况：
  1. 主动调用了 `schedule` 函数：将当前线程放回 `ready` 线程列表中，切换上下文，将执行权交给下一个线程。
  2. 一个线程被唤醒了，且它比当前正在运行的线程具有更高的优先级。
  3. 程序主动更改了一个线程的优先级（注意：这个线程一定是当前正在运行的线程！这个 `lab` 里面不需要实现更改其它线程的优先级），并且使得处于 `ready` 列表的线程中，存在一个线程的优先级更高。
  4. 一个线程请求了一个未被释放的锁。
  5. `time interrupt`。在内核中有一个计时器与一个切换阈值，如果距离上次切换的时间长于这个阈值，那么操作系统会触发一次 `time interrupt` 陷阱。如果不关闭 `time interrupt`，这个陷阱内会执行一次 `schedule`。因此如果要从系统层面上确保当前操作是原子的，需要关闭 `time interrupt`（关上这个之后甚至连 `tick` 都不会调用）。每个线程都有自己独立的 `time interrupt` 状态。在多核 `cpu` 上频繁地关 `interrupt` 不是一个好习惯，因为所有 `cpu` 都会停下，效率很低。不过我们只需要考虑单核的情况，因此该关还是关吧。
  
- 关于 `CondVar`：在使用互斥锁的场景下，如果我们手持一个锁，但同时又想要等待另外一个锁被释放，此时有两种想法：

  - 不释放当前手持的锁，等待被释放的锁。好处是不会产生数据竞争，但是此时任何其他需要第一个锁的线程都会被阻塞，这样效率不高。
  - 释放当前手持的锁，等待被释放的锁。这样做有一定风险：等待此线程持有的锁的那些线程有可能会在中间插入，从而导致数据竞争。

  `CondVar` 就是专门针对这样的场景设计的工具，核心部分由三个函数组成：

  1. `wait(cond, guard)` 这里 `cond` 是条件变量，`guard` 是锁，且必须处于持有状态。这个函数可以理解为“释放锁 `guard`，等待条件变量 `cond` 被触发，再重新获取锁 `guard`” 的原子过程，尽管其内部实现并不是原子的。
  2. `notify_one(cond)` 通知一个等待这个条件变量触发的线程。如果没有线程在等待，那么这次触发将会被丢失。
  3. `notify_all(cond)` 通知所有正在等待这个变量触发的线程。 

<!-- more -->

可以说从头到尾，这个 `lab` 有几个核心之处：

1. `src/thread.rs`，这个文件内实现了 `thread` 的所有接口。我修改了 `wake_up, sleep, get_priority, set_priority` 这些函数。我们需要实现让线程能非阻塞 `sleep`，以及能够设置当前线程的优先级。
2. `src/thread/manager.rs`，这个文件内实现了函数 `schedule`，可以说这是整个 `lab` 最为核心的函数：操作系统从 `ready` 列表中拿出优先级最高的线程，停止执行当前线程，切换上下文并移交执行权。
3. `src/thread/imp.rs`，这个文件内是线程各项属性的定义，为了实现 task2 的第二部分我们需要对线程本身的结构体做出很多修改。
4. 已经实现好的各种锁的代码。由于我们需要向线程切换以及唤醒线程中加入优先级的概念，因此所有涉及“唤醒等待线程”的地方都要考虑它们的优先级。

最后说一说调试方法。我还是最喜欢直接 `kprintln`。可以在 `schedule` 函数、设置优先级这些地方输出当前线程的信息，非常方便。由于 `rust` 的 `release` 编译非常非常慢，因此要调试测试数据的时候可以先加入参数 `--dry` 显示调用的命令，然后在根目录下手动执行这个命令。这样可以快很多。

比如，如果我想要调试测试点 `priority-preempt`，我就可以在 `tool` 文件夹里面输入命令 `cargo tt -c priority-preempt --dry`，它的输出是这样的


{% asset_img image-20240324194932498.png This is an image %}

在根目录下运行命令 `cargo run -r -F test-schedule -- -append priority-preempt` 就可以了。`-q` 参数可删可不删，我觉得看着编译信息有一种它正在工作的感觉。

## Alarm Clock

这个 `task` 的意思是需要让我们更改原有的 `sleep` 函数。这个函数默认的实现方式是每次被唤醒的时候就看看当前时刻有没有到目标时刻，如果没有到话就 `schedule`，让 `cpu` 执行其它线程。我们需要把它改为非阻塞的实现方式。

实现方式也很简单，可以发现整个操作系统有一个增加当前时刻的函数 `tick`，我们只需要让这个函数每次被调用的时候都检查一下当前正在等待的线程。可以用 `BTreeMap<i64, Vec<Arc<Thread>>>` 将线程唤醒时间与线程本身关联起来，每 `tick` 都检查一下当前正在 `sleep` 的线程列表。

## Priority Scheduling

### Basic Priority Scheduling

这里我们需要实现简单的线程优先级系统，为了确保我们对于优先级的理解正确，以下有几点共识：
- 操作系统在**任意时刻**正在运行的线程都一定具有最高的优先级。不存在一个有着更高优先级的线程位于等待列表中。
- 对于相同优先级的线程，在 `schedule` 中需要采用**先进先出**的策略：先被添加到 `ready` 列表中的线程应当被优先唤醒。
- 在信号量中优先唤醒优先级高的线程。但是对于相同优先级的线程，它们被唤醒的顺序可以是随机的。

为了让程序有更高的运行效率，你可能会想到用 `BinaryHeap` 实现取出优先级最高的线程，但是相信我，下一个任务你会把它改回 `Vec` 的（或者 `VecDeque`）

推荐在写这个部分之前先认真思考一下这几个问题：
1. 每个线程可能有哪些状态？
1. 什么时候线程有可能会调用 `set_priority`？需不需要考虑那些正在被 `Blocked` 的线程？
2. 什么时候最高优先级的线程会发生变化？

你会需要修改 `src/thread/scheduler/fcfs.rs` 文件，可以照着里面已经有的那个东西实现一个优先队列版本的。

值得一提的是我发现 `src/sync/sema.rs` 中对 `up` 函数的默认实现似乎有点问题：
```rust src/sync/sema.rs
/// V operation
pub fn up(&self) {
    let old = sbi::interrupt::set(false);
    let count = self.value.replace(self.value() + 1);

    // Check if we need to wake up a sleeping waiter
    if let Some(thread) = self.waiters.borrow_mut().pop_back() {
        assert_eq!(count, 0);

        thread::wake_up(thread.clone());
    }

    sbi::interrupt::set(old);
}
```

这里实际上是不能 `assert` 的：唤醒线程并不代表立即运行这个线程，而 `count` 只有在请求锁的线程开始运行的时候才会 $-1$。因此如果执行权没有被交给被唤醒的线程，进行相邻的两次 `V` 后 `count` 确实可能非 $0$。所以这里的 `assert` 需要删去。

### Priority Donation

聪明的你在写完上一个关于优先级的任务之后也许会有这样的疑问：诶？这样的规则是不是非常容易导致死锁？文档中描述了这样一种情况：假设现在有高优先级线程 `H`，中优先级线程 `M` 以及低优先级线程 `L`，如果 `L` 持有一个锁，而 `M` 正在运行，那么 `L` 这个线程无论如何也不会被启动（因为 `M` 的优先级高于它，根据优先级策略 `M` 应该被优先执行），如果 `H` 需要 `L` 的锁，那么它就完蛋了。

为了让这种情况不要发生，文档中提出了一种“线程捐赠”的机制：如果线程 `A` 要请求线程 `B` 的锁，它就会把自己的优先级“捐”给 `B`：`B` 此时的优先级应当变为 `A`，`B` 两者的较大值。我们将这个值称为“有效优先级”。如果 `B` 又开始请求线程 `C` 的锁，那么 `B` 就会把自己的有效优先级捐给 `C`，此时 `C` 的优先级就是 `A`，`B`，`C` 三者的最大值。

你仔细思考了一下，发现很难在信号量这种多对多的场景实现这样的线程捐赠机制：你怎么知道当前线程在 `P` 完之后或者 `V` 完之后会不会继续管这个信号量？即使它不会管，也有许许多多其它的线程有可能会 `V`。那这种情况是谁捐给谁呢？

答案是其实我们不需要管这种情况。只需要实现最简单的 `sleep` 内（不是第一个任务里面的那个 `sleep`）的优先级捐赠情况就行了，它相当于只能为 `0` 或者为 `1` 的信号量。`sleep` 内是一对多的（一个 `holder`，多个 `waiter`）。

如果把线程捐赠的网络画成一张图，首先它一定是一个 `DAG`，再思考一下会发现它一定是一棵树：每个线程只可能等待另外一个线程，因为一旦 `require` 失败它就会 `Block` 直到这个锁被释放。因此我们需要实现的东西就是维护这么一棵捐赠的树形结构：可能会连一条新的边，也有可能断开原有的边，每个点的有效优先级是子树内的最大值。如果你非常厉害，那么不妨来试试 `lct` 吧。

需要注意的是由于我们可能会请求一个正在 `sleep` 或者正在被阻塞的线程所持有的锁，所以那些没有运行的程序的优先级，乃至整条链上的优先级都可能会被改变，这也是上一个任务中不能使用 `BinaryHeap` 的原因：它是不支持修改的。

另一件令人迷惑的问题是，我们需要在即使只有 `Arc<Thread>` 的情况下也能对此线程内的依赖关系进行修改，这就要求我们给那些可能被修改的数据套上互斥锁（`RefCell` 是不行的，因为它不 `Sync`，多线程访问的话 `rust` 会觉得不安全），这是相当奇怪的一件事：在上锁的过程中会涉及优先级捐赠，但实现优先级捐赠的部分却需要上锁。实际上在涉及到更改捐赠树的操作时我们会关闭 `interrupt`，因此可以确保访问的唯一性，但是为了通过编译仍然需要在相关的数据结构外套上一层 `Mutex`。

最后聊一聊 `acquire` 和 `release` 内具体要干的事情。

{% asset_img  image-20240324210533293.png 好图%}