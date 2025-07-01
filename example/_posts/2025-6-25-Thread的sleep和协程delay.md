---
layout: post
description: > 
  本文介绍了Thread.sleep()和协程delay的区别，以及在哪些场景下使用。
image: 
  path: /assets/img/blog/blogs_jvm_thread_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_jvm_thread_cover.png
    960w:  /assets/img/blog/blogs_jvm_thread_cover.png
    480w:  /assets/img/blog/blogs_jvm_thread_cover.png
accent_image: /assets/img/blog/blogs_jvm_thread_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Thread.sleep和协程delay
## Thread.sleep()
`Thread.sleep()` 方法在 Java 中用于使当前正在执行的线程暂停指定的时间。它的实现涉及到 Java 虚拟机（JVM）以及操作系统层面的协作。

> “该方法一般用来告知 CPU 让出处理时间给 App 的其他线程或者其他 App 的线程。”

方法签名：

```java
public static native void sleep(long millis) throws InterruptedException;
```

> 可以看到sleep是一个native方法，底层是通过系统调用(System Call)来实现的。

调用 `Thread.sleep()` 时，有以下需要注意的点：

* `Thread.sleep()` 作用于当前正在执行的线程，会**暂停当前线程**。暂停意思就是当前线程会放弃 CPU 时间片，用 `sleep` 后，当前线程会放弃 CPU 的使用权，进入 **`TIMED_WAITING` 状态**。这意味着它不会参与 CPU 的调度，直到睡眠时间结束或者被中断。操作系统接收到休眠请求后，会将当前线程从可运行队列中移除，并将其放入一个等待队列，直到**指定的休眠时间**过去。**操作系统内核的调度器**负责管理这些等待的线程，并在时间到达后将其重新放回可运行队列。(放回队列不代表立即执行，操作系统内核的调度器会根据优先级等因素再次决定是否执行该线程)
* 需要注意，在slepp期间**不会释放锁**。这是一个非常重要的特性。`Thread.sleep()` **不会释放任何线程持有的监视器锁（monitor lock）**。如果一个线程在持有锁的情况下调用 `sleep`，其他试图获取该锁的线程将仍然被阻塞。这与 `Object.wait()` 方法不同，`wait()` 会释放锁。
* 另外，`Thread.sleep()` 还是一个可中断的方法。在休眠中的线程，其他线程可以调用它的 `interrupt()` 方法，`sleep()` 会立即抛出 `InterruptedException`。这允许你提前唤醒一个正在睡眠的线程。

### 经典使用场景
**GC优化**：在快速循环中插入sleep(0)
```java
while (true) {
    processBatch();
    Thread.sleep(0); // 让出CPU给GC线程
}
```

**降低CPU占用**：批处理任务降频
```java
for (int i = 0; i < 1_000_000; i++) {
    processItem();
    if (i % 100 == 0) {
        Thread.sleep(1); // 每100次休眠1ms
    }
}
```

**测试调试场景**
```java
Thread.sleep(5000);
viewmodel.getTestData();
```

### sleep时间精度问题
操作系统中，CPU竞争有很多种策略。Unix系统使用的是**时间片算法**，而Windows则属于**抢占式**的。

> 在时间片算法中，所有的进程排成一个队列。操作系统按照他们的顺序，给每个进程分配一段时间，即该进程允许运行的时间。如果在时间片结束时进程还在运行，则CPU将被剥夺并分配给另一个进程。如果进程在时间片结束前阻塞或结束，则CPU当即进行切换。调度程序所要做的就是维护一张就绪进程列表，当进程用完它的时间片后，它被移到队列的末尾。

> 所谓抢占式操作系统，就是说如果一个进程得到了 CPU 时间，除非它自己放弃使用 CPU ，否则将完全霸占 CPU 。在抢占式操作系统中，假设有若干进程，操作系统会根据他们的**优先级、饥饿时间（已经多长时间没有使用过 CPU 了）**，给他们算出一个总的优先级来。操作系统就会把 CPU 交给总优先级最高的这个进程。当进程执行完毕或者自己主动挂起后，操作系统就会重新计算一 次所有进程的总优先级，然后再挑一个优先级最高的把 CPU 控制权交给他。

