---
layout: post
description: > 
  本文介绍了 Android 领域重难点面试题精选
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
# 【Android进阶】Android视图加载与刷新
## 初始化
整体的冷启动流程在这篇文章有详细记录：

[【Android进阶】APP冷启动流程解析](./2022-12-21-【Android进阶】APP冷启动流程解析.md)

### Activity Window View初始加载
Activity、Window 和 View 这三者是构成安卓应用用户界面的核心。

这三者之间的层级和协作关系：
* **Activity (活动)**：是安卓应用的四大组件之一，是用户交互的直接入口。它本身并不负责视图的绘制，而是作为窗口（Window）的容器，并管理界面的生命周期（例如，创建、暂停、销毁等）。你可以把它想象成一个舞台的管理者或导演。
* **Window (窗口)**：每个 Activity 都包含一个 Window 对象，通常是 `PhoneWindow` 的实例。Window 才是真正代表一个“窗口”的概念。它负责承载界面元素，并将这些元素传递给 `WindowManager` 进行显示。你可以把它看作是舞台本身，所有的布景（View）都在这个舞台上。
* **View (视图)**：是所有 UI 控件（如 `Button`, `TextView`）的基类。它负责在屏幕上绘制具体的内容，并处理用户的触摸事件。一个 Window 内部通常包含一个复杂的 View 树（View Hierarchy），最顶层的 View 被称为 `DecorView`。你可以把 View 看作是舞台上的演员和布景。

总结来说，Activity 持有一个 Window，而 Window 持有一个 View 树（以 DecorView 为根）。Activity 负责逻辑控制和生命周期管理，Window 负责承载和管理视图，而 View 负责最终的绘制和事件处理。

![activity_window](/assets/img/blog/blogs_activity_window.png)

#### 初始化时机与关键周期事件
Activity 对象的初始化发生在 `ActivityThread` 中，通过 `performLaunchActivity()` 方法完成。在这个过程中，系统会通过反射调用 Activity 的无参构造函数来创建 Activity 实例。紧接着，系统会调用 Activity 的 `attach()` 方法，在这个方法内部，Activity 会创建一个 `PhoneWindow` 实例，从而将 Activity 和 Window 关联起来。

#### attach 时期
`attach()` 方法并不是 Activity 生命周期的一部分，开发者通常不需要也**不应该**重写它。它是框架在内部用于初始化 Activity 的一个关键步骤。

1.  **提供 Context**：`attach()` 的最重要职责是关联一个 `Context` 对象。在调用 `attach()` 之前，Activity 实例内部的 `mBase` (Context) 是 `null` 的。调用之后，Activity 才拥有了上下文，从而能够执行诸如 `getResources()`、`getSystemService()`、`getPackageName()` 等操作。没有 Context，Activity 几乎什么都做不了。
2.  **创建 Window**：在 `attach()` 方法内部，Activity 会创建一个 `PhoneWindow` 的实例，并赋值给成员变量 `mWindow`。这意味着在 `onCreate()` 被调用之前，Activity 已经有了一个关联的窗口对象。这就是为什么你可以在 `onCreate()` 里立即调用 `setContentView()` 的原因，因为 `setContentView()` 实际上是调用了 `mWindow.setContentView()`。
3.  **关联其他组件**：除了 Context 和 Window，`attach()` 还会将 Activity 与其他一些重要的系统组件关联起来，例如 `Application` 对象、`Instrumentation` 等。

```java
// Activity.java window的创建与初始化
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        ...) {
    // 创建PhoneWindow实例
    mWindow = new PhoneWindow(this, window);
    // 设置Window回调
    mWindow.setCallback(this);
    // 设置Window管理器
    mWindow.setWindowManager(...);
}
```

#### `onCreate` 时期
`onCreate()` 是我们熟知的 Activity 生命周期的第一个回调方法。它是开发者进行 Activity 初始化的主要入口。

最常见的操作就是调用 `setContentView(R.layout.activity_main)`，这一步依赖于在 `attach()` 中创建好的 `Window` 对象。

