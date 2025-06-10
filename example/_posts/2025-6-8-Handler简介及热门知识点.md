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
# Handler简介及热门知识点
Android的Handler机制是Android异步通信的核心机制之一，它主要用于解决在子线程中进行耗时操作后，需要更新UI的问题。由于Android的UI操作是非线程安全的，只能在主线程（UI线程）中进行，因此Handler机制提供了一种安全地从子线程向主线程发送消息并处理消息的方式。
## 核心组件
Handler机制主要由以下四个核心组件组成：

1.  **Message（消息）**:
    * Message是Handler机制中传递的数据单元，可以携带少量数据。
    * 它包含了消息的类型（`what`）、优先级（`arg1`、`arg2`）、对象数据（`obj`）以及一个Bundle数据（`data`）等信息。
    * Message可以由`Handler`发送到`MessageQueue`中。
    * 通常，我们通过`Message.obtain()`或`Handler.obtainMessage()`来获取一个Message对象，以避免频繁创建Message对象造成的性能损耗。

2.  **MessageQueue（消息队列）**:
    * MessageQueue是一个存储Message的队列，它采用**单链表**的数据结构来存储消息。
    * 它负责管理所有由Handler发送的消息，包括插入消息和按照一定的规则（通常是消息的发送时间）取出消息。
    * 每个Looper都对应一个MessageQueue，它会不断地从MessageQueue中取出消息。
    * MessageQueue是一个“先进先出”的队列，但是它也支持延迟消息的插入和取出。

3.  **Looper（消息循环器）**:
    * Looper是一个线程内部的循环器，它负责不断地从`MessageQueue`中取出消息，并将其分发给对应的`Handler`进行处理。
    * 每个线程最多只能有一个Looper。
    * **主线程（UI线程）默认拥有一个Looper**，系统会自动为它创建和启动。
    * 在子线程中，如果需要使用Handler机制，则必须手动为该线程创建并启动Looper，通常通过`Looper.prepare()`和`Looper.loop()`方法实现。
    * `Looper.loop()`方法是一个阻塞的循环，会一直从`MessageQueue`中取消息，直到`MessageQueue`中没有消息或者Looper被退出（通过`Looper.quit()`或`Looper.quitSafely()`）。

4.  **Handler（消息处理器）**:
    * Handler是Looper和MessageQueue的接口。
    * 它主要有两个作用：
        1.  **发送消息**: 可以通过`handler.sendMessage()`、`handler.post()`等方法将Message或Runnable发送到其关联的MessageQueue中。
        2.  **处理消息**: 当Looper从MessageQueue中取出消息后，会将消息回调给对应的Handler的`handleMessage()`方法进行处理。
    * Handler在哪个线程创建，默认就和哪个线程的Looper绑定。因此，如果Handler在主线程创建，那么它发送和处理的消息都会在主线程中执行。

## 工作原理
Handler机制的工作流程可以概括为以下步骤：

1.  **在主线程或子线程中创建Handler**:
    * 当Handler被创建时，它会自动关联当前线程的Looper（如果当前线程有Looper的话）。如果当前线程没有Looper，并且在子线程中直接创建Handler，会抛出异常。
    * 通常在主线程中创建一个Handler，这样这个Handler处理的消息都会在主线程中执行，方便更新UI。
    * 在子线程中，如果需要创建Handler，则需要先调用`Looper.prepare()`为当前线程准备一个Looper，然后创建Handler，最后调用`Looper.loop()`启动消息循环。

2.  **子线程发送消息**:
    * 当子线程需要向主线程发送消息时（例如，后台任务完成，需要更新UI），它可以通过主线程创建的Handler对象，调用`sendMessage()`或`post()`等方法，将一个Message或Runnable发送到Handler关联的MessageQueue中。

3.  **MessageQueue接收并存储消息**:
    * Handler发送的消息会被放入到MessageQueue中，按照其发送的时间（或延迟时间）进行排序。

4.  **Looper循环取出消息**:
    * Looper会不断地从MessageQueue中取出消息。
    * 如果MessageQueue中没有消息，Looper会阻塞等待。
    * 如果MessageQueue中有消息，Looper会取出最先到达（或最先到期）的消息。

5.  **Handler处理消息**:
    * Looper取出消息后，会将消息分发给对应的Handler的`dispatchMessage()`方法。
    * `dispatchMessage()`方法会根据Message的类型，最终调用`handleMessage()`方法或执行Post的Runnable。
    * 由于Handler是在主线程创建的，所以`handleMessage()`方法或执行的Runnable代码会在主线程中执行，从而可以安全地进行UI操作。

Handler机制主要用于以下场景：