**基于Linux的Android系统**则是结合了以上两种方案来分配cpu资源的。

**抢占式调度是核心机制**

> 内核可以在任何时刻强制中断当前正在运行的进程（任务），将CPU分配给更高优先级的任务。这是Linux内核（包括安卓）的核心调度特性。例如，当有来电（高优先级实时任务）时，系统会立即抢占当前正在运行的应用进程，优先处理通话相关任务。​中断驱动​​诸如硬件中断（如触摸事件、传感器数据）可以触发内核抢占当前任务，快速响应事件。

**​时间片轮转（辅助策略）​​**

> 每个任务被分配一个固定的时间片（CPU执行时间），时间片用完后，即使任务未完成，也会被强制挂起，调度器选择下一个任务运行。​​CFS调度器（Completely Fair Scheduler）​​：Linux内核默认的调度器（安卓也使用），通过虚拟运行时间（vruntime）动态调整任务的优先级和时间片分配，尽量保证公平性。安卓会根据任务类型（如交互式应用、后台服务）动态调整时间片长度，优先保障前台应用的响应速度。

`Thread.sleep()` 的**实际休眠时间不一定精确**。这是因为：

* **操作系统调度：** 操作系统调度是基于时间片和优先级进行的，即使休眠时间到了，线程也需要等待操作系统再次调度它。
* **系统负载：** 如果系统负载很高，CPU 资源紧张，线程可能无法在精确的时间点被唤醒。
* **时钟粒度：** 操作系统时钟中断的粒度（resolution）也影响了 `sleep` 的精度。例如，如果操作系统的时钟中断是 10 毫秒，那么即使你 `sleep(1)`，线程也可能至少休眠 10 毫秒。

如果在休眠期间发生中断，底层的系统调用会检测到这个中断信号，并返回一个错误码给 JVM。JVM 会捕获这个错误，并向上层 Java 代码抛出 `InterruptedException`。

#### Thread.sleep(1000)，1000ms后是否立即执行？
不一定，在未来的1000毫秒内，线程不再参与到CPU竞争。那么1000毫秒过去之后，这时候也许另外一个线程**正在使用CPU**，那么这时候操作系统是**不会重新分配CPU**的，直到那个线程挂起或结束；况且，即使这个时候恰巧轮到操作系统进行CPU 分配，那么当前线程也不一定就是**总优先级最高**的那个，CPU还是可能被其他线程抢占去。
#### Thread.sleep(0)，是否有用？
休眠0ms，这样的休眠有何意义？Thread.Sleep(0)的作用，就是**触发操作系统立刻重新进行一次CPU竞争，重新计算优先级**。竞争的结果也许是当前线程仍然获得CPU控制权，也许会换成别的线程获得CPU控制权。这也是我们在大循环里面经常会写一句Thread.sleep(0) ，因为这样就给了其他线程获得CPU控制权的权力，界面就不会假死在那里。
### SystemClock.sleep()
在 Android 平台中，SystemClock.sleep(long millis) 是一个用于让当前线程休眠指定毫秒数的方法。它位于 `android.os.SystemClock` 类中，是一个非常常用的工具方法，在 Android 中被推荐使用，尤其是在系统级开发或需要 **更稳定、更可预测** 的休眠行为时。

```java
    public static void sleep(long ms)
    {
        long start = uptimeMillis();
        long duration = ms;
        boolean interrupted = false;
        do {
            try {
                Thread.sleep(duration);
            }
            catch (InterruptedException e) {
                interrupted = true;
            }
            duration = start + ms - uptimeMillis();
        } while (duration > 0);

        if (interrupted) {
            // Important: we don't want to quietly eat an interrupt() event,
            // so we make sure to re-interrupt the thread so that the next
            // call to Thread.sleep() or Object.wait() will be interrupted.
            Thread.currentThread().interrupt();
        }
    }
```

