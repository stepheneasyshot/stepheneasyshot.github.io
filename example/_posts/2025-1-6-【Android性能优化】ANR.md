---
layout: post
description: > 
  本文介绍了Android应用性能优化的方法论和实例
image: 
  path: /assets/img/blog/blogs_anr_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_anr_cover.png
    960w:  /assets/img/blog/blogs_anr_cover.png
    480w:  /assets/img/blog/blogs_anr_cover.png
accent_image: /assets/img/blog/blogs_anr_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android性能优化】ANR
ANR是Android系统上，应用交互过程中可能出现的最差的体验了，它代表了应用程序已经经历了长时间无响应，并导致崩溃。

当应用程序的主线程被冻结，导致应用程序无法响应用户输入时，就会发生 ANR。届时，用户将面临一个 ANR 对话框，提示用户是等待还是强制关闭应用程序。

ANR本身是一套兜底机制，它监控Android应用响应是否及时。

我们可以把发生ANR比作是引爆炸弹，那么整个流程包含三部分组成：

1. 埋定时炸弹：中控系统(system_server进程)启动倒计时，在规定时间内如果目标(应用进程)没有干完所有的活，则中控系统会定向炸毁(杀进程)目标。
2. 拆炸弹：在规定的时间内干完工地的所有活，并及时向中控系统报告完成，请求解除定时炸弹，则幸免于难。
3. 引爆炸弹：中控系统立即封装现场，抓取快照，搜集目标执行慢的罪证(traces)，便于后续的案件侦破(调试分析)，最后是炸毁目标。

常见的ANR类型有service、broadcast、provider以及input.

各个ANR类型的超时时间阈值如下：

![](/assets/img/blog/blogs_android_anr_time.png)

可以看到，ANR的判定时间和前台后台强相关，差距很大，通常在前台的app会有更严格的限制。

那么Android系统是如何划分前台后台应用的？

## 前台后台
如果满足以下任一条件，则进程会被认为位于前台。
* 正在用户的互动屏幕上运行一个 Activity（其 onResume() 方法已被调用）。
* 有一个 BroadcastReceiver 目前正在运行（其 BroadcastReceiver.onReceive() 方法正在执行）
* 有一个 Service 目前正在执行其某个回调（Service.onCreate()、Service.onStart() 或 Service.onDestroy()）中的代码。

## 输入事件耗时
Google总结有如下几种常见的情况：

| 原因 | 出现的情况 | 建议的解决方法 |
| --- | --- | --- |
| binder 调用缓慢 | 主线程 binder 调用缓慢 | 将该调用移出主线程或尝试优化该调用（如果您拥有该 API）。 |
| 连续多次进行 binder 调用 | 主线程连续多次进行 binder 调用 | 请勿在紧密循环中执行 binder 调用。 |
| 阻塞 I/O | 主线程阻塞 I/O，例如数据库或网络访问。 | 将所有阻塞 IO 移出主线程。 |
| 锁争用 | 主线程处于阻塞状态，正在等待获取锁。 | 减少主线程和其他线程之间的锁争用。 优化其他线程中运行缓慢的代码。 |
| 耗用大量资源的帧 | 单个帧中的渲染工作量太大，导致严重卡顿。 | 减少帧渲染工作。请勿使用 n2 算法。使用高效的组件实现滚动或分页等操作，例如使用 Jetpack Paging 库。 |
| 被其他组件阻塞 | 另一个组件（如广播接收器）正在运行并阻塞主线程。 | 尽可能将非界面工作移出主线程。在其他线程上运行广播接收器。 |
| GPU 挂起 | GPU 挂起是系统或硬件问题，会导致渲染被阻塞，进而导致输入调度 ANR。 | 遗憾的是，通常无法在应用端解决这些问题。如有可能，请与硬件团队联系来排查问题。 |

例如：
#### System.Settings系统数据库的写操作
System.Settings 涉及到对系统级配置的修改，这些操作可能需要：
* 磁盘 I/O 操作： 写入设置需要将数据持久化到存储中。磁盘 I/O 可能是阻塞性的，尤其是在设备性能不佳、存储空间紧张或有其他高负载操作时。
* 进程间通信 (IPC)： System.Settings 的修改通常需要通过 Binder 机制与系统服务进行通信。如果系统服务繁忙或响应缓慢，主线程可能会被阻塞。
* 并发访问： 如果有多个线程或进程同时尝试写入或读取 System.Settings，可能会导致锁竞争，从而阻塞主线程。
* 权限检查： 写入某些 System.Settings 值需要特定的权限（例如 WRITE_SETTINGS），系统在执行操作前会进行权限验证，这也会占用一定时间。

