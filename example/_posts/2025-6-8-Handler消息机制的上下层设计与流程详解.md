---
layout: post
description: > 
  本文介绍了Android中特有的Handler消息处理机制
image: 
  path: /assets/img/blog/blogs_handler_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_handler_cover.png
    960w:  /assets/img/blog/blogs_handler_cover.png
    480w:  /assets/img/blog/blogs_handler_cover.png
accent_image: /assets/img/blog/blogs_handler_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Handler消息机制的上下层设计与流程详解
## Linux体系中的eventfd
Linux系统中，应用程序到系统内核的体系架构：

![](/assets/img/blog/blogs_linux_app_arch.png)

## 经典问题点
### postDelay消息是如何实现的？
当你使用 `Handler.postDelayed(Runnable r, long delayMillis)` 或 `Handler.sendMessageDelayed(Message msg, long delayMillis)` 发送延时消息时，Handler 机制会利用底层的一些巧妙设计来确保消息在指定的时间后才被处理。其核心在于 **消息队列的排序** 和 **Looper 的休眠/唤醒机制**。

下面我们来深入了解一下 `postDelayed` 的底层实现原理：

1.  **消息的封装与时间戳**

    当你调用 `postDelayed` 时，Handler 会创建一个 `Message` 对象（如果是 `postDelayed(Runnable r, ...)`，Runnable 会被封装到 Message 的 `callback` 字段中）。这个 `Message` 对象会被赋予一个关键属性：**`when`**。

    `when` 字段表示的是消息应该被处理的 **绝对时间**，计算方式是：

    `when = SystemClock.uptimeMillis() + delayMillis`

    * `SystemClock.uptimeMillis()`：返回系统开机以来的毫秒数，不包括深度睡眠的时间。这是一个稳定的、适合计算延时的时钟源。
    * `delayMillis`：你指定的延时时长。

    所以，`when` 字段就存储了这条延时消息的“到期时间”。

2.  **消息入队与排序**

    `MessageQueue` 并不是一个简单的 FIFO 队列，它实际上是一个 **有序队列**，消息会根据它们的 `when` 值进行插入，确保队列中的消息始终按 `when` 值从小到大（即按到期时间从早到晚）的顺序排列。

    当一个延时消息被发送并准备入队时，`MessageQueue` 会遍历现有消息，将其插入到正确的位置，以保持队列的有序性。这意味着，到期时间最早的消息总是在队列的头部。

3.  **Looper 的休眠与唤醒**

    这是实现延时消息的关键部分。`Looper` 在它的无限循环中，会不断地调用 `MessageQueue.next()` 方法来获取下一个要处理的消息。

    * **计算下一次唤醒时间**: `MessageQueue.next()` 方法会检查队列头部的消息。
        * 如果队列头部有消息，并且该消息的 `when` 值已经小于或等于当前 `SystemClock.uptimeMillis()`（即消息已到期），那么 `next()` 方法会立即返回该消息，Looper 会立即处理它。
        * 如果队列头部有消息，但它的 `when` 值大于当前时间（即消息还未到期），`next()` 方法会计算一个 **`nextPollTimeoutMillis`**。这个超时时间就是当前时间到队列头部消息到期时间之间的时间差。
            `nextPollTimeoutMillis = 队列头部消息的when - SystemClock.uptimeMillis()`
            这个超时时间会告诉底层的阻塞机制，Looper 最多可以休眠多久。

    * **Native 层的阻塞**: 计算出 `nextPollTimeoutMillis` 后，`MessageQueue.next()` 会调用到其 Native 层的实现。在 Native 层，Looper 会利用 Linux 内核的 **`epoll_wait`** 或 **管道 (pipe)** 机制，传入这个 `nextPollTimeoutMillis` 作为超时参数。
        * **如果 `nextPollTimeoutMillis` 大于 0**: Looper 线程会进入阻塞状态，最长休眠 `nextPollTimeoutMillis` 毫秒。这意味着线程会暂停执行，不会消耗 CPU 资源，直到：
            * 指定的时间过去（消息到期）。
            * 有新的消息入队（新的消息可能到期时间更早，需要提前唤醒）。
            * 有其他文件描述符事件发生（例如，用户输入、网络数据到达等）。
        * **如果 `nextPollTimeoutMillis` 小于或等于 0**: 说明队列头部的消息已经到期或没有延时，Looper 会立即返回并处理消息，不会阻塞。

    * **唤醒与消息处理**:
        * **时间到期**: 当 Looper 休眠的时间达到 `nextPollTimeoutMillis` 后，它会自动被系统唤醒。唤醒后，它会再次调用 `MessageQueue.next()`，此时原先的延时消息应该已经到期，于是被取出并分发处理。
        * **新消息提前唤醒**: 如果在 Looper 休眠期间，有新的消息入队，并且这个新消息的 `when` 值比当前队列头部消息的 `when` 值更早（即新消息需要更早处理），或者 Looper 根本就没有休眠，那么 `MessageQueue` 会通过向管道写入数据的方式，立即 **唤醒** 正在休眠的 Looper 线程。Looper 线程被唤醒后，会重新计算下一次休眠时间，或者直接处理更早到期的消息。

---

#### 总结

`postDelayed` 延时消息的底层实现可以归纳为以下几点：

1.  **绝对时间戳**：消息携带一个 `when` 属性，表示其到期处理的绝对时间。
2.  **有序消息队列**：`MessageQueue` 内部维护一个按 `when` 值排序的队列，确保到期时间最早的消息总在队首。
3.  **Looper 的智能休眠**：`Looper` 会根据队首消息的到期时间来计算休眠时长，并通过 Native 层的 `epoll_wait` 或管道机制进入高效的阻塞状态。
4.  **及时唤醒机制**：无论是时间到期，还是有更早到期的新消息入队，Looper 都能被及时唤醒以处理相应的任务。

这种设计使得 Android 的延时消息机制既高效又准确，能够确保消息在指定时间后才被处理，同时避免了不必要的 CPU 消耗。