---
layout: post
description: > 
  本文介绍了Android应用性能优化的方法论和实例
image: 
  path: /assets/img/blog/blogs_performance.png
  srcset: 
    1920w: /assets/img/blog/blogs_performance.png
    960w:  /assets/img/blog/blogs_performance.png
    480w:  /assets/img/blog/blogs_performance.png
accent_image: /assets/img/blog/blogs_performance.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android性能优化】冷启动

## 冷启动优化
在 Android 应用开发中，冷启动（Cold Start） 是指应用从完全关闭状态（进程不存在）到用户看到第一个界面（通常是 Launcher 或 SplashActivity）的启动过程。冷启动是用户感知应用性能的关键环节之一，如果冷启动时间过长，会导致用户流失或体验下降。一般的测试流程里，将手指点击图标后，应用首帧显示到屏幕上的时长作为指标，这个比较符合用户的真实体验。

因此，冷启动优化是 Android 性能优化的重要部分。以下是常见的冷启动优化手段，按优化方向分类进行详细说明。

简单来说，冷启动可以分为以下几个阶段：
> 应用进程创建，系统接收到启动应用的请求后，首先会创建应用的进程（Zygote 进程 fork 出新进程）。应用进程创建后，会初始化 Application 对象，执行 Application.onCreate() 方法。如果在 Application 的 onCreate 中执行了耗时操作（如初始化第三方库、加载大量数据等），会导致冷启动时间变长。然后系统会创建目标 Activity 的实例，并调用其生命周期方法（如 onCreate()、onStart()、onResume()）。然后是Activity 的布局加载、视图测量与绘制（Measure、Layout、Draw）。

详细流程可以看这一篇：

[Android 冷启动流程分析](./2024-9-21-APP冷启动流程解析.md)

### 优化手段
#### 减少 Application.onCreate() 和 Activity.onCreate() 中的工作量
Application 是应用的入口点，很多开发者会在 Application.onCreate() 中初始化各种第三方库、框架或服务。如果这些初始化操作耗时较长，会直接影响冷启动时间。有些第三方库或服务并不需要在应用启动时立即初始化（如统计 SDK、日志 SDK、推送 SDK 等），可以在应用启动后，真正需要使用这些库时再进行初始化。也可以将非关键的初始化操作移到后台线程中执行。

异步加载，将数据加载、图片处理、网络请求等耗时操作放到后台线程中执行，避免阻塞主线程。可以使用 Kotlin Coroutines、RxJava 或 `Executor` 来处理异步任务。

优化数据加载，如果需要从本地存储或网络加载数据，只加载初始屏幕所需的数据，而不是一次性加载所有数据。可以考虑分页加载或按需加载。
#### 优化布局和视图层次结构
减少布局的嵌套层级。过深的视图层次会增加测量和绘制时间。

对于简单的线性布局，`LinearLayout` 和 `FrameLayout` 通常比 `ConstraintLayout` 更快。对于复杂布局，`ConstraintLayout` 可以帮助减少嵌套，从而提升性能。

对于不经常显示或在启动时不需要显示的 UI 部分，可以使用 `ViewStub` 作为占位符，在需要时再动态加载。这可以减少初始布局的膨胀时间。

同时，减少不必要的背景、重叠视图等，这些都会增加 GPU 的绘制负担。
#### 利用 Android 平台提供的优化工具
* Baseline Profiles (基线配置文件)，这是 Google 推荐的重要优化手段。Baseline Profiles 可以在首次启动时将代码执行速度提高 30%，使应用启动、屏幕导航、内容滚动等用户交互更加流畅。它通过在编译时优化 DEX 布局来提高启动速度。
* App Startup 库，这个库允许你定义一个内容提供者来统一初始化多个组件，而不是为每个组件都定义一个单独的内容提供者，从而显著提高应用启动时间。
* R8 优化编译器，启用 R8 的完整模式可以进行更激进的代码优化，包括代码缩减、资源优化、DEX 布局优化等，从而减少应用大小并提高运行时性能，包括启动速度。

