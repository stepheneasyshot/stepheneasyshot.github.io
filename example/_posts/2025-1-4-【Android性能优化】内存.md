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
# 【Android性能优化】内存
## 内存
### 几种引用类型

![referance_type](/assets/img/blog/blogs_jvm_referance_type.png){:loading="lazy"}

* 强引用：
    * `Object a=new object()`; 
    * Java中采用new关键字创建对象就是一种强引用。对于强引用的对象，就算是出现了OOM也不会对该对象进行回收。在Java中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它 **处于可达状态** ，它是不可能被垃圾回收机制回收的。
    * 强引用是造成Java内存泄漏的主要原因之一。 对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为null，一般认为就是可以被垃圾收集的了。
* 软引用：
    * `SoftReference<Object> softReference=new SoftReference<>(o1);`
    * 对于只有软引用的对象来说，**当系统内存充足时它不会被回收，当系统内存不足时它会被回收** 。软引用通常用在对内存敏感的程序中，比如高速缓存就有用到软引用，内存够用的时候就保留，不够用就回收！
* 弱引用：
    * `WeakReference<Object> weakReference=new WeakReference<>(o1);`
    * 对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管JVM的内存空间是否足够，都会回收该对象占用的内存。需要使用WeakReference来实现。
* 虚引用：
    * `PhantomReference<Object> phantomReference=new PhantomReference<>(o1,referenceQueue);`
    * 虚引用是所有引用类型中最弱的一个。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。**为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知**。（虚引用必须和引用队列 （ReferenceQueue）联合使用 ）。

### 内存泄漏
内存泄漏是指在应用程序中，由于某些原因导致不再使用的对象（即垃圾对象）无法被垃圾回收器回收，从而占用了内存空间。内存泄漏可能会导致应用程序的性能下降、内存占用增加，甚至导致应用程序崩溃。

在 Android 开发中，内存泄漏是一个需要特别关注的问题。以下是一些常见的内存泄漏场景：

#### **单例模式引起的内存泄漏**
如果单例对象持有了一个生命周期较短的对象的引用，而这个单例的生命周期与整个应用程序的生命周期相同，就可能导致内存泄漏。

```java
public class Singleton {
    private static Singleton instance;
    private Context context;

    private Singleton(Context context) {
        this.context = context;
    }

    public static Singleton getInstance(Context context) {
        if (instance == null) {
            instance = new Singleton(context);
        }
        return instance;
    }
}
```
在这个单例中，保存了对 Context 的引用。如果传入的是 Activity 的上下文，当 Activity 销毁时，由于单例仍然持有其引用，导致 Activity 无法被回收，从而造成内存泄漏。

解决方案就是尽量使用Application的Context。

#### **非静态内部类引起的内存泄漏**
非静态内部类会隐式地持有外部类的引用。如果在外部类（如 Activity）的生命周期内，非静态内部类的实例一直存在，就可能导致外部类无法被回收。在Android开发中，设置的点击监听器，还有Handler等都是常见的非静态内部类的使用场景。

```java
public class MainActivity extends AppCompatActivity {
    private Button button;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        button = findViewById(R.id.button);
        button.setOnClickListener(new MyClickListener());
    }

    private class MyClickListener implements View.OnClickListener {
        @Override
        public void onClick(View v) {
            // 处理点击事件
        }
    }
}
```

当 MainActivity 销毁时，由于 MyClickListener 实例持有 MainActivity 的引用，导致 MainActivity 无法被回收。

#### **Handler 引起的内存泄漏**
如果在 Activity 中使用 Handler 发送延迟消息，当 Activity 销毁时，消息可能还未被处理，而 Handler 又持有 Activity 的引用，就会导致 Activity 无法被回收。

```java
public class MainActivity extends AppCompatActivity {
    private Handler handler = new Handler();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                // 处理延迟任务
            }
        }, 5000);
    }
}
```

当 MainActivity 销毁后，由于 Handler 中的延迟任务可能还未执行完毕，导致 MainActivity 无法被回收。

解决方案：
* 在 Activity 销毁时，及时移除 Handler 中的所有消息。
* 将Handler改为静态内部类 + WeakReference 来避免内存泄漏。

#### **资源未关闭引起的内存泄漏**
例如，对数据库、文件流、网络连接等资源未及时关闭，可能导致资源对象一直被持有，从而造成内存泄漏。

```java
public class MainActivity extends AppCompatActivity {
    private SQLiteDatabase database;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        database = openOrCreateDatabase("mydb", Context.MODE_PRIVATE, null);
    }
}
```

如果在 Activity 销毁时没有 **关闭数据库连接** ，database 对象将一直存在，导致 MainActivity 无法被回收。

解决方案就是在 Activity 销毁时，及时关闭数据库连接。或者在单次读取的场景下使用try-with-resources语句，完毕后会自动关闭资源。

#### **注册的监听器未注销引起的内存泄漏**
如果在 Activity 中注册了监听器，如 **广播接收器、系统服务、系统数据库的监听器** 等等，在 Activity 销毁时没有注销这些监听器，就会导致 Activity 无法被回收。

```java
public class MainActivity extends AppCompatActivity {
    private BroadcastReceiver receiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        receiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                // 处理广播
            }
        };
        registerReceiver(receiver, new IntentFilter("com.example.action"));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 忘记注销广播接收器
    }
}
```

#### **线程未停止引起的内存泄漏**
如果在 Activity 中启动了一个线程，该线程持有了 Activity 的引用，即使 Activity 已经被销毁，线程仍然在运行，就会导致 Activity 无法被回收。原理和解决方案和Handler导致的泄露基本相同。

```java
public class MainActivity extends AppCompatActivity {
    private Thread thread;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        thread = new Thread(new Runnable() {
            @Override
            public void run() {
                // 执行耗时操作
            }
        });
        thread.start();
    }
}
```

#### **集合类存储长生命周期的对象导致泄露**
集合类使用不当导致的内存泄漏，这里分两种情况来讨论：

1）集合类添加对象后不移除的情况
对于所有的集合类，如果存储了对象，如果该 **集合类实例的生命周期比里面存储的元素还长** ，那么该集合类将一直持有所存储的短生命周期对象的引用，那么就会产生内存泄漏，尤其是使用static修饰该集合类对象时，问题将更严重。我们知道static变量的生命周期和应用的生命周期是一致的，如果添加对象后不移除，那么其所存储的对象将一直无法被gc回收。解决办法就是**根据实际使用情况，存储的对象使用完后将其remove掉**，或者使用完集合类后**清空集合**。

2）根据hashCode的值来存储数据的集合类使用不当造成的内存泄漏以HashSet为例子，当一个对象被存储进HashSet集合中以后，就不能再修改该对象中 **参与计算hashCode的字段值** 了，否则，原本存储的对象将无法再找到，导致无法被单独删除，除非清空集合。

#### **第三方库使用不当造成的内存泄漏**
使用第三方库的时候，务必要按照官方文档指定的步骤来做，否则使用不当也可能产生内存泄漏，比如：
* EventBus，也是使用观察者模式实现的，同样注册和反注册要成对出现。
* Rxjava中，上下文销毁时，Disposable没有调用dispose()方法。
* Glide中，在子线程中大量使用Glide.with(applicationContext)，可能导致内存溢出。

### 内存泄露问题分析方法
排查内存问题，我们的项目中可以在debug构建的情况下，使用LeakCanary来进行内存泄漏的检测。LeakCanary会在应用程序发生内存泄漏时，自动生成一个报告，帮助开发者定位和修复内存泄漏问题。

在生产环境中遇到一个内存泄漏问题，如果是必现，我们可以直接根据必现流程，判断我们的应用中执行了哪些代码块，正向追代码引用流程排查。

如果是偶现的，需要先按照日志显示的手顺流程，尝试去使用带LeakCanary的debug版本复现问题。

其次，我们可以使用AS的 `Memory Profiler` 功能模块来分析内存使用情况，实时地查看内存使用的对象和引用关系，从而定位内存泄漏的原因。如果有内存泄露，应用运行一段时间后内存占用会持续上升。

Profiler入口：

![memory_profiler](/assets/img/blog/blogs_android_studio_profiler.png)

Live Telemetry 也可以实时查看应用的内存使用情况，包括内存分配、内存释放等信息。可以通过点击 Live Telemetry 按钮来打开 Live Telemetry 窗口。

![memory_profiler_1](/assets/img/blog/blogs_android_live_telemetry.png)

还可以直接在shell中使用top或者ps命令来查看应用的内存使用情况，边操作边观察，看看是哪里导致的内存泄漏，再去分析可能的原因来解决。

![memory_top](/assets/img/blog/blogs_top_performance.png)

#### **LeakCanary检测原理**
前天的另一篇文章详细介绍了检测泄漏的流程和原理：

[LeakCanary工具的原理解析](./2025-1-1-LeakCanary工具的原理解析.md)

### 内存抖动（Memory Stutter）
内存抖动是指应用程序在短时间内频繁地分配和释放内存，导致系统频繁地进行垃圾回收，从而降低应用程序的性能。内存抖动通常发生**在循环中**，创建临时对象或者频繁地进行内存分配和释放操作。

可能的原因：
1. 频繁的对象创建和销毁
    ◦ 在循环中频繁地创建和销毁对象，会导致内存的频繁分配和释放，从而增加垃圾回收的负担。例如，在一个循环中创建了大量的临时对象，这些对象在循环结束后就不再使用，但由于没有及时释放，会导致内存占用不断增加。
    ◦ 解决方案：尽量避免在循环中频繁创建对象，可以考虑使用对象池或者重用对象的方式来减少内存分配的次数。
2. 内存分配和释放不平衡
    ◦ 内存分配和释放的频率不平衡，导致内存的使用量不断变化，从而增加了垃圾回收的负担。例如，在一个循环中，每次循环都分配了一个新的对象，但在循环结束后没有及时释放这些对象，导致内存占用不断增加。
    ◦ 解决方案：在循环结束后，及时释放不再使用的对象，避免内存的过度分配和释放。可以使用对象池或者重用对象的方式来减少内存分配的次数。
3. 内存占用过大
    ◦ 应用程序的内存占用过大，导致系统频繁地进行垃圾回收，从而降低应用程序的性能。例如，应用程序的内存占用超过了系统的可用内存，导致系统频繁地进行垃圾回收，从而降低应用程序的性能。
    ◦ 解决方案：优化应用程序的内存使用，减少内存的占用。可以使用内存分析工具（如 Android Studio 的 Memory Profiler）来检测应用程序中的内存泄漏情况，并进行相应的优化。
4. 内存碎片 
    ◦ 内存碎片是指内存中的空闲区域分散不均匀，导致无法分配足够大的连续内存空间。例如，在一个循环中，每次循环都分配了一个新的对象，但在循环结束后没有及时释放这些对象，导致内存中的空闲区域分散不均匀，无法分配足够大的连续内存空间。    
    ◦ 解决方案：尽量避免在循环中频繁创建对象，可以考虑使用对象池或者重用对象的方式来减少内存分配的次数。

### 应用内存溢出（Out Of Memory）
内存溢出是指应用程序试图分配比系统可用内存更多的内存空间。在 Android 中，每个应用程序都有一个特定的内存限制，这个限制取决于设备的硬件和操作系统版本。当应用程序尝试分配的内存超过这个限制时，就会发生内存溢出错误。在手机上，内存溢出错误通常会导致应用程序崩溃。在车机开发中，系统应用的内存大小如果没有限制，发生OOM甚至会直接导致系统重启。发生内存溢出的可能场景有：

1. 频繁加载大图片，如果直接加载高分辨率的大图片而不进行适当的处理，会占用大量的内存。例如，加载一张 4000x3000 像素的图片，可能会消耗几十兆甚至上百兆的内存。可以使用图片加载库（如 Glide、Picasso 等），这些库通常会自动根据设备的屏幕尺寸和内存情况对图片进行缩放和缓存管理，以减少内存占用。
2. 过多的对象创建，在循环中频繁创建对象或者创建大量不必要的临时对象，会导致内存快速增长。比如在一个频繁调用的方法中不断创建新的字符串对象。可以尽量复用对象，避免不必要的对象创建。对于在循环中创建的对象，可以考虑在循环外部创建并重复使用。
3. 数据缓存不合理，如果缓存的数据过多或者没有及时清理过期的缓存，会占用大量内存。例如，一个网络请求缓存了大量的 JSON 数据，但没有设置缓存大小限制或过期时间。解决方案：合理设置缓存大小和过期时间，定期清理不再需要的缓存数据。可以使用 LruCache 等缓存工具来管理内存缓存。
4. 内存泄漏，不再使用的对象仍然被其他对象引用，导致垃圾回收器无法回收它们的内存。随着时间的推移，内存泄漏会积累大量无法回收的内存，最终导致内存溢出。解决方案：及时释放不再使用的资源，避免静态变量持有长生命周期对象的引用，注意 Activity、Service 等组件的生命周期管理，防止内存泄漏的发生。

在 Android 开发中，要避免内存溢出问题，需要注意合理使用内存资源，优化代码结构和算法，及时清理不再需要的资源，并使用合适的工具进行内存分析和优化。

## Low Memory Killer (LMK) 机制
Low Memory Killer 是 Android 系统为了在内存不足时保持系统响应性和性能而设计的一套进程终止机制。

它预测性地、分级地在**系统内存压力达到某个阈值时**，就主动杀死“不重要”的进程，以释放内存，避免等到系统彻底耗尽内存（OOM）时才触发最后的紧急清理。

Low Memory Killer 的功能由用户空间的守护进程 lmkd (Low Memory Killer Daemon) 来执行，它不再是传统的内核驱动程序，而是通过监听内核的内存压力信号（如 vmpressure 事件或 PSI - 压力失速信息）来工作。

LMK 的决定是基于 Android 系统分配给每个进程的 **“优先级分数”** ，通常称为 oom_adj_score（或简写为 adj 值）。这个分数由 Android 的 ActivityManagerService 动态计算和设置，反映了进程对用户的价值和其生命周期状态。

lmkd 会根据内存压力程度，查看所有运行中的进程，并根据它们的 oom_adj_score 从高到低（即“最不重要”到“最重要”）开始杀死进程，直到释放足够的内存。


`oom_adj_score`（在 Android 源码中通常称为 `adj` 或 `oom_adj`）并不是通过一个简单的数学公式计算出来的，而是由 Android 核心组件 **`ActivityManagerService` (AMS)** **动态分配和调整**的。

它的计算逻辑是复杂的，它基于应用程序进程中**运行的组件**及其**与用户的交互状态**。这个分数决定了进程的**重要性**，分数越高，优先级越低，越容易被 Low Memory Killer (LMK) 杀死。

### oom_adj_score 计算
以下是 `oom_adj_score` 的主要决定因素和计算逻辑：

| 进程状态/组件 | `oom_adj` 值（示例） | 描述 |
| :--- | :--- | :--- |
| **前台进程 (Foreground)** | 0 | 进程内有用户当前正在交互的 Activity。**最高优先级，不会被 LMK 杀死。** |
| **可见进程 (Visible)** | 1-200 | 进程内有可见但非焦点的 Activity（如对话框下的 Activity）。优先级仅次于前台。 |
| **前台服务进程 (Foreground Service)** | 100-200 | 进程内有通过 `startForeground()` 运行的服务（如音乐播放、GPS）。 |
| **后台服务进程 (Service)** | 200-400 | 进程内有正常运行的 Service，但没有 Activity。 |
| **可缓存的空进程 (Cached)** | 900+ | 进程内没有任何活动的组件，仅保留在内存中以便快速重启。**最低优先级，LMK 的主要目标。** |

## Memory Profiler
使用 `Memory Profiler` 来分析内存是性能优化的重要方式。其中显示的各个区域反映了进程地址空间中不同类型内存的占用情况。

| 区域名称 | 对应内容 (Content) | 内存类型 (Type) | 典型占用 (Typical Occupants) |
| :--- | :--- | :--- | :--- |
| **Java Heap** | JVM（Java/Dalvik）虚拟机管理的内存区域。 | 堆 (Heap) | Java 对象实例、Kotlin 对象实例、非压缩的 `Bitmap` 对象引用、应用自定义类实例等。这是最常分析的区域。 |
| **Native Heap** | C/C++ 代码（或底层系统库）直接通过 `malloc`/`new` 等函数分配的内存区域。 | 堆 (Heap) | **`Bitmap` 的像素数据**、渲染引擎（如 Skia、Vulkan）、JNI 分配的内存、游戏引擎（如 Unity/Cocos）数据。 |
| **Code** | 应用程序和系统库的可执行机器码（`.dex`、`.so` 文件）以及运行时生成的代码。 | 代码段 (Text Segment) | DEX 文件（包含 Java 字节码）、本地库 (`.so` 文件)、JIT/AOT 编译后的机器码。 |
| **Stack** | 为每个线程分配的私有内存区域。 | 栈 (Stack) | 函数调用栈、方法参数、局部变量（基本类型、对象引用）。Stack 内存通常很小，不会导致 OOM。 |
| **Graphics** | 用于处理显示和图形相关的内存。 | 专用内存 (Dedicated) | 图像缓冲区 (Buffers)、纹理、`SurfaceFlinger` 相关的内存。 |
| **Other** | 未归类到以上任何区域的内存。 | 混合 (Mixed) | 内存映射文件（`mmap`）、文件 I/O 缓冲区、系统内核分配的页表等。 |

## Bitmap 对象的内存体现
`Bitmap` 是 Android 内存分析中最特殊也最重要的对象之一。很多的泄露问题中，如果是图片加载导致的泄露，现象一般比较明显和严重。

它的内存占用被**分割**到两个不同的内存区域中：
* Java Heap 是 `Bitmap` 对象的**引用和元数据**存储的地方。存储 `Bitmap` 对象的 Java 引用（`java.lang.Object`）以及它的元信息（如宽度、高度、配置等）。这部分内存很小，通常只有几十字节。
* Native Heap（原生堆），这是 `Bitmap` 占用的**绝大部分内存**，即**像素数据 (Pixel Data)** 存储的地方。注意在 Android 8.0 之后`Bitmap` 的像素数据是存储在 **Java Heap** 中的。如果使用了 `BitmapFactory.Options.inPreferredConfig = Bitmap.Config.HARDWARE` 或 JNI C/C++ 代码来处理图像，像素数据仍然可能被分配在 Native Heap 或 Graphics 区域。

### 后台更新ImageView的危险操作
曾经开发过一个悬浮窗式的APP，没有Activity组件，即需要自己管理View和数据的生命周期关系。有一次在后台不断地更新ImageView的帧动画对象，导致了严重的内存问题。

当你通过 `AnimationDrawable`（帧动画的实现类）加载一系列图片帧时，无论 `ImageView` 是否在屏幕上显示，这个过程都会发生以下事情：

1.  **解码成Bitmap：** 每一张图片资源（比如 `R.drawable.frame1`）都会被解码成一个 `Bitmap` 对象。`Bitmap` 是Android中表示位图的类，它包含了图片的像素数据。
2.  **内存位置：** 这些 `Bitmap` 对象占用的内存位于应用的 **堆内存（Heap Memory）** 中。`Bitmap` 的像素数据也主要分配在应用的堆内存里，由Java/Kotlin的垃圾回收器（GC）管理。
3.  **持有引用：** `AnimationDrawable` 对象会内部持有一个列表，用来存储每一帧对应的 `Drawable` 对象。这些 `Drawable` 对象最终会持有对 `Bitmap` 对象的强引用。

所以，即使界面没有显示，只要你的代码执行了加载帧动画的逻辑，那些被解码后的图片 `Bitmap` 对象就会被创建并存储在应用的堆内存中。`AnimationDrawable` 实例持有这些 `Bitmap` 的引用，防止它们被垃圾回收器回收。

这个“缓存”其实就是 `AnimationDrawable` 对象自身对所有帧图像`Bitmap`的直接持有。

**如果不断读取新的图片，会使后台的内存占用越来越大**。这是一个非常危险的操作，极有可能导致应用崩溃。**

如果应用在后台（用户看不到UI），还在持续地加载新的图片帧到 `AnimationDrawable` 中，会发生以下情况：

1.  **内存持续增长：** 每加载一张新的图片，就会在堆内存中创建一个新的 `Bitmap` 对象。由于 `AnimationDrawable` 持有它的引用，这块内存就无法被释放。应用的堆内存占用会像滚雪球一样越来越大。
2.  **触发 OOM (OutOfMemoryError)：** 每个Android应用都有一个固定的堆内存上限（具体大小因设备而异）。当你的应用内存占用超过这个上限时，系统会抛出 `OutOfMemoryError` 异常，导致应用直接崩溃。
3.  **被系统“杀死”：** 即使没有立刻OOM，一个在后台占用大量内存的应用也会给系统带来很大压力。当系统需要更多内存给前台应用（比如用户正在使用的其他App）时，你的应用会成为被系统强制关闭（kill process）的优先目标。用户下次回到你的App时，会发现它被重启了，体验非常糟糕。

#### 最佳实践与解决方案

核心原则是：**UI资源的加载和释放，必须与UI组件的生命周期严格绑定。**

对于帧动画，正确的处理方式如下：

**1. 不要在后台加载和启动动画**

动画是给用户看的，当UI不可见时，任何动画操作都是在浪费CPU和内存。你应该在界面变为可见时才开始加载和播放动画。

**2. 遵循Activity/Fragment的生命周期**

在 `Activity` 或 `Fragment` 的生命周期回调方法中管理 `AnimationDrawable` 是最标准、最安全的方式。

  * **`onStart()` 或 `onResume()`:** 在这里获取 `AnimationDrawable` 对象并调用 `start()` 方法开始播放。这时UI对用户是可见的。
  * **`onStop()` 或 `onPause()`:** 在这里必须调用 `stop()` 方法停止动画。这时UI已经不可见或被部分遮挡。停止动画不仅可以节省CPU，更重要的是，系统可以有机会回收 `AnimationDrawable` 内部的 `Bitmap` 资源（如果你也解除了对它的引用）。

