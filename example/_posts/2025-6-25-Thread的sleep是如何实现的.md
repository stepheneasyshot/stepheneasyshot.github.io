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
# Thread的sleep是如何实现的
-----

## `Thread.sleep` 在 Java 中的实现原理

`Thread.sleep()` 方法在 Java 中用于使当前正在执行的线程暂停指定的时间。它的实现涉及到 Java 虚拟机（JVM）以及操作系统层面的协作。

### 上层实现：Java 层面

从 Java 语言层面来看，`Thread.sleep(long millis)` 方法是一个 **`native` 方法**：

```java
public static native void sleep(long millis) throws InterruptedException;
```

`native` 关键字表示这个方法不是由 Java 代码实现的，而是由底层 C/C++ 代码实现的，通常在 JVM 内部或者通过 Java 本地接口（JNI）调用操作系统的功能。

当你在 Java 代码中调用 `Thread.sleep()` 时，会发生以下情况：

1.  **暂停当前线程：** `Thread.sleep()` 总是作用于当前正在执行的线程。
2.  **放弃 CPU 时间片：** 调用 `sleep` 后，当前线程会放弃 CPU 的使用权，进入 **`TIMED_WAITING` 状态**。这意味着它不会参与 CPU 的调度，直到睡眠时间结束或者被中断。
3.  **不释放锁：** 这是一个非常重要的特性。`Thread.sleep()` **不会释放任何线程持有的监视器锁（monitor lock）**。如果一个线程在持有锁的情况下调用 `sleep`，其他试图获取该锁的线程将仍然被阻塞。这与 `Object.wait()` 方法不同，`wait()` 会释放锁。
4.  **可中断性：** `Thread.sleep()` 是一个可中断的方法。如果在线程睡眠期间，其他线程调用了它的 `interrupt()` 方法，`sleep()` 会立即抛出 `InterruptedException`。这允许你提前唤醒一个正在睡眠的线程。

### 下层实现：JVM 与操作系统层面

`Thread.sleep()` 的底层实现深度依赖于 Java 虚拟机和其运行的操作系统。具体步骤如下：

1.  **JNI 调用：** 当 Java 代码调用 `Thread.sleep()` 这个 `native` 方法时，JVM 会通过 **Java 本地接口（JNI）** 调用到一个底层的 C/C++ 函数。
2.  **系统调用（System Call）：** 这个 C/C++ 函数会进一步调用操作系统提供的 **系统调用（System Call）** 来实现线程的暂停。不同的操作系统有不同的系统调用来实现类似的功能：
      * **在 Unix/Linux 系统中：** 通常会调用 `nanosleep()` 或 `usleep()` 等系统调用。这些函数允许线程休眠指定的时间（纳秒或微秒级别）。
      * **在 Windows 系统中：** 通常会调用 `Sleep()` 函数（Win32 API）。
3.  **操作系统调度：** 操作系统接收到休眠请求后，会将当前线程从可运行队列中移除，并将其放入一个等待队列，直到指定的休眠时间过去。操作系统内核的调度器负责管理这些等待的线程，并在时间到达后将其重新放回可运行队列。
4.  **时间精度：** `Thread.sleep()` 的实际休眠时间往往**不精确**。这是因为：
      * **操作系统调度：** 操作系统调度是基于时间片和优先级进行的，即使休眠时间到了，线程也需要等待操作系统再次调度它。
      * **系统负载：** 如果系统负载很高，CPU 资源紧张，线程可能无法在精确的时间点被唤醒。
      * **时钟粒度：** 操作系统时钟中断的粒度（resolution）也影响了 `sleep` 的精度。例如，如果操作系统的时钟中断是 10 毫秒，那么即使你 `sleep(1)`，线程也可能至少休眠 10 毫秒。
5.  **中断处理：** 如果在休眠期间发生中断，底层的系统调用会检测到这个中断信号，并返回一个错误码给 JVM。JVM 会捕获这个错误，并向上层 Java 代码抛出 `InterruptedException`。

### 总结

`Thread.sleep()` 的实现是一个分层的过程：

  * **上层（Java）：** 提供一个简洁的 `native` 方法接口，用于暂停当前线程，且不释放锁，并支持中断。
  * **下层（JVM + OS）：** 通过 JNI 调用操作系统提供的底层系统调用来实际暂停线程的执行，并由操作系统调度器管理线程的生命周期和唤醒。

理解 `Thread.sleep` 的底层机制有助于你在多线程编程中更好地使用它，尤其是在处理线程同步和中断时。