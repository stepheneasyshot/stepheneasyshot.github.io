---
layout: post
description: > 
  本文介绍了 Android 系统触摸事件分发的流程
image: 
  path: /assets/img/blog/blogs_android_common_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_common_cover.png
    960w:  /assets/img/blog/blogs_android_common_cover.png
    480w:  /assets/img/blog/blogs_android_common_cover.png
accent_image: /assets/img/blog/blogs_android_common_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】Android事件分发流程
首先经由一个经典的ANR报错引入：

```
// 1. InputDispatcher 记录应用无响应
I/InputDispatcher: Application is not responding: Window{<Window Token> u0 com.your.package/com.your.package.YourActivity}.
    It has been 5001.0ms since event, 5001.0ms since wait started. 
    Reason: Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago. 
    Wait queue length: 1. Wait queue head age: 5001.0ms.

// 2. WindowManagerService 记录输入事件调度超时
I/WindowManager: Input event dispatching timed out sending to com.your.package/com.your.package.YourActivity.
    Reason: Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago.
```

`Application is not responding: Window{...}` 指出了是哪个应用程序窗口发生了无响应。触摸事件到达已经超过 5 秒（ANR 默认超时时间）。

`Reason: Waiting to send non-key event...` 这就是核心原因。 它表示 InputDispatcher 正试图发送一个后续的触摸事件（比如 ACTION_MOVE 或 ACTION_UP），但是前一个事件（通常是 ACTION_DOWN）发送到这个窗口后，窗口还没有返回结果（即没有完成 Activity.dispatchTouchEvent() 的调用）。在 ANR 触发的 5 秒总时长中，这个等待事件在队列头部停留的时间已经超过了 500 毫秒（这个数值是内部用于判断是否阻塞的阈值，不是总的 ANR 超时）。

## 事件传递到Window之前
这部分在APP冷启动，点击Launcher图标的阶段有所介绍。直接复制过来：

Android 系统是由事件驱动的，而 input 是最常见的事件之一，用户的点击、滑动、长按等操作，都属于 input 事件驱动，其中的核心就是 `InputReader` 和 `InputDispatcher` 。 `InputReader` 和 `InputDispatcher` 是跑在 SystemServer进程中的两个 native 循环线程，负责读取和分发 Input 事件。

* `InputReader` 负责从 `EventHub` 里面把 **Input事件** 读取出来，然后交给 `InputDispatcher` 进行**事件分发**；
* `InputDispatcher` 在拿到 `InputReader` 获取的事件之后，对事件进行包装后，寻找并分发到目标窗口;

`system_server` 的native线程 `InputReader` 读取到了一个触控事件。它会唤醒 InputDispatcher 去进行事件分发，先放入 `InboundQueue` 队列中，再去 **寻找处理事件的窗口** ，找到窗口后就会放入 `OutboundQueue` 队列，等待通过socket通信发送到 **launcher应用** 的窗口中，此时事件处于 `WaitQueue` 中，等待事件被处理，若5s内没有处理，就会向 `systemserver` 报ANR异常。

![input_event](/assets/img/blog/blogs_input_event.jpg)

Launcher进程接收到之后，通过 `enqueueInputEvent` 函数放入 **“aq”** 本地待处理队列中，唤醒 **UI线程** 的 `deliverInputEvent` 流程进行事件分发处理，具体交给界面window里的类型来处理。

从View布局树的根节点DecorView开始遍历整个View树上的每一个子View或ViewGroup界面进行事件的分发、拦截、处理的逻辑。

这次的触摸事件被消耗后，Launcher及时调用 `finishInputEvent` 结束应用的处理逻辑，再通过JNI调用到native层InputConsumer的 `sendFinishedSignal` 函数通知 `InputDispatcher` 事件处理完成，及时从 waitqueue 里移除待处理事件，避免ANR异常。

**整个处理流程是按照责任链的设计模式进行**

## 整体传递流程
应用程序在Activity创建之初，在 `setView` 方法中，就通过 WindowManager 发起Binder通信向 WMS 注册了自己的 Window 到系统进程中。后续有触摸事件时， `InputDispatcher` 会接收到来自 `InputReader` 的 `MotionEvent` 。InputDispatcher 会查询 WMS 持有的窗口列表，根据触摸事件的 X/Y 坐标，找出在触摸点下方的最顶层、可见且可接收输入的窗口。WMS 还会判断当前哪个窗口拥有输入焦点 (Input Focus)。对于非触摸事件（如按键），焦点窗口是首选目标。最终， `InputDispatcher` 确定了事件要发送到的目标窗口。

在此之前，会将这个触摸事件封装成一个 `MotionEvent`。

* **Window (PhoneWindow)**：
    `MotionEvent` 最终会被发送给这个触摸点所对应的**窗口**（`Window`）实例。在 Android 中，Activity 对应的 Window 实际是 `PhoneWindow` 的一个实例。`PhoneWindow` 的 `superDispatchTouchEvent()` 方法是事件进入 View 体系的第一站。
* **DecorView (根 ViewGroup)**：
    `PhoneWindow` 会将事件传递给其所持有的 **DecorView**。
    * **DecorView** 是一个 `FrameLayout`，它是 Activity 窗口的**根 View**。它包含了状态栏、标题栏（如果存在）以及 Activity 的实际内容区域。
    * DecorView 是一个 **ViewGroup**，它实现了 `dispatchTouchEvent()` 方法。
* **Activity 的 `dispatchTouchEvent()`**：
    DecorView 在分发事件时，会调用 **Activity** 的 `dispatchTouchEvent(MotionEvent ev)` 方法。
    * **这是 Activity 参与事件分发的最早入口点。** Activity 的 `dispatchTouchEvent()` 方法通常会调用 `getWindow().superDispatchTouchEvent(ev)`，最终会把事件重新传给 **DecorView** 的 `dispatchTouchEvent()`。
* **View/ViewGroup 树的向下分发：**
    * `DecorView`（作为根 ViewGroup）调用自己的 `dispatchTouchEvent()`，开始将事件**向下**分发给其子视图（即你通过 `setContentView()` 设置的布局）。
    * 事件沿着 View 树，从父 `ViewGroup` 经过 `onInterceptTouchEvent()` 检查，然后传给子 `View/ViewGroup` 的 `dispatchTouchEvent()`，直到找到一个处理该事件的 `View`。

### 总结

* 触摸事件首先被系统发送给对应的 **Window** (PhoneWindow)。
* Window 调用 **Activity 的 `dispatchTouchEvent()`**（Activity 获得第一次处理机会）。
* Activity 的 `dispatchTouchEvent()` 又将事件交给 **Window 的 DecorView**。
* **DecorView**（作为 View 树的根）开始将事件向下传递给内部的 **View/ViewGroup 控件**，完成真正的 View 树内部分发。

所以，Activity 和 Window 都是事件进入 View 树之前的重要环节，它们之间有交替的关系。

## 源码分析待补齐