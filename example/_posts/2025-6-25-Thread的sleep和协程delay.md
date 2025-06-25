---
layout: post
description: > 
  本文针对鸿蒙系统的基础特性，应用层开发所需基础知识，进行一个简单的调研，对比一下和其他平台的差异。
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
# Thread的sleep和协程delay
## Thread.sleep()
`Thread.sleep()` 方法在 Java 中用于使当前正在执行的线程暂停指定的时间。它的实现涉及到 Java 虚拟机（JVM）以及操作系统层面的协作。

> “该方法一般用来告知 CPU 让出处理时间给 App 的其他线程或者其他 App 的线程。”

方法签名：

```java
public static native void sleep(long millis) throws InterruptedException;
```

> 可以看到sleep是一个native方法，底层是通过系统调用来实现的。

调用 `Thread.sleep()` 时，有以下需要注意的点：

`Thread.sleep()` 作用于当前正在执行的线程，会暂停当前线程。

暂停意思就是当前线程会放弃 CPU 时间片，用 `sleep` 后，当前线程会放弃 CPU 的使用权，进入 **`TIMED_WAITING` 状态**。这意味着它不会参与 CPU 的调度，直到睡眠时间结束或者被中断。

需要注意，在slepp期间不会释放锁。这是一个非常重要的特性。`Thread.sleep()` **不会释放任何线程持有的监视器锁（monitor lock）**。如果一个线程在持有锁的情况下调用 `sleep`，其他试图获取该锁的线程将仍然被阻塞。这与 `Object.wait()` 方法不同，`wait()` 会释放锁。

另外，`Thread.sleep()` 还是一个可中断的方法。在休眠中的线程，其他线程可以调用它的 `interrupt()` 方法，`sleep()` 会立即抛出 `InterruptedException`。这允许你提前唤醒一个正在睡眠的线程。

### 下层实现：JVM 与操作系统层面
当 Java 代码调用 `Thread.sleep()` 这个 `native` 方法时，JVM 会通过 **Java 本地接口（JNI）** 调用到一个底层的 C/C++ 函数。

这个 C/C++ 函数会进一步调用操作系统提供的 **系统调用（System Call）** 来实现线程的暂停。

操作系统接收到休眠请求后，会将当前线程从可运行队列中移除，并将其放入一个等待队列，直到指定的休眠时间过去。操作系统内核的调度器负责管理这些等待的线程，并在时间到达后将其重新放回可运行队列。

**时间精度问题：** `Thread.sleep()` 的实际休眠时间往往**不精确**。这是因为：
  * **操作系统调度：** 操作系统调度是基于时间片和优先级进行的，即使休眠时间到了，线程也需要等待操作系统再次调度它。
  * **系统负载：** 如果系统负载很高，CPU 资源紧张，线程可能无法在精确的时间点被唤醒。
  * **时钟粒度：** 操作系统时钟中断的粒度（resolution）也影响了 `sleep` 的精度。例如，如果操作系统的时钟中断是 10 毫秒，那么即使你 `sleep(1)`，线程也可能至少休眠 10 毫秒。

如果在休眠期间发生中断，底层的系统调用会检测到这个中断信号，并返回一个错误码给 JVM。JVM 会捕获这个错误，并向上层 Java 代码抛出 `InterruptedException`。

## suspend fun delay
上述的 `Thread.sleep` 和 **Kotlin 协程的 `delay` 函数** 之间存在一个根本性的区别：`Thread.sleep` 会**阻塞（block）当前线程，而 `delay` 函数是非阻塞的（non-blocking）**。这个核心差异是协程之所以高效的关键。

### `delay` 函数的上层实现：Kotlin 协程层面

`delay` 是一个 **挂起函数（suspending function）**，它只能在协程或另一个挂起函数中调用。它的签名如下：

```kotlin
suspend fun delay(timeMillis: Long)
```

当你调用 `delay` 时，在 Kotlin 协程层会发生以下情况：