* **子线程更新UI**: 这是Handler最主要的用途。在子线程中执行耗时操作（如网络请求、数据库操作），完成后通过Handler将结果发送到主线程，由主线程更新UI。
* **线程间通信**: 不同的线程之间可以通过Handler发送消息进行通信。
* **延迟执行任务**: 使用`postDelayed()`或`sendMessageDelayed()`可以实现延迟执行某个任务。
* **定时执行任务**: 结合`postDelayed()`或`sendMessageDelayed()`，可以在`handleMessage()`中再次发送延迟消息，实现定时任务。

## 热门知识点
### 安卓的 Handler 的消息处理流程是怎样的？
**1. 消息创建与发送**
- **创建消息**：  
  通过 `Handler.obtainMessage()` 或直接实例化 `Message` 对象（推荐复用消息池）。
- **发送消息**：  
  - `handler.sendMessage(msg)`：发送即时消息。
  - `handler.post(Runnable)`：将 `Runnable` 封装为 `Message`。
  - `sendMessageDelayed()` 或 `postDelayed()`：发送延迟消息。

---

**2. 消息入队（MessageQueue）**
- **消息队列**：  
  每个线程的 `Looper` 持有一个 `MessageQueue`，负责以时间顺序维护消息链表。
- **插入逻辑**：  
  根据 `when`（触发时间）将消息插入队列，如果是延迟消息，`when = SystemClock.uptimeMillis() + delayTime`。

---

**3. 消息循环（Looper）**
- **Looper.prepare()**：  
  初始化线程的 `Looper` 和 `MessageQueue`（主线程已默认初始化）。
- **Looper.loop()**：  
  无限循环从 `MessageQueue` 中取消息：
  ```java
  while (true) {
      Message msg = queue.next(); // 可能阻塞
      if (msg == null) return;    // 无消息时退出循环
      msg.target.dispatchMessage(msg); // 分发消息
  }
  ```
  - **阻塞时机**：队列为空或下一个消息未到触发时间（通过 `epoll` 机制节省CPU）。
  - **epoll机制简介**
    - 在 Android 平台中，epoll 是 Linux 内核提供的一种高效的 I/O 事件通知机制，被广泛应用于消息循环（如 Looper）和跨进程通信（如 Binder）等核心场景。它的作用是监控多个文件描述符（FD）的状态变化（如可读、可写、错误等），避免轮询带来的 CPU 资源浪费。
    - MessageQueue 使用 epoll 监控一个管道（pipe）的读端 FD。当队列为空时，Looper 线程阻塞在 epoll_wait() 上；当有新消息加入时，通过写管道触发 EPOLLIN 事件唤醒线程。若有延迟消息（如 postDelayed），epoll_wait 的超时时间会设置为最近消息的触发时间，到期后自动唤醒。

---

**4. 消息分发与处理（Handler）**
- **dispatchMessage()**：  
  ```java
  public void dispatchMessage(Message msg) {
      if (msg.callback != null) {
          msg.callback.run(); // 处理 Runnable 消息
      } else if (mCallback != null) {
          if (mCallback.handleMessage(msg)) return;
      }
      handleMessage(msg); // 子类重写处理逻辑
  }
  ```
  - 优先级：`Runnable` > `Handler.Callback` > `Handler.handleMessage()`。

---

**5. 消息回收**
- 处理完成后，`Message` 会被回收至全局池（通过 `recycleUnchecked()`），避免重复创建对象。

### MessageQueue消息排序机制是怎样的？
MessageQueue消息排序机制是通过时间戳（`when`）来实现的。

当调用handler.sendMessage()或handler.post()时：
* 消息会被赋予一个when时间戳（当前时间 + delay时间）
* 消息按照when值插入到队列中的正确位置
* 队列始终保持按when从小到大排序

**消息类型与优先级**

虽然所有消息都按时间排序，但有一些特殊情况：
* 同步屏障消息：优先级最高，会阻塞普通消息，只允许异步消息通过
* 异步消息：当存在同步屏障时，异步消息优先处理
* 普通消息：按时间顺序处理

这种基于时间戳的排序机制使得Android能够高效地处理大量定时消息，同时保证了消息执行的时序正确性。

### Handler 的 post(Runnable) 和 sendEmptyMessage 有什么区别？
* `post(Runnable)`：将 Runnable 封装为 Message 并发送。直接通过 Runnable 定义任务逻辑，代码更直观。不需要手动创建 Message 对象，内部会自动将 Runnable 封装为 Message（通过 Message.callback 字段存储）。适合简单的、无需传递数据的异步任务。
* `sendEmptyMessage(int)`：发送一个空消息，不携带任何数据。需要重写 Handler.handleMessage(Message msg) 方法，根据 msg.what 处理不同逻辑。适合需要区分多种消息类型或需要传递数据的场景（可通过 Message.arg1、Message.obj 等字段携带数据）。使用上更灵活，但代码稍显繁琐。