#### 生命周期回调里做了耗时操作
Android 的生命周期函数（如 onCreate()、onResume()、onPause()、onDestroy() 等）都是在主线程上调用的。它们的职责是快速完成 UI 初始化、数据加载、状态保存等轻量级任务，以便应用能够迅速响应用户操作并呈现界面。如果在这些函数中执行以下类型的耗时操作，就会阻塞主线程，导致 ANR。

常见的可能导致ANR的操作：

##### 1\. `onCreate()` 中进行耗时操作

  * **场景：** 在 `onCreate()` 中加载大量数据、进行复杂的数据库查询、或执行网络请求。
  * **ANR 原因：** 应用启动时，`onCreate()` 需要快速完成，才能显示第一个界面。如果耗时过长，用户将看到黑屏或卡顿，并最终收到 ANR 提示。
  * **解决方案：**
      * **数据加载和网络请求：** 将这些操作移动到**后台线程**中执行。
          * **Kotlin Coroutines (协程)：** 推荐使用，利用 `Dispatchers.IO` 或 `Dispatchers.Default`。
          * **Java `ExecutorService` / `Thread`：** 手动管理线程池。
          * **Android Architecture Components (如 Room, ViewModel)：** 配合 `LiveData` 或 `Flow`，在 `ViewModel` 中处理数据逻辑，然后在 UI 线程观察数据变化。
      * **UI 初始化：** 仅在 `onCreate()` 中进行必要的 UI 视图膨胀和组件绑定。

##### 2\. `onResume()` 中进行耗时操作

  * **场景：** 在 `onResume()` 中刷新大量数据、注册耗时监听器。
  * **ANR 原因：** 当 Activity 从后台回到前台，或从部分遮盖状态恢复时，会调用 `onResume()`。如果这里有耗时操作，用户会感到界面卡顿，无法立即与应用交互。
  * **解决方案：**
      * 同 `onCreate()`，将耗时操作放到**后台线程**。
      * 考虑使用 **懒加载 (Lazy Loading)** 或**按需加载**策略，只加载屏幕可见部分的数据。

##### 3\. `onPause()` / `onStop()` 中进行耗时操作

  * **场景：** 在 `onPause()` 或 `onStop()` 中保存大量数据到磁盘、执行复杂的数据库事务。
  * **ANR 原因：** 当用户离开当前 Activity (例如，按下 Home 键、切换到其他应用、或者启动新的 Activity) 时，系统会调用 `onPause()` 和 `onStop()`。这些方法需要迅速完成，以便系统能够释放资源或切换到其他应用。如果耗时过长，系统可能认为当前应用卡死，导致 ANR。
  * **解决方案：**
      * **数据保存：** 将耗时的数据持久化操作（如数据库写入、文件写入）移至**后台线程**。
      * **小量数据：** 对于少量非关键数据，可以使用 `SharedPreferences.apply()` (异步写入) 而不是 `SharedPreferences.commit()` (同步写入)。
      * **复杂的保存逻辑：** 考虑使用 `WorkManager` 来调度后台任务进行数据同步或上传。
      * **注意：** 尽管 `onPause()` 和 `onStop()` 应该快速完成，但它们是保存用户状态的关键时机。务必确保重要数据的保存，即使将其推迟到后台线程，也要确保任务的可靠性。

##### 4\. `onDestroy()` 中进行耗时操作

  * **场景：** 在 `onDestroy()` 中释放大量资源、关闭文件句柄、清理缓存等。
  * **ANR 原因：** 当 Activity 被销毁时调用。虽然此时应用可能即将退出，但如果 `onDestroy()` 阻塞，也可能导致系统资源长时间不释放，甚至在特定情况下触发 ANR。
  * **解决方案：**
      * **资源释放：** 大多数资源释放（如 `MediaPlayer.release()`、大图片 Bitmap 释放）可以放在主线程，但如果涉及到大量文件 I/O 或网络断开连接的阻塞，**仍应考虑放在后台线程**。
      * **清理工作：** 确保清理工作简洁高效。

## 广播接收器超时
接收到广播之后，如果在一定时间内没有执行完onReceive，也会被判定为ANR。

![](/assets/img/blog/blogs_anr_broadcastreceiver.png)

##### **goAsync的作用**
广播接收器goAsync()的用处，简单说就是手动地拖延onReceive执行的时间到子线程结束后。

所以使用的时机就是我们需要在接收到广播之后，开子线程处理耗时任务的时候。广播接收器接收到广播后，开始执行onReceive的方法，这时候进程是前台状态，一旦走完，又会恢复到后台的状态。如果在onReceive回调里直接开子线程，那么onReceive走完后，进程优先级较低，其内的线程优先级也较低，可能任务没有执行完就结束了。分析onReceive源码，可以看到在其结束时，会检查 PendingResult 的状态，如果不为空就表明任务执行完毕。也就恢复到了后台状态。

goAsync方法就是将 PendingResult设置为 null，也就不会马上结束掉当前的广播，相当于 “延长了广播的生命周期”，让广播依然处于活跃状态。在子线程的任务执行完毕，再调用一次 PendingResult.finish()，结束onReceive方法的计时。

所以广播接收器ANR的情况就是onReceive方法超时，或者goAsync方法调用完之后，超时时间内没有调用finish。

## Service执行超时
**onCreate()，onStartCommand()，onBind()**等生命周期在20s内没有处理完成，就会发生ANR。

## ANR日志分析
原生位置：/data/anr/anr_2024-08-19-17-16-58-475，定制化的系统对日志输出有优化则为其自定的位置。

文件首行会表明ANR的类型，以下为demo故意触发ANR的例子。

输入事件超时未处理

```
Subject: Input dispatching timed out 。。。。
```
广播接收器处理超时

```
Subject: Broadcast of Intent { act=android.intent.action.SCREEN_ON flg=0x50200010 }
```

Service超时

```
Subject: executing service com.stephen.commondemo/.anr.AnrService
```

ContentProvider超时

```
日志关键字：timeout publishing content providers
```

### 分析步骤
1. 首先我们搜索am_anr，找到出现ANR的时间点、进程PID、ANR类型、然后再找搜索PID，找前5秒左右的日志。
2. 过滤ANR IN 查看CPU信息
3. 接着查看traces.txt，找到java的堆栈信息定位代码位置，最后查看源码，分析与解决问题。

分析举例

#### 场景一

```
07-20 15:36:36.472  1000  1520  1597 I am_anr  : [0,1480,com.xxxx.moblie,952680005,Input dispatching timed out (AppWindowToken{da8f666 token=Token{5501f51 ActivityRecord{15c5c78 u0 com.xxxx.moblie/.ui.MainActivity t3862}}}, Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)]
```

ANR时间：07-20 15:36:36.472
进程pid：1480
进程名：com.xxxx.moblie
ANR类型：KeyDispatchTimeout

我们已经知道了发生KeyDispatchTimeout的ANR是因为 input事件在5秒内没有处理完成。那么在这个时间07-20 15:36:36.472 的前5秒，也就是（15:36:30 ~15:36:31）时间段左右程序到底做了什么事情？

#### 场景二

```
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: ANR in com.xxxx.moblie (com.xxxx.moblie/.ui.MainActivity) (关键字ANR in + 进程名 + Activity名称)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: PID: 1480 (进程pid)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: Reason: Input dispatching timed out (AppWindowToken{da8f666 token=Token{5501f51 ActivityRecord{15c5c78 u0 com.xxxx.moblie/.ui.MainActivity t3862}}}, Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)（ANR的原因，输入分发超时）
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: Load: 0.0 / 0.0 / 0.0 (Load表明是1分钟,5分钟,15分钟CPU的负载)
07-20 15:36:58.711  1000  1520  1597 E ActivityManager: CPU usage from 20ms to 20286ms later (2018-07-20 15:36:36.170 to 2018-07-20 15:36:56.436):
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   42% 6774/pressure: 41% user + 1.4% kernel / faults: 168 minor
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   34% 142/kswapd0: 0% user + 34% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   31% 1520/system_server: 13% user + 18% kernel / faults: 58724 minor 1585 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   13% 29901/com.ss.android.article.news: 7.7% user + 6% kernel / faults: 56007 minor 2446 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   13% 32638/com.android.quicksearchbox: 9.4% user + 3.8% kernel / faults: 48999 minor 1540 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   11% (CPU的使用率)1480/com.xxxx.moblie: 5.2%(用户态的使用率) user + (内核态的使用率) 6.3% kernel / faults: 76401 minor 2422 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   8.2% 21000/kworker/u16:12: 0% user + 8.2% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   0.8% 724/mtd: 0% user + 0.8% kernel / faults: 1561 minor 9 major
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   8% 29704/kworker/u16:8: 0% user + 8% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   7.9% 24391/kworker/u16:18: 0% user + 7.9% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   7.1% 30656/kworker/u16:14: 0% user + 7.1% kernel
07-20 15:36:58.711  1000  1520  1597 E ActivityManager:   7.1% 9998/kworker/u16:4: 0% user + 7.1% kernel
```

通过上面所提供的案例我们可以分析出以下几点：
* ANR发生的位置是：com.xxxx.moblie/.ui.MainActivity
* com.xxxx.moblie 占用了11%的CPU，CPU的使用率并不是很高，基本可以排除CPU负载的原因
* Reason提示我们是输入分发超时导致的ANR
通过上面几点我们虽然排除了CPU过度负载的可能，但我们并不能准确定位出ANR的确切位置，要想准确定位出ANR发生的确切位置，就要借助系统为了解决ANR问题而提供的终极大杀器——traces.txt文件了。
找到anr目录下的trace.txt

```
trace:
Cmd line:com.xxxx.moblie
 
"main" prio=5 tid=1 Runnable
  | group="main" sCount=0 dsCount=0 obj=0x73bcc7d0 self=0x7f20814c00
  | sysTid=20176 nice=-10 cgrp=default sched=0/0 handle=0x7f251349b0
  | state=R schedstat=( 0 0 0 ) utm=12 stm=3 core=5 HZ=100
  | stack=0x7fdb75e000-0x7fdb760000 stackSize=8MB
  | held mutexes= "mutator lock"(shared held)
  // java 堆栈调用信息,可以查看调用的关系，定位到具体位置
  at ttt.push.InterceptorProxy.addMiuiApplication(InterceptorProxy.java:77)
  at ttt.push.InterceptorProxy.create(InterceptorProxy.java:59)
  at android.app.Activity.onCreate(Activity.java:1041)
  at miui.app.Activity.onCreate(SourceFile:47)
  at com.xxxx.moblie.ui.b.onCreate(SourceFile:172)
  at com.xxxx.moblie.ui.MainActivity.onCreate(SourceFile:68)
  at android.app.Activity.performCreate(Activity.java:7050)
  at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1214)
  at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2807)
  at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2929)
  at android.app.ActivityThread.-wrap11(ActivityThread.java:-1)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1618)
  at android.os.Handler.dispatchMessage(Handler.java:105)
  at android.os.Looper.loop(Looper.java:171)
  at android.app.ActivityThread.main(ActivityThread.java:6699)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:246)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:783)
```

这里详细解析一下traces.txt里面的一些字段，看看它到底能给我们提供什么信息.

* main：main标识是主线程，如果是线程，那么命名成“Thread-X”的格式,x表示线程id,逐步递增。
* prio：线程优先级,默认是5
* tid：tid不是线程的id，是线程唯一标识ID
* group：是线程组名称
* sCount：该线程被挂起的次数
* dsCount：是线程被调试器挂起的次数
* obj：对象地址
* self：该线程Native的地址
* sysTid：是线程号(主线程的线程号和进程号相同)
* nice：是线程的调度优先级
* sched：分别标志了线程的调度策略和优先级
* cgrp：调度归属组
* handle：线程处理函数的地址。
* state：是调度状态
* schedstat：从 /proc/[pid]/task/[tid]/schedstat读出，三个值分别表示线程在cpu上执行的时间、线程的等待时间和线程执行的时间片长度，不支持这项信息的三个值都是0；
* utm：是线程用户态下使用的时间值(单位是jiffies)
* stm：是内核态下的调度时间值
* core：是最后执行这个线程的cpu核的序号。

Java的堆栈信息是我们最关心的，它能够定位到具体位置。从上面的traces,我们可以判断ttt.push.InterceptorProxy.addMiuiApplicationInterceptorProxy.java:77 导致了com.xxxx.moblie发生了ANR。这时候可以对着源码查看，找到出问题，并且解决它。