```java
// PhoneWindow的setContentView方法
public void setContentView(int layoutResID) {
    // 1. 检查是否有DecorView，没有则创建
    if (mContentParent == null) {
        installDecor();
    } else {
        // 如果已有内容视图，则移除
        mContentParent.removeAllViews();
    }
    
    // 2. 将布局inflate到mContentParent中
    mLayoutInflater.inflate(layoutResID, mContentParent);
    
    // 3. 通知Activity内容已改变
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

> mContentParent 实例通常是一个 `FrameLayout` 对象。用于容纳内容视图，这一步就是将 `R.layout.main` 对应的视图结构，作为子视图添加（addView()）到这个 mContentParent（即 FrameLayout）中。

#### `onResume()` 时期
Activity和窗口创建完成后， `ActivityThread` 调用 `handleResumeActivity` 来执行其 `onResume()` 流程，在 Activity 的 `onResume()` 周期回调之后，执行 `makeVisible()` 。

然后 `WindowManager` 执行 `addView` 动作，开启视图绘制逻辑，创建 `ViewRootImpl` 对象，并调用其 `setView` 方法。

```java
public void addView(...) {
     // 创建ViewRootImpl对象
     root = new ViewRootImpl(view.getContext(), display);
     ...
     try {
         // 执行ViewRootImpl的setView函数
         root.setView(view, wparams, panelParentView, userId);
     } catch (RuntimeException e) {
         ...
     } 
}
```

`setView()` 源码：

```java
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
      synchronized (this) {
         if (mView == null) {
             mView = view;
         }
         ...
         // 开启绘制硬件加速，初始化RenderThread渲染线程运行环境
         enableHardwareAcceleration(attrs);
         ...
         // 1.触发绘制动作
         requestLayout();
         ...
         inputChannel = new InputChannel();
         ...
         // 2.Binder调用访问系统窗口管理服务WMS接口，实现addWindow添加注册应用窗口的操作,并传入inputChannel用于接收触控事件
         res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mDisplayCutout, inputChannel,
                            mTempInsets, mTempControls);
         ...
         // 3.创建WindowInputEventReceiver对象，实现应用窗口接收触控事件
         mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                            Looper.myLooper());
         ...
         // 4.设置DecorView的mParent为ViewRootImpl
         view.assignParent(this);
         ...
      }
}
```

`setView()` 方法中，ViewRootImpl 会将传入的 View（即 DecorView）与窗口管理器（WindowManager）关联起来，并设置必要的参数。随后，ViewRootImpl 会调用 `requestLayout()` 来请求布局更新，这会触发后续的测量、布局和绘制流程。

关于绘制三大步主要涉及不同的View和ViewGroup的测量布局规则不同，细节也可以看冷启动文章。

## UI刷新流程
### Choreographer 编舞者介绍
Choreographer 是 Android 框架中一个**至关重要的系统服务**，它主要负责协调动画、输入事件和 UI 绘制操作的计时，确保这些操作都在**每一次屏幕硬件刷新信号（Vsync）**到来时同步进行。

简单来说，Choreographer的核心作用是**实现流畅的、与屏幕刷新同步的 UI 渲染**。

#### 核心作用：同步 Vsync 信号
Choreographer 最主要的作用是**将应用程序的渲染操作（如绘制、动画计算）与显示屏的垂直同步信号（Vertical Synchronization，简称 Vsync）对齐**。

Vsync 信号是显示硬件发出的一个周期性信号，表示屏幕已经完成了当前帧的显示，可以开始接收下一帧的数据。 在大多数设备上，Vsync 信号的频率是 $60Hz$，意味着每 $16.67$ 毫秒（$1000ms / 60$ 帧）发生一次。

在 **没有 Choreographer 协调的情况下** ，如果应用在屏幕刷新到一半时提交了新的帧数据，就会导致屏幕的上下部分显示两帧不同的内容，形成视觉上的“撕裂”现象（Tearing）。

Choreographer 确保应用的绘制操作只在 Vsync 信号到来后才开始执行，并且在下一个 Vsync 信号到来之前完成，从而**彻底消除画面撕裂**。

如果应用程序在 16.67ms 内没有完成 Measure、Layout 和 Draw 的全部过程，它就会错过当前的 Vsync 信号，导致该帧无法及时显示，用户就会感觉到“卡顿”或“丢帧”（Jank）。

Choreographer 的职责是提供一个清晰的计时框架，让开发者能明确知道自己有多少时间来完成渲染。它为所有需要基于时间同步的操作（如动画、滚动）提供了一个统一、可靠的时间源（Vsync 时间），确保它们以相同的节奏进行。

还可以将在一个短时间内发生的多个 `View.invalidate()` 请求合并起来，只在下一个 Vsync 周期内统一执行一次 `Measure/Layout/Draw`，避免不必要的重复渲染，优化性能。

#### Choreographer工作流程详解
当您执行一个需要更新 UI 的操作（例如调用 $View.invalidate()$ 或启动一个动画）时，Choreographer 的工作流程如下：

1.  **注册回调：** 应用层（如 ViewRootImpl 或 Animator）会向 Choreographer 注册一个回调。
2.  **等待 Vsync：** Choreographer 收到注册请求后，不会立即执行，而是等待系统下一次 Vsync 信号的到来。
3.  **Vsync 信号到达：** 当 Vsync 信号到来时，Choreographer 会被唤醒。
4.  **执行回调：** Choreographer 会在当前这一帧的处理周期内，按照预定的优先级顺序依次执行已注册的各类回调：
    * **CALLBACK_INPUT：** 处理输入事件（如触摸）。
    * **CALLBACK_ANIMATION：** 执行动画计算（如 $ValueAnimator$ 的值更新）。
    * **CALLBACK_TRAVERSAL (最重要)：** 执行 View 树的“遍历”（$Measure$、 $Layout$、 $Draw$）操作，即完成 UI 的实际渲染。
    * **CALLBACK_COMMIT：** 提交绘制结果到 SurfaceFlinger。

所有的操作在一个 Vsync 周期（16.67ms）内完成，并将新的图像数据提交给显示系统，等待下一次 Vsync 信号到来时显示。
### View.invalidate() 刷新流程
整个渲染流水线通常可以分为以下几个核心阶段：**触发 (Invalidate)**、**同步 (Sync/Vsync)**、**绘制 (Draw)**、**提交 (Issue Commands)**、**光栅化 (Rasterization)** 和 **显示 (Display)**。
#### 1\. 触发与同步阶段
当 View 的内容发生变化，需要重绘时，调用此方法。它不会立即重绘，而是将 View 标记为“脏 (dirty)”。 `invalidate()` 最终会将重绘请求传递给 `ViewRootImpl`。`ViewRootImpl` 会调度一个重绘操作 (通过 `Choreographer.postCallback`)，等待下一个 Vsync 信号。

设备屏幕以固定的刷新率（如 60Hz）定时发出垂直同步信号 (Vsync)。`Choreographer` 收到 Vsync 信号。 

同步 (Sync) 阶段开始后`ViewRootImpl` 会执行 `traversal`。包括  **Measure 和 Layout** 以确定 View 的位置。还有动画任务计算动画的下一帧属性值。
#### 2\. 绘制阶段 (CPU 生成 DisplayList)
系统从根 View 开始递归调用 `draw()` 方法。但**在硬件加速开启时**，这个 `draw()` 不再是直接绘制像素，而是**记录**绘制操作。

每个 View 的 `draw()` 方法会将绘制命令（如 "画一个矩形"、"画一个 Bitmap" 等）记录到它自己的 **DisplayList** 中。DisplayList 是一种可重用的，优化的渲染操作序列。这一步由 Android 的渲染引擎 (在旧版本是 OpenGL ES，新版本是 Vulkan/Skia) 在 CPU 上完成。
#### 3\. 提交与传输阶段 (CPU/GPU 协同)
当所有 View 的 DisplayList 都生成后，这些列表会被交给 **RenderThread (渲染线程)**，这是一个独立于主线程的线程。RenderThread 会处理 DisplayList，并将所需的资源（例如，新解码的 **Bitmap**）从 CPU 内存传输到 GPU 内存，作为 **纹理 (Texture)**。这是 CPU 和 GPU 内存之间的同步操作。RenderThread 将 DisplayList 中的高级绘制命令，转换为底层的图形 API 命令，即 **Draw Calls**（通常是 OpenGL ES 或 Vulkan API 调用），并将这些命令排队等待 GPU 执行。
#### 4\. 光栅化与处理阶段 (GPU 工作为主)
**光栅化 (Rasterization) 阶段开始**：GPU 从队列中取出 Draw Calls。光栅化是将向量图形指令（如绘制一个三角形）转换为屏幕上的像素颜色值的过程。 GPU 利用其强大的并行计算能力，对 Draw Calls 中引用的纹理进行采样、应用着色器 (Shader) 程序（如顶点着色器和片段着色器）来确定每个像素的最终颜色。 GPU 将处理完成的像素数据写入到它控制的 **帧缓冲区 (Frame Buffer)** 中。通常有前后两个缓冲区（双缓冲机制）。
#### 5\. 显示阶段 (系统级工作)
当一帧完全渲染到“后缓冲区”后，RenderThread 会调用 **`swapBuffers()`** 或类似的命令。这个操作会告诉 **SurfaceFlinger**（系统级的窗口合成器）该帧已准备好。
 
**SurfaceFlinger** 是一个系统服务，它负责收集所有可见窗口（应用、状态栏、导航栏等）的最新帧缓冲区，并根据它们的 Z-order、位置和透明度，将它们合成到最终的屏幕缓冲区中。这个合成过程本身也可以由 GPU 加速完成。

在下一个 Vsync 信号到来时，**显示硬件**（Display Hardware）从最终的合成缓冲区读取数据，并将图像电流发送到屏幕，最终用户才能看到更新后的内容。