### Handler 导致的内存泄漏原因是什么？如何避免？
主要由于 Handler 持有外部类的隐式引用，导致 Activity/Fragment 无法被及时回收。

默认情况下，Handler 作为非静态内部类（或匿名内部类）会隐式持有外部类（如 Activity）的引用。例如：

```java
// 匿名内部类 Handler（危险！）
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        // 隐式持有外部 Activity 的引用
    }
};
```

此时，Handler 的实例会持有外部 Activity 的引用。如果 Handler 发送了延迟消息（如 postDelayed()），而消息尚未执行时 Activity 被销毁，Handler 会继续持有 Activity 的引用（因为消息队列的 Message 持有 Handler 的引用），导致 Activity 无法被 GC 回收。

解决：

**将 Handler 定义为静态内部类，并通过弱引用持有 Activity 的引用**

```java
private static class SafeHandler extends Handler {
    private final WeakReference<Activity> mActivityRef;

    SafeHandler(Activity activity) {
        mActivityRef = new WeakReference<>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        Activity activity = mActivityRef.get();
        if (activity == null || activity.isFinishing()) {
            return; // Activity 已销毁，不处理消息
        }
        // 正常处理逻辑
    }
}

// 在 Activity 中使用
private SafeHandler mHandler = new SafeHandler(this);
```

**在 onDestroy() 中移除所有未处理的 Message 和 Runnable**

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    mHandler.removeCallbacksAndMessages(null); // 清除所有消息
}
```

### Handler 的同步屏障(SyncBarrier)机制是什么？
Android Handler 的同步屏障（SyncBarrier）机制是 Android 消息处理机制中一个比较高级和不常用的特性，它主要用于**提高特定消息的处理优先级**。

同步屏障并不是一个常规的消息，它本身并不携带任何执行的任务。它的作用更像一个“路障”或“优先级指示器”。当它被插入到消息队列中时，会阻止所有**普通（非异步）消息**在它之前被处理。只有那些被标记为**异步（asynchronous）**的消息才能越过屏障，优先被处理。

同步屏障机制主要用于解决一些性能敏感或需要高优先级的场景，例如：

* **UI 动画：** 在进行复杂的 UI 动画时，你可能希望动画相关的消息（如帧更新）能够立即被处理，而不会被其他耗时的普通消息阻塞。这时可以将动画消息标记为异步，并插入同步屏障，确保动画流畅。
* **输入事件处理：** 为了提供更好的用户体验，输入事件（如触摸、点击）通常需要被立即响应。如果这些事件的消息被其他普通消息延迟，可能会导致界面卡顿。
* **低延迟任务：** 某些对延迟有严格要求的任务，可以使用同步屏障来保证其及时执行。

典型应用：
在界面进行绘制任务时，系统会插入同步屏障，确保绘制任务能够立即执行，避免界面卡顿。

requestLayout() 方法：

```java
@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
         ...
         // 注意此处会往主线程的MessageQueue消息队列中添加同步栏删，因为系统绘制消息属于异步消息，需要更高优先级的处理
         mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
         // 通过Choreographer往主线程消息队列添加CALLBACK_TRAVERSAL绘制类型的待执行消息，用于触发后续UI线程真正实现绘制动作
         mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
         ...
     }
}
```

### 如何在子线程中使用Handler？
Android 提供了 HandlerThread 类，封装了子线程的 Looper 管理，简化代码:

```java
// 1. 创建 HandlerThread
HandlerThread handlerThread = new HandlerThread("WorkerThread");
handlerThread.start();

// 2. 获取子线程的 Handler
Handler workerHandler = new Handler(handlerThread.getLooper()) {
    @Override
    public void handleMessage(Message msg) {
        // 处理消息
    }
};

// 3. 主线程发送消息
workerHandler.sendEmptyMessage(1);

// 4. 退出时释放资源
handlerThread.quitSafely();
```

注意事项
* 子线程的 Handler 必须在子线程创建，如果直接在主线程创建 Handler(workerThread.getLooper())，虽然可以发送消息，但 handleMessage() 仍会在子线程执行。
* 避免 Looper 未启动时发送消息，确保子线程的 Looper 已初始化（如通过 CountDownLatch 同步）。
* 及时退出 Looper，防止子线程长期运行导致内存泄漏。

### ​​Handler 在多线程环境下如何保证线程安全？
MessageQueue 是线程安全的，内部通过 synchronized 和 native 层的锁机制保证入队（enqueueMessage）和出队（next）的原子性。每个 Looper 绑定到一个特定线程，Handler 处理消息时始终在该线程执行，天然避免多线程并发问题。

如果必须通过 Handler 修改共享数据，需在 handleMessage 中加锁：

```java
private final Object lock = new Object();
private int sharedData;

Handler handler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(Message msg) {
        synchronized (lock) {
            sharedData = msg.arg1;
        }
    }
};
```
