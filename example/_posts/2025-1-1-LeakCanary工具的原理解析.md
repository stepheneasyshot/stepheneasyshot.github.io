---
layout: post
description: > 
  本文介绍了性能测试和性能优化的方法论和实例
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
# LeakCanary工具的原理解析
LeakCanary 是 Square 公司开源的一款用于检测 Android 应用中 内存泄漏（Memory Leak） 的自动化工具。它能够在应用运行时自动检测内存泄漏，尤其是像 **Activity、Fragment** 等组件的泄漏，并在发现泄漏时通过通知提醒开发者，同时提供详细的泄漏引用链信息，帮助开发者快速定位问题。
## 工作原理
主要分为以下几个主要阶段：
### 1. 监控 Activity 和 Fragment 的生命周期
在 Application 类中，通常会调用 `LeakCanary.install(this)`。这是 LeakCanary 的入口点。（在2.0版本已经实现了隐式调用，无需手动调用install方法）

> LeakCanary 2.0 利用了 Android 的 ContentProvider 自动初始化机制，通过在库中注册一个内部的 LeakCanaryInstaller ContentProvider，系统会在 Application.onCreate() 之前自动初始化它。这样设计有几个好处：一是简化了集成流程，开发者 **只需添加依赖** 即可；二是实现了自动按需初始化，只在 debug 构建中工作；三是遵循了现代 Android 库的设计趋势。这种改变使得内存泄漏检测对开发者更加透明和无侵入。

LeakCanary 通过注册 `Application.ActivityLifecycleCallbacks` 和 `FragmentManager.FragmentLifecycleCallbacks` 来监听所有 Activity 和 Fragment 的生命周期事件。

在这期间，还会初始化后台线程池。LeakCanary 会创建一个专门的后台线程池来执行耗时的操作，例如后面的堆转储操作，以避免阻塞主线程。初始化通知管理器，用于在检测到泄漏或进行堆转储时显示通知。
### 2. 检测内存泄漏
#### **监听onDestroy回调**
以Activity为例，当用户退出一个 Activity 时， `Activity` 的 `onDestroy()` 方法会被调用。

由于 LeakCanary 注册了 `Application.ActivityLifecycleCallbacks` ，它会接收到这个 Activity 的 onDestroyed 回调。

在收到 onDestroyed 回调通知后，LeakCanary 会对即将被销毁的 Activity 对象创建一个特殊的 `KeyedWeakReference` 。这个 KeyedWeakReference 不仅仅是一个普通的弱引用，它还包含一个唯一的 key 和一些元数据（如 Activity 的类名、创建时间等），用于在后续分析中识别对象。

 这个 KeyedWeakReference 会被添加到 LeakCanary 内部的一个 ObjectWatcher 维护的观察列表中。
#### **检查弱引用观测列表**
LeakCanary 内部有一个周期性的任务，会定期在后台线程中运行。在这个任务中，LeakCanary 会主动调用 `System.gc()` 来触发一次垃圾回收。

> 需要注意的是，System.gc() 只是建议 JVM 进行垃圾回收，并不能保证立即执行或完全清除所有可回收对象。 LeakCanary 会多次尝试 GC，以提高清除弱引用的概率。

在 GC 之后，LeakCanary 会遍历之前创建的集合，查看其中的弱引用是否已经被清除。
* 弱引用已清除： 如果 KeyedWeakReference.get() 返回 null，说明它引用的 Activity 对象已经被垃圾回收了。这表示 Activity 正常地被销毁， **没有发生内存泄漏** 。这个 KeyedWeakReference 就会从集合中移除。
* 弱引用未清除（被保留）： 如果 KeyedWeakReference.get() 仍然返回 **非 null** ，说明它引用的 Activity 对象仍然存在于内存中，它其实应该是被销毁的。此时，LeakCanary 就认为这个 Activity 对象被“保留（retained）”了，并且很可能发生了内存泄漏。 LeakCanary 会在 Logcat 中打印一条信息，指示哪个 Activity 被保留了。
### 3. 触发堆转储
LeakCanary 会统计被保留对象的数量。默认情况下，当被保留对象的数量达到 5个（可配置）时，LeakCanary 会触发一次堆转储。这是为了避免频繁的堆转储对用户体验造成影响。