#### 图片和资源优化
确保图片大小合适，并进行有效压缩。对于显示在 `ImageView` 中的图片，将其尺寸调整为与 `ImageView` 匹配，避免加载过大的图片。
* 考虑使用 WebP 等高效的图片格式。
* 使用 Glide、Picasso 等图片加载库在后台线程加载和缓存图片。
* 如果应用包含大量功能，可以考虑将其拆分为动态模块，按需下载和安装，从而减小初始安装包大小，加快启动速度。

### 其他技巧
* **闪屏页优化：** 如果使用 Splash 闪屏页，可以在闪屏页显示期间进行一些必要的初始化工作，从而充分利用用户等待的时间。
* 如果应用中注册了多个 ContentProvider，系统会在应用启动时初始化这些 ContentProvider，可能导致冷启动时间变长。可以保留必要的 ContentProvider，移除不必要的 ContentProvider。
* 同样的，如果应用注册了大量的广播接收器（尤其是静态注册的广播接收器），系统会在应用启动时加载这些接收器，可能导致冷启动时间变长。尽量使用动态注册的广播接收器，避免静态注册。只注册必要的广播接收器，移除不必要的广播接收器。
* 在冷启动时，如果频繁调用系统服务（如 LocationManager、SensorManager 等），可能会导致系统资源竞争，增加冷启动时间。可以延迟调用系统服务，避免在 Application.onCreate() 或 Activity.onCreate() 中立即调用。或者使用缓存机制，避免重复调用系统服务。

需要注意的是
* 冷启动优化不能以牺牲功能为代价。例如，延迟初始化某些 SDK 可能会导致功能不可用，需要根据实际场景权衡。
* 还有，冷启动时间可能因设备性能、系统版本等因素而异。建议在多种设备上进行测试，确保优化效果。
* 冷启动优化的最终目标是提升用户体验。可以通过启动主题、加载动画等方式掩盖部分初始化时间，提高用户感知的流畅性。

## 悬浮窗主题切换速度优化
一个悬浮窗应用，内部无Activity，界面显示是在Service里通过 `windowmanager` 来添加View，外部通过AIDL接口来通信。

在响应系统深浅主题切换时，复写 `Service` 的 `onConfigurationChanged()` 方法，在里面改变View的主题，给每一个View动态设置资源，文字换颜色，背景，图片替换资源等等。

然而界面内部需要替换的图片资源众多，发现切换耗时有1s以上，观感上很卡顿。

简化后的代码如下：

```kotlin
    override fun onConfigurationChanged(newConfig: Configuration) {
        super.onConfigurationChanged(newConfig)
        changeTheme()
    }

    private fun changeTheme() {
        mLargePictureList[0].setImageResource(R.drawable.pic_large_picture_1)
        mLargePictureList[1].setImageResource(R.drawable.pic_large_picture_2)
        mLargePictureList[2].setImageResource(R.drawable.pic_large_picture_3)
        mLargePictureList[3].setImageResource(R.drawable.pic_large_picture_4)
        mLargePictureList[4].setImageResource(R.drawable.pic_large_picture_5)
    }
```

每个ImageView都是通过`setImageResource()`来设置不同的资源。切换主题后，应用自动根据内部的资源Config去找到对应的深浅图片资源。

我们看看这个 `setImageResouce()` 方法里面做了什么：

```java
@android.view.RemotableViewMethod(asyncImpl="setImageResourceAsync")
public void setImageResource(@DrawableRes int resId) {
    // The resource configuration may have changed, so we should always
    // try to load the resource even if the resId hasn't changed.
    final int oldWidth = mDrawableWidth;
    final int oldHeight = mDrawableHeight;

    updateDrawable(null);
    mResource = resId;
    mUri = null;

    resolveUri();

    if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
        requestLayout();
    }
    invalidate();
}
```


根据这个简化的demo，我采集一段trace，查看主题切换过程中的耗时情况：

![](/assets/img/blog/blogs_configchange_trace.png)

可以看到，共计耗时161ms，其中主要耗时都在五个ImageView的 `ImageDecoder.decodeDrawable()` 方法，这正是用来解码图片的。