从源码中可以看到，`SystemClock.sleep` 是一个**循环调用 `Thread.sleep`** 的方法，直到时间到了才返回。内部做了try catch，在中途如果线程被中断，循环会临时忽略这次打断。在预设的睡眠时间结束之后，也会重新设置中断标志位，以便上层调用者知道线程被中断了。

在使用时需要注意不要在主线程中进行过长的休眠，以避免影响用户体验和应用性能。根据具体需求，也可以考虑使用其他更灵活的调度机制来实现延迟或定时任务。
## Kotlin协程的delay
上述的 `Thread.sleep` 和Kotlin 协程的 `delay` 函数之间存在一个根本性的区别：`Thread.sleep` 会阻塞（block）当前线程，而 `delay` 函数是非阻塞的（non-blocking）。故delay一般**不称为阻塞，而是挂起**。
### delay函数的上层实现
`delay` 是一个 **挂起函数（suspending function）**，它只能在协程或另一个挂起函数中调用。它的签名如下：

```kotlin
suspend fun delay(timeMillis: Long)
```

当你调用 `delay` 时，在 Kotlin 协程层会发生以下情况：

1.  **协程暂停，而非线程阻塞：** `delay` 会暂停当前正在执行的协程，而不是它所运行的底层线程。
2.  **放弃 CPU，但线程可用于其他工作：** 当协程被 `delay` 挂起时，它会从执行它的线程上“脱离”下来，但该线程并没有被阻塞，它可以立即被用来执行其他等待中的协程或任务。
3.  **状态保存（Continuation）：** 在编译时，遇到挂起函数，Kotlin 编译器会生成特殊的代码来保存当前协程的执行上下文，包括局部变量、程序计数器（即下一条要执行的指令）等。这个保存下来的上下文被称为 **Continuation（续体）**。这个过程就是**CPS转换**。
4.  **调度器交互：** `delay` 函数会将这个 Continuation 对象以及指定的延迟时间传递给当前的 **`CoroutineDispatcher`（协程调度器）**。调度器负责安排协程的执行。
5.  **可取消性：** 与 `Thread.sleep` 类似，`delay` 也是可取消的。如果协程在 `delay` 期间被取消（例如，通过调用其 `Job.cancel()` 方法），`delay` 会立即抛出 `CancellationException`。

### delay的下层实现：JVM 与 `CoroutineDispatcher`
`delay` 的非阻塞特性主要由 `CoroutineDispatcher` 和底层的计时机制实现。

`CoroutineDispatcher` 是 Kotlin 协程框架中的核心组件，它决定了协程在哪个线程上执行以及如何调度。

当 `delay` 被调用时，调度器不会让线程调用 `Thread.sleep()` 来阻塞自己。相反，它会**注册一个定时任务**。

调度器通常会维护一个**任务队列**。当协程被遇到 **delay** 这个挂起函数挂起时，它的 Continuation 会被放入一个内部的延迟任务队列中，并关联一个唤醒时间。

**底层计时器：** 调度器会利用底层的非阻塞计时机制来处理这些延迟任务。这可能涉及到：
* **`ScheduledExecutorService`：** 在 JVM 上，`CoroutineDispatcher` 内部通常会使用 `java.util.concurrent.ScheduledExecutorService` 来安排在未来某个时间点执行任务。`ScheduledExecutorService` 维护一个线程池，可以异步地执行定时任务，而不会阻塞调用者线程。
* **系统事件循环：** 在某些特定环境下（例如 **Android平台** 的主线程），调度器可能会与操作系统或框架提供的事件循环（event loop）集成，将协程的恢复任务作为事件注册到事件队列中。

具体的挂起和恢复流程可以看之前写的更详细的分析：

[Kotlin协程挂起恢复源码解析](./2025-2-16-Kotlin协程挂起恢复源码解析.md)