应用在后台： 如果应用进入后台，LeakCanary 会更积极地触发堆转储，默认情况下，只要监测到被保留对象时就会触发。因为在后台时，堆转储对用户体验的影响较小。

当触发堆转储时，LeakCanary 会显示一个 Toast 提示用户，同时在通知栏显示一个进度通知。

LeakCanary 会调用 `Debug.dumpHprofData(filePath)` 方法将当前 Java 堆的完整快照保存为一个 `.hprof` 文件到应用的私有存储空间。这个过程是一个耗时操作，会短暂地阻塞应用的主线程。
### 4. 分析堆转储文件与展示
为了不影响应用的主进程，LeakCanary 会在一个 **独立的后台进程** 中启动一个服务（HeapAnalyzerService）来处理 `.hprof` 文件的分析。这样做的好处是即使堆分析崩溃或出现内存问题，也不会影响到应用本身。

对堆转储文件的分析会使用 LeakCanary 内部的 Shark 库来解析 .hprof 文件。
* 查找 GC Roots：Shark 会首先识别出**所有的 GC Roots**（垃圾回收的根对象）。
* 遍历对象图：从 GC Roots 开始，Shark 会遍历整个对象图，查找所有**可达的对象**。
* 定位被保留对象： Shark 会通过之前 KeyedWeakReference 中存储的 key 来定位到之前被标记为 **“被保留”** 的对象。
* 计算最短强引用路径： 这是分析的核心。Shark 会从 GC Roots 到被保留对象之间，**计算出最短的强引用路径**。这个路径就是导致泄漏的“泄漏跟踪（leak trace）”。它会显示哪些对象持有对泄漏对象的强引用，直到某个 GC Root。
* 过滤已知泄漏：LeakCanary 内置了一些规则，可以识别并忽略一些 Android 框架内部的已知泄漏，避免误报。
* 识别可疑点： LeakCanary 会尝试根据泄漏跟踪识别出最可能导致泄漏的代码位置或对象类型。

```
哪些可以作为GC Roots的对象呢？Java 语言中包含了如下几种：
        1）虚拟机栈（栈帧中的本地变量表）中的引用的对象。
        2）方法区中的类静态属性引用的对象。
        3）方法区中的常量引用的对象。
        4）本地方法栈中JNI（即一般说的Native方法）的引用的对象。
        5）运行中的线程
        6）由引导类加载器加载的对象
        7）GC控制的对象
```

分析完成后，LeakCanary 会在通知栏弹出一条通知，提示开发者检测到了内存泄漏。

提供一个直观的 UI 界面（通常是 LeakActivity），展示泄漏对象的详细信息，包括：
* 泄漏对象的类型（如 MainActivity）。
* 泄漏对象的引用链（即哪些对象持有了它的引用）。
* 可能的泄漏原因分析（如静态变量持有、Handler 未释放等）。

开发者可以通过这个界面快速定位问题，并进行修复。

## 优点
LeakCanary 是一款非常优秀的内存泄漏检测工具。

具体来具有以下优点：
* 自动化程度高：无需手动触发，自动检测内存泄漏，适合开发和测试阶段使用。
* 直观易用：提供清晰的 UI 界面展示泄漏信息，帮助开发者快速定位问题。
* 轻量级：对应用性能影响较小，不会显著增加应用的体积或运行时开销。
* 开源免费：由 Square 公司维护，代码开源，社区活跃，易于集成和定制。

## 局限性
尽管 LeakCanary 是一款非常优秀的内存泄漏检测工具，但它也有一些局限性：
* 仅针对 Activity 和 Fragment：默认情况下，LeakCanary 主要检测 Activity 和 Fragment 的泄漏，其他对象（如自定义 View、Service 等）需要手动扩展。
* Heap Dump 分析耗时：生成和分析 Heap Dump 可能会消耗一定的时间和内存资源，尤其是在内存较大的应用中。
* 无法实时监控：LeakCanary 是在对象销毁后检测泄漏，无法实时监控内存的使用情况（如内存增长趋势）。
* 对 ProGuard/R8 混淆支持有限：如果应用启用了代码混淆，泄漏引用链中的类名和方法名可能会被混淆，增加分析难度（但 LeakCanary 提供了一定的反混淆支持）。