1.  **协程暂停，而非线程阻塞：** `delay` 会暂停当前正在执行的协程，而不是它所运行的底层线程。
2.  **放弃 CPU，但线程可用于其他工作：** 当协程被 `delay` 挂起时，它会从执行它的线程上“脱离”下来，但该线程并没有被阻塞，它可以立即被用来执行其他等待中的协程或任务。
3.  **状态保存（Continuation）：** 当协程被挂起时，Kotlin 编译器会生成特殊的代码来保存当前协程的执行上下文，包括局部变量、程序计数器（即下一条要执行的指令）等。这个保存下来的上下文被称为 **Continuation（续体）**。
4.  **调度器交互：** `delay` 函数会将这个 Continuation 对象以及指定的延迟时间传递给当前的 **`CoroutineDispatcher`（协程调度器）**。调度器负责安排协程的执行。
5.  **可取消性：** 与 `Thread.sleep` 类似，`delay` 也是可取消的。如果协程在 `delay` 期间被取消（例如，通过调用其 `Job.cancel()` 方法），`delay` 会立即抛出 `CancellationException`。

### `delay` 函数的下层实现：JVM 与 `CoroutineDispatcher`

`delay` 的非阻塞特性主要由 `CoroutineDispatcher` 和底层的计时机制实现：

1.  **`CoroutineDispatcher` 的作用：**

      * **调度器职责：** `CoroutineDispatcher` 是 Kotlin 协程框架中的核心组件，它决定了协程在哪个线程上执行以及如何调度。
      * **非阻塞等待：** 当 `delay` 被调用时，调度器不会让线程调用 `Thread.sleep()` 来阻塞自己。相反，它会注册一个定时任务。
      * **任务队列：** 调度器通常会维护一个任务队列。当协程被 `delay` 挂起时，它的 Continuation 会被放入一个内部的延迟任务队列中，并关联一个唤醒时间。
      * **底层计时器：** 调度器会利用底层的非阻塞计时机制来处理这些延迟任务。这可能涉及到：
          * **`ScheduledExecutorService`：** 在 JVM 上，`CoroutineDispatcher` 内部通常会使用 `java.util.concurrent.ScheduledExecutorService` 来安排在未来某个时间点执行任务。`ScheduledExecutorService` 维护一个线程池，可以异步地执行定时任务，而不会阻塞调用者线程。
          * **系统事件循环：** 在某些特定环境下（例如 Android 的主线程），调度器可能会与操作系统或框架提供的事件循环（event loop）集成，将协程的恢复任务作为事件注册到事件队列中。

2.  **唤醒与恢复：**

      * 当计时器到期时，`ScheduledExecutorService`（或类似机制）会执行预定的任务。
      * 这个任务会找到之前保存的协程 Continuation 对象。
      * 调度器会选择一个可用的线程（可能是之前挂起协程的同一个线程，也可能是线程池中的另一个线程），然后在这个线程上\*\*恢复（resume）\*\*该协程的执行，从 `delay` 调用点之后继续执行。

### `delay` 与 `Thread.sleep` 的关键对比

| 特性         | `Thread.sleep()`                               | `delay()` (Kotlin 协程)                                    |
| :----------- | :--------------------------------------------- | :--------------------------------------------------------- |
| **阻塞性质** | **阻塞**当前线程                               | **非阻塞**，只挂起当前协程                                 |
| **资源利用** | 线程被占用，无法执行其他任务，效率较低         | 线程可以被复用，执行其他协程，资源利用率高                 |
| **适用场景** | 线程级别的暂停，通常用于简单测试或少量并发场景 | 协程级别的暂停，适用于高并发、异步编程场景，如网络请求、UI 更新 |
| **锁释放** | **不释放**持有的监视器锁                       | **不涉及**线程锁，因为协程不直接持有线程锁                 |
| **可中断性** | 通过 `InterruptedException` 抛出              | 通过 `CancellationException` 抛出                          |
| **实现机制** | JVM 通过 JNI 调用操作系统系统调用              | 协程调度器通过 `ScheduledExecutorService` 等非阻塞计时机制管理 |

总而言之，`delay` 的实现充分利用了 Kotlin 协程的**非阻塞特性**和**结构化并发**。它将协程的暂停操作委托给了一个能够异步调度任务的组件（`CoroutineDispatcher`），从而避免了像 `Thread.sleep` 那样直接阻塞底层线程，显著提高了程序的并发性能和资源利用率。