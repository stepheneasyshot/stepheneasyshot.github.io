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
# Android性能优化
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
    * 对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管JVM的内存空间是否足够，都会回收该对象占用的内存。需要使用WeakReference来实现。
* 虚引用：
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
内存溢出是指应用程序试图分配比系统可用内存更多的内存空间。在 Android 中，每个应用程序都有一个特定的内存限制，这个限制取决于设备的硬件和操作系统版本。当应用程序尝试分配的内存超过这个限制时，就会发生内存溢出错误。在手机上，内存溢出错误通常会导致应用程序崩溃。在车机开发中，系统应用的内存大小如果没有限制，发生OOM甚至会直接导致系统重启。

可能的原因：

1. 频繁加载大图片
    ◦ 如果直接加载高分辨率的大图片而不进行适当的处理，会占用大量的内存。例如，加载一张 4000x3000 像素的图片，可能会消耗几十兆甚至上百兆的内存。
    ◦ 解决方案：可以使用图片加载库（如 Glide、Picasso 等），这些库通常会自动根据设备的屏幕尺寸和内存情况对图片进行缩放和缓存管理，以减少内存占用。
2. 过多的对象创建
    ◦ 在循环中频繁创建对象或者创建大量不必要的临时对象，会导致内存快速增长。比如在一个频繁调用的方法中不断创建新的字符串对象。
    ◦ 解决方案：尽量复用对象，避免不必要的对象创建。对于在循环中创建的对象，可以考虑在循环外部创建并重复使用。
3. 数据缓存不合理
    ◦ 如果缓存的数据过多或者没有及时清理过期的缓存，会占用大量内存。例如，一个网络请求缓存了大量的 JSON 数据，但没有设置缓存大小限制或过期时间。
    ◦ 解决方案：合理设置缓存大小和过期时间，定期清理不再需要的缓存数据。可以使用 LruCache 等缓存工具来管理内存缓存。
4. 内存泄漏
    ◦ 内存泄漏是指不再使用的对象仍然被其他对象引用，导致垃圾回收器无法回收它们的内存。随着时间的推移，内存泄漏会积累大量无法回收的内存，最终导致内存溢出。
    ◦ 解决方案：及时释放不再使用的资源，避免静态变量持有长生命周期对象的引用，注意 Activity、Service 等组件的生命周期管理，防止内存泄漏的发生。

解决方法：

1. 优化图片加载
    ◦ 如前所述，使用图片加载库来加载图片，并根据实际需求进行压缩和缩放。可以设置图片的采样率、质量等参数，以减少内存占用。
    ◦ 对于显示在列表或网格中的图片，可以使用异步加载和缓存机制，避免同时加载过多的图片。
2. 避免不必要的对象创建
    ◦ 检查代码中是否存在频繁创建临时对象的地方，可以通过复用对象、使用对象池等方式来减少对象创建的次数。
    ◦ 对于字符串操作，可以使用 StringBuilder 或 StringBuffer 来减少字符串对象的创建。
3. 合理管理缓存
    ◦ 根据应用的实际需求设置合适的缓存大小和过期时间。可以使用 LruCache、DiskLruCache 等缓存工具来管理内存和磁盘缓存。
    ◦ 定期清理不再需要的缓存数据，以释放内存空间。
4. 检测和修复内存泄漏
    ◦ 使用内存分析工具（如 Android Studio 的 Memory Profiler）来检测应用中的内存泄漏情况。这些工具可以帮助你找到哪些对象没有被正确释放，以及它们被哪些对象引用。
    ◦ 修复内存泄漏问题，通常需要注意对象的生命周期管理，避免不必要的引用和静态变量的滥用。
5. 优化数据结构和算法
    ◦ 选择合适的数据结构和算法可以减少内存占用。例如，使用稀疏数组代替密集数组可以节省大量内存空间。
    ◦ 对于大数据集的处理，可以考虑分页加载、流式处理等方式，避免一次性加载所有数据到内存中。

总之，在 Android 开发中，要避免内存溢出问题，需要注意合理使用内存资源，优化代码结构和算法，及时清理不再需要的资源，并使用合适的工具进行内存分析和优化。

## CPU优化
在 Android 应用中，CPU（中央处理器） 是执行所有计算任务的核心硬件。无论是用户交互、数据处理，还是系统后台任务，几乎所有的操作都需要依赖 CPU 来完成。

下面是对CPU比较敏感的操作

### UI 操作
UI 操作是用户直接感知的部分，虽然 Android 的 UI 渲染主要由 GPU 协助完成，但 CPU 仍然承担了大量的计算任务。
#### **布局测量与布局（Measure/Layout）**
* XML 布局解析：当加载一个 XML 布局文件时，系统需要解析 XML 并将其转换为视图树，这一过程由 CPU 完成。
* 视图测量与布局（Measure/Layout）：在视图树构建完成后，系统需要计算每个视图的大小和位置，这一过程也由 CPU 执行。
* 视图绘制（Draw）：虽然最终的像素填充由 GPU 完成，但绘制指令的生成（如 onDraw() 方法中的 Canvas 操作）是由 CPU 处理的。

CPU 占用高的原因可能有布局过于复杂（嵌套过深的视图树）。频繁调用 requestLayout() 或 invalidate()，导致视图反复测量和绘制。
#### **动画效果**
属性动画、补间动画和帧动画（如 AnimationDrawable）都需要 CPU 参与计算每一帧的状态。如果动画复杂度较高（如大量视图的联动动画），CPU 占用会显著增加。

CPU 占用高的原因可能为帧率过高（如 60fps），导致动画计算过于频繁。自定义属性动画在主线程中执行动画计算，导致主线程负担过重。
### 数据处理与计算
任何涉及数据处理的操作都需要 CPU 参与计算，尤其是在处理大量数据或复杂算法时，CPU 占用会显著增加。
#### **数据解析**
* JSON/XML 解析：从网络或本地文件中读取 JSON 或 XML 数据并解析为对象，这一过程需要 CPU 进行字符串处理和数据结构转换。
* Protobuf/FlatBuffer 解析：虽然这些格式的解析效率较高，但在数据量较大时仍然会占用 CPU。
* 网络数据的处理：在网络请求中，CPU 通常用于处理数据的序列化和反序列化。

#### **数据计算**
* 数学运算：如加密解密、图像处理、视频编解码、物理模拟等。
* 排序与搜索：对大量数据进行排序（如 Collections.sort()）或搜索（如二分查找、哈希表查询）。
* 数据转换：如图片缩放、颜色格式转换、音频采样率转换等。

### 多线程与并发操作
Android 应用中，开发者通常会使用多线程来执行耗时任务（如网络请求、文件读写、数据处理等），以避免阻塞主线程。然而，线程的创建、调度和同步也会占用 CPU 资源。

每个线程的创建和销毁都会消耗一定的 CPU 资源。如果创建了过多的线程（如没有使用线程池），线程调度会成为 CPU 的负担。

使用锁（如 synchronized、ReentrantLock）或其他同步机制（如 CountDownLatch、Semaphore）时，线程可能会因为等待锁而被挂起或唤醒，这一过程会占用 CPU。

CPU 占用高的原因：
* 线程数量过多，导致频繁的线程切换。
* 锁竞争激烈，线程频繁阻塞和唤醒。

### 后台任务与系统服务
Android 应用可能会在后台执行一些任务（如数据同步、日志上传、定时任务等），这些任务通常由系统服务或应用自带的线程池管理，但仍然需要 CPU 参与。

例如使用 AlarmManager、Handler 或 WorkManager 执行定时任务时，任务的执行逻辑会占用 CPU。

当应用接收到广播（如系统广播或自定义广播）时，注册的广播接收器会执行相应的回调逻辑，这一过程可能涉及 CPU 计算。

如果应用注册了传感器监听（如加速度传感器、陀螺仪），传感器的回调数据需要由 CPU 处理。

CPU 占用高的原因：
* 定时任务或广播接收器的逻辑过于复杂。
* 传感器数据采样频率过高，导致回调处理负担过重。

### 图片与多媒体处理
图片和多媒体处理是 CPU 占用较高的场景之一，尤其是在处理高分辨率图片或高清视频时。

使用 BitmapFactory 解码图片时，CPU 需要将原始字节数据转换为位图对象。如果图片分辨率过高（如几 MB 的图片），解码过程会非常耗时。

对图片进行缩放、裁剪、旋转等操作时，CPU 需要进行大量的像素计算。

播放或录制音频/视频时，编解码过程通常由 CPU 完成（除非使用了硬件加速）。

CPU 占用高的原因：
* 图片或视频分辨率过高。
* 没有使用硬件加速（如 MediaCodec 的硬件解码功能）。

### 优化 CPU 占用
针对上述场景，我们可以采取以下优化策略：
* 减少布局嵌套，使用 ConstraintLayout 等高效布局。避免频繁调用 requestLayout() 或 invalidate()。使用硬件加速（如开启 `setLayerType(View.LAYER_TYPE_HARDWARE, null)` ）来分担 CPU 的绘制压力。还有避免频繁计算复杂的自定义动画。
* 使用高效的算法和数据结构（如 HashMap 替代嵌套循环）。对大数据集进行分页加载，避免一次性处理过多数据。将数据处理任务移到子线程中执行，避免阻塞主线程。
* 避免频繁创建和销毁线程，合理使用线程池来管理线程，可以避免创建过多线程。减少锁的使用，避免线程竞争。使用无锁数据结构（如 ConcurrentHashMap）或异步编程模型（如 RxJava、Kotlin 协程）。
* 合并定时任务，减少任务执行的频率。使用 WorkManager 管理后台任务，避免重复执行。在广播接收器中只执行轻量级逻辑，耗时操作移到服务或线程中执行。
* 压缩图片分辨率，避免加载过大的图片。使用图片加载库（如 Glide、Picasso），它们会自动处理图片的解码和缓存。使用硬件加速的编解码器（如 MediaCodec）处理音视频。

## GPU优化
在 Android 应用中，GPU（图形处理单元） 主要负责图形渲染相关的任务，即**将 CPU 提交的绘制指令转化为屏幕上的像素**。

虽然 GPU 的主要职责是图形渲染，但在现代 Android 应用中，GPU 的使用场景已经不仅限于传统的图形绘制，还涉及到一些与图形相关的计算任务（如图像处理、视频渲染等）。

以下是 Android 应用中常见的会使用到 GPU 的操作。
### UI 渲染相关操作
UI 渲染是 GPU 最常见的使用场景，因为 Android 的界面是由大量的视图（View）组成的，而这些视图的绘制和显示需要 GPU 的参与。

Android 的视图系统通过 CPU 生成绘制指令（如 Canvas.drawXXX() 方法），然后将这些指令提交给 GPU 进行实际的像素填充。不管是绘制基本图形（如矩形、圆形、路径等），还是绘制文本，位图等。都是由 GPU 将 CPU 提交的绘制指令转化为屏幕上的像素。如果视图树过于复杂或绘制逻辑过于频繁，GPU 的负载会增加。
### 动画效果
动画的本质是每一帧的视图状态变化，而每一帧的状态变化需要通过 GPU 进行渲染。例如属性动画、补间动画、帧动画和自定义动画（如通过 ValueAnimator 实现的动画）。GPU 需要计算每一帧的像素变化并渲染到屏幕上。如果动画复杂度较高（如多个视图的联动动画）或帧率过高GPU 的负载会显著增加。
### 过度绘制（Overdraw）
过度绘制是指屏幕上的某些像素被多次绘制（如背景色、View 的背景、子 View 的背景等叠加绘制）。例如多层嵌套的背景色（如父布局和子布局都设置了背景色）。不可见的视图仍然被绘制（如 View.setVisibility(View.GONE) 的视图仍然被调用 draw() 方法）。过度绘制会增加 GPU 的负担，导致渲染性能下降。
### 图片与图像处理相关操作
图片加载、解码和显示是 Android 应用中常见的操作，这些操作通常需要 GPU 参与渲染，尤其是在处理高分辨率图片或复杂图像效果时。例如从网络或本地加载图片后，图片需要被解码为位图（Bitmap），然后通过 GPU 渲染到屏幕上。GPIU会将解码后的位图渲染到屏幕上。

如果图片分辨率过高（如几 MB 的图片），解码和渲染的负担会显著增加。

还有对图片进行缩放、裁剪、旋转等操作时，可能需要 GPU 参与像素计算。例如使用 Matrix 对图片进行变换（如缩放、旋转）。使用 Bitmap.createScaledBitmap() 对图片进行缩放。如果图片处理逻辑过于复杂，GPU 的负载会增加。

**图像滤镜与特效**

应用中可能还会使用一些图像滤镜或特效（如模糊、锐化、色彩调整等），这些操作通常需要对每个像素进行计算。
* 使用 OpenGL ES 或 Vulkan 实现自定义滤镜。
* 使用第三方库（如 GPUImage）实现图像特效。

图像滤镜和特效通常是计算密集型任务，会显著增加 GPU 的负载。
### 视频与多媒体相关操作
视频播放、录制和处理是 GPU 的重要使用场景，因为视频本质上是由大量的帧组成的，每一帧的解码、渲染和处理都需要 GPU 的参与。视频播放需要对每一帧进行解码和渲染，而 GPU 可以加速帧的渲染过程。
当我们使用使用 MediaPlayer 或 ExoPlayer 播放视频。使用 SurfaceView 或 TextureView 显示视频画面。GPU 会参与进来加速视频帧的渲染。

如果视频分辨率过高（如 4K 视频），或者播放过程中存在跳帧、卡顿，可能是 GPU 的负载过高。

视频录制需要对摄像头采集的每一帧进行处理和编码，而 GPU 可以加速帧的处理过程。例如使用 Camera2 API 或 CameraX 录制视频。或者使用 OpenGL ES 或 Vulkan 对视频帧进行实时处理（如滤镜、特效）。 GPU 会加速视频帧的处理和渲染。
### 游戏与高性能图形应用
游戏和高性能图形应用是 GPU 的主要使用场景，因为这些应用通常需要实时渲染大量的图形和动画。例如 2D 游戏需要实时渲染大量的精灵（Sprite）、背景、文字等图形元素。3D 游戏需要实时渲染复杂的三维模型、光影效果、粒子系统等。当我们使用 OpenGL ES 或 Vulkan 进行 3D 渲染。使用游戏引擎（如 Unity、Unreal Engine）进行 3D 游戏开发时，GPU 会加速三维模型的渲染、光影计算和粒子效果。
### 机器学习与图像处理
一些机器学习模型（如卷积神经网络）和图像处理算法（如目标检测、图像分割）可以利用 GPU 的并行计算能力加速。
* 使用 TensorFlow Lite 或 ML Kit 进行图像分类、目标检测等任务。
* 使用 GPU 加速的图像处理库（如 OpenCV + GPU 模块）。

GPU 主要用来加速矩阵运算和像素计算。机器学习和图像处理的计算量通常较大，对 GPU 的性能要求较高。
## 掉帧优化
Android 系统每隔 **16.67ms** 发出VSYNC信号，触发对UI进行渲染。如果某一帧的渲染时间超过 16.67ms，就会导致掉帧。例如，某一帧渲染耗时 33ms，就会导致掉 1 帧（因为 33ms > 16.67ms × 2）。

掉帧的表现形式包括：
* 界面卡顿、不流畅。
* 动画效果出现“跳帧”。
* 滑动列表时出现“拖影”或“延迟”。

掉帧可能是由于多种原因引起，CPU、GPU、内存等资源都可能成为瓶颈。

GPU 负责将 CPU 提交的绘制指令转化为屏幕上的像素。如果 GPU 的 **负载过高** ，或者 **渲染任务过于复杂** ，会导致帧渲染时间超过 16.67ms，从而引发掉帧。

内存问题可能间接导致掉帧，尤其是在内存不足时，系统会频繁进行内存回收，甚至触发 onTrimMemory() 回调，影响应用的性能。例如出现内存泄漏会导致应用的内存占用不断增加，最终触发频繁的垃圾回收（GC），从而影响主线程的执行。​表现​​为界面卡顿，尤其是在长时间运行后。如果设备的内存不足，系统可能会频繁进行内存回收，甚至杀死后台进程以释放内存。这会导致应用的性能下降。应用启动变慢，滑动列表时出现卡顿。

**系统层面** 

如果设备上有大量的后台任务（如其他应用的后台服务、系统更新等），会占用 CPU 和内存资源，影响当前应用的性能。尤其是在设备整体负载较高时，当前应用运行也会跟随变慢。

CPU 资源竞争也是一个因素，如果应用创建了过多的线程，或者线程调度不合理，会导致 CPU 资源竞争，影响主线程的执行。界面卡顿，尤其是在多线程任务较多的场景中。

出现掉帧问题的经典log：

```
"Skipped xx frames! The application may be doing too much work on its main thread"
```

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
