---
layout: post
description: > 
  本文介绍了Android应用性能测试的手段和指标
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
# 【Android性能优化】性能测试指标
## 奶酪模型
从产品设计到上线，每一个流程都像一片奶酪，令人不愉快的bug就像正好穿透了每一片奶酪的孔，到达了用户那里。比如开发逻辑考虑不全，测试漏测，环境不一致，验收不严格，发布的人员配置错了包。。。

做性能问题跟进时，要多往前一步，加强各个环节的管理，尽可能早的捕获异常。

## 测试项目
### CPU
在启动，卡顿和功耗测试时，需要做CPU的相关测试。

**获取CPU核心数量**

```
adb shell cat /sys/devices/system/cpu/present
```

**获取cpu最大频率**

```
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
```

**获取cpu当前频率**

```
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

**获取cpu使用时间**

```
C:\Users\stephen>adb shell cat proc/stat
       user   nice   system   idle     iowait  irq   softirq
cpu  11570106 908753 10084803 182776887 83576 1710507 375950 0 0 0
cpu0 2083516 133403 2103739 21423628 10242 435957 101041 0 0 0
cpu1 2159620 138975 2070350 21482805 9850 409451 66651 0 0 0
cpu2 2079859 134242 2058410 21601693 9440 391933 66413 0 0 0
cpu3 2058259 133264 2049679 21604397 9473 377789 123115 0 0 0
cpu4 1117565 193035 979789 24387557 20360 45444 9989 0 0 0
cpu5 689868 54942 326106 25660003 10828 21026 3717 0 0 0
cpu6 681939 55379 321498 25672120 10775 20727 3652 0 0 0
cpu7 699477 65509 175229 20944681 2604 8177 1368 0 0 0
intr 854517694 0 455381193 23874980 0 0 0 51669558 0 0 0 0 207557250 0 0 0 0 0 914 1197 2 1 40218 0 0 0 0 0 0 12476206 0 0 5793575 126636 42212 0 0 18958468 169317 4351606 1895070 396 65628 2311956 0 0 0 0 0 0 0 0 0 0 0 0 0 35577 0 0 0 0 0 0 0 0 0 0 0 0 269992 448 0 1572833 0 0 0 0 0 0 0 0 0 0 0 0 9075516 238492 93961 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 162631 66441 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 11059235 0 0 0 0 0 0 0 0 801 491 136 26 440 0 3062479 367011 0 18325 0 35584 207672 0 0 0 0 171047 106114 1054921 616439 0 0 35780 34946 254079 3376 0 9909125 3318728 0 0 5 422 6 0 4 20536 4 4 9814 0 839 255 780289 325 13994 0 0 0 0 0 0 18 60 0 0 0 0 0 0 0 0 0 0 0 0 278 418 1933 0 0 0 0 84 0 0 3 2510466 46 136 133710 251287 66 390 2800222 385903 1408 0 0 0 0 0 7154 1538107 670023 937725 1422598 1489379 0 114654 0 0 0 0 0 0 0 0 0 0 0 0 0 1 1 0 0 0 1 1 0 0 0 0 1 1 0 0 1 1 0 0 15 19901 3343 256936 52 0 276656 0 0 0 206182 0 699076 3231 0 0 1289340 7315827 0 0 400 66 17 0 0 0 10 1 1 0 6 0 0 11 4810173 54848
ctxt 1403823113
btime 1733724528
processes 879090
procs_running 1
procs_blocked 0
softirq 142741039 18220587 27876585 91010 10189720 5052564 0 3964375 24884673 501992 51959533
```

上面的数据打印，第一行有7个字段描述：
* user:用户态时间
* nice:通过nice修改优先级之后的进程的用户态时间
* sysetm:内核态时间
* idle:空闲时间
* iwait:等待IO完成的时间
* irq:硬件中断的时间
* softirq:软件中断的时间

数据单位是jiffies，表示时钟中断次数，一般为1/100s

#### **测试应用CPU**
对一个 Android 应用进行 CPU 测试，包括 CPU 占用率、CPU 密集型任务的性能、线程使用情况等。通过 CPU 测试，可以发现应用是否存在 CPU 过载、线程阻塞、死锁 或 性能瓶颈 等问题。

可以使用如下方法：
##### **1. 使用 Android Studio Profiler 进行 CPU 测试**
Android Studio 内部集成了 Profiler 工具，可以帮助开发者实时监控应用的 CPU 使用情况、方法调用耗时、线程状态等信息。它是Systrace的升级版本，提供了更全面的性能分析功能。

具体的，打开 Android Studio，连接设备或启动模拟器。
* 在顶部菜单栏选择 View > Tool Windows > Profiler。
* 选择要测试的应用进程。

在 Profiler 中，切换到 CPU 标签页，可以选择以下两种分析模式：
* Sample Java Methods（采样 Java 方法）：通过采样方式记录方法的调用情况，适合分析 CPU 密集型任务的性能。对应用性能影响较小，适合长时间测试。
* Trace Java Methods（跟踪 Java 方法）：记录每个方法的调用耗时，适合分析具体方法的性能瓶颈。对应用性能影响较大，适合短时间测试。

点击 Record 按钮，开始记录 CPU 使用情况。在应用中执行目标操作（如启动页面、滑动列表、点击按钮等）。操作完成后，点击 Stop 按钮，停止记录。

Profiler 会生成 CPU 使用情况的图表和调用树（Call Chart），包括：
* CPU 使用率：应用在测试期间的 CPU 占用情况。
* 调用树（Call Chart）：显示方法的调用关系和耗时情况。
* 火焰图（Flame Chart）：以可视化的方式展示方法调用的堆栈信息，帮助快速定位性能瓶颈。
* 线程活动：显示各个线程的活动情况，帮助分析是否存在线程阻塞或死锁。

##### **2. 命令行工具**
`adb shell top`：查看设备的 CPU 使用情况，包括应用的 CPU 占用率。

`adb shell dumpsys cpuinfo`：查看应用的 CPU 使用统计信息。

`adb shell pidstat`（需要安装 sysstat 工具）：更详细地查看进程的 CPU 使用情况。

##### **3. Perfetto**
Perfetto 是 Android 平台上的性能分析工具，可以帮助开发者分析应用的 CPU 使用情况、线程状态、方法调用耗时等。

在终端运行以下命令，启动 Perfetto 数据采集：

```
adb shell perfetto --txt -c /data/misc/perfetto-configs/trace_config.pbtxt -o /data/misc/perfetto-traces/trace.perfetto-trace
```

需要提前配置 trace_config.pbtxt 文件，指定要采集的数据类型（如 CPU、内存、线程等）。在数据采集期间，在设备上执行目标操作。停止数据采集后，将生成的 .perfetto-trace 文件导出到电脑：

```
adb pull /data/misc/perfetto-traces/trace.perfetto-trace
```

打开 https://ui.perfetto.dev/ ，上传 `perfetto-trace` 文件进行分析。

可以查看 CPU 使用情况、线程状态、方法调用耗时等详细信息。
### GPU
GPU测试对于Android手机的性能评估、优化、兼容性检查和故障诊断等方面都具有重要意义。它有助于提高手机的整体性能和用户体验，同时也为开发者提供了优化应用程序的依据。

通过GPU测试，可以了解手机GPU的性能表现，包括图形处理能力、渲染速度、帧率等。确保GPU与操作系统、驱动程序和各种应用程序之间的兼容性。

**获取GPU类型**

```
dumpsys SurfaceFlinger | grep GLES
 ------------RE GLES------------
GLES: Qualcomm, Adreno (TM) 730, OpenGL ES 3.2 V@0615.73 (GIT@8f5499ec14, Ie6ef1a0a80, 1689341690) (Date:07/14/23)
```

**gpubusy** 
这是一个与 GPU 使用率相关的信息。在 Android 系统中，gpubusy 通常指的是 GPU 繁忙程度的指标，它表示 GPU 在某个时间段内处于忙碌状态的时间比例。这个指标可以帮助开发者了解 GPU 的负载情况，以便优化图形渲染性能。

```
adb shell cat /sys/class/kgsl/kgsl-3d0/gpubusy
```

**gpuclk** 
通常指的是 GPU 时钟频率（GPU Clock Frequency）。GPU 时钟频率是指 GPU 芯片内部的时钟信号的频率，它决定了 GPU 每秒钟能够执行的操作次数。GPU 时钟频率越高，GPU 的性能通常就越强，但同时也会消耗更多的电力并产生更多的热量。

```
adb shell cat /sys/class/kgsl/kgsl-3d0/gpuclk
```

**联发科平台**

```
adb shell cat sys/kernel/debug/ged/hal/gpu_utilization
adb shell cat sys/kernel/debug/ged/hal/current_frequency
```

### FPS
卡顿测试时的测试数据。

* FPS frames，在数据获取的周期内，用实际绘制帧数除以时间间隔所得
* Skipped frames，表示掉帧数，在数据时间周期内实际掉帧数量
* Janky frames，掉帧率，实际掉帧数量除以实际绘制数可得

使用下面这个命令计算单个app的卡顿信息，这里面信息很多，主要有四个部分。

* 卡顿统计信息
* 内存占用信息
* 绘制一帧各个阶段的时间
* 布局层级和总布局数

```
redfin:/ # dumpsys gfxinfo com.stephen.redfindemo framestats
```

#### 卡顿统计数据

```
** Graphics info for pid 7206 [com.stephen.redfindemo] **

Stats since: 992991790325ns
Total frames rendered: 84
Janky frames: 4 (4.76%)
Janky frames (legacy): 6 (7.14%)
50th percentile: 5ms
90th percentile: 10ms
95th percentile: 20ms
99th percentile: 105ms
Number Missed Vsync: 1
Number High input latency: 29
Number Slow UI thread: 4
Number Slow bitmap uploads: 0
Number Slow issue draw commands: 3
Number Frame deadline missed: 4
Number Frame deadline missed (legacy): 3
```

#### 绘制相关占用的内存

```
CPU Caches:
  Glyph Cache: 37.14 KB (1 entry)
  Glyph Count: 6
Total CPU memory usage:
  38034 bytes, 37.14 KB (0.00 bytes is purgeable)
GPU Caches:
  Other:
    Other: 7.90 KB (1 entry)
  Image:
    Texture: 10.57 MB (7 entries)
  Scratch:
    Texture: 2.00 MB (1 entry)
    Buffer Object: 48.00 KB (1 entry)
Total GPU memory usage:
  13240552 bytes, 12.63 MB (10.57 MB is purgeable)
```

#### 绘制一帧各阶段时间图

```
---PROFILEDATA---
Flags,FrameTimelineVsyncId,IntendedVsync,Vsync,InputEventId,HandleInputStart,AnimationStart,PerformTraversalsStart,DrawStart,FrameDeadline,FrameInterval,FrameStartTime,SyncQueued,SyncStart,IssueDrawCommandsStart,SwapBuffers,FrameCompleted,DequeueBufferDuration,QueueBufferDuration,GpuCompleted,SwapBuffersCompleted,DisplayPresentTime,
1,3323,993103720308,993103720308,0,993103921638,993103922419,993103923096,993185780292,993124220308,993103919554,11111111,

...

993188207896,993188438677,993190030396,993207265294,993210785034,302969,728802
---PROFILEDATA---
```

#### 布局层级和总布局数

```
View hierarchy:

  com.stephen.redfindemo/com.stephen.redfindemo.feature.main.MainActivity/android.view.ViewRootImpl@14a75f8
  68 views, 115.76 kB of render nodes

  /android.view.ViewRootImpl@fc532c0
  74 views, 120.03 kB of render nodes


Total ViewRootImpl   : 2
Total attached Views : 142
Total RenderNode     : 235.79 kB (used) / 732.03 kB (capacity)
```

### 文件读写
启动速度和卡顿测试，还要关注文件读写情况。

**获取pid**

```
adb shell pidof packageName
```

**获取进程的文件读写数据**

```
redfin:/ # cat /proc/2866/io
rchar: 197231
wchar: 3874
syscr: 40
syscw: 48
read_bytes: 9613312
write_bytes: 0
cancelled_write_bytes: 0
```

可以获取到读取的总字节数，通过一定时间的差值，就可以计算出改进程读写字节数的增量。

### Layout Inspector 
Layout Inspector 是 Android Studio 提供的一个强大工具，用于查看和分析 Android 应用程序的布局层级。

#### 捕获布局快照
点击 Layout Inspector 窗口中的 Capture New Snapshot 按钮（一个相机图标）。
Layout Inspector 会捕获当前应用程序的布局快照，并显示在窗口中。
#### 查看布局层级
在 Layout Inspector 窗口的左侧，你会看到布局的层级结构。
点击层级结构中的节点，可以在右侧的 Properties 窗口中查看该视图的详细属性。
你还可以在 Layout Inspector 窗口的中间部分查看布局的可视化表示。
#### 分析布局性能
在 Layout Inspector 窗口的右上角，有一些工具按钮，如 Show Layout Bounds、Show System UI 等。
使用这些工具可以帮助你分析布局的性能，例如查看布局边界、隐藏系统 UI 等。
#### 保存和分享布局快照
在 Layout Inspector 窗口中，点击菜单栏的 File -> Save As 来保存当前的布局快照为一个文件。
你还可以点击 File -> Export to Bitmap 来将布局快照导出为一个图片，以便与他人分享或用于文档中。

### uptime
uptime通常指的是设备自上次重启以来已经运行的时间。
```
redfin:/ # uptime
 21:26:37 up 33 min,  0 users,  load average: 2.42, 2.24, 2.02
```

### top
在Android系统中，top命令用于实时显示系统中各个进程的资源占用情况，包括CPU、内存等。top命令输出的每一列代表的含义如下：

* PID：进程ID（Process ID），每个进程都有一个唯一的ID。
* USER：进程所属的用户。
* PR：进程的优先级（Priority）。
* NI：进程的Nice值，用于调整进程的优先级。
* VIRT：进程使用的虚拟内存大小。
* RES：进程使用的物理内存大小（Resident Set Size），即实际占用的内存。
* SHR：进程使用的共享内存大小。
* S：进程的状态（Status），包括R（运行）、S（睡眠）、D（不可中断睡眠）、Z（僵尸）等。
* %CPU：进程占用的CPU百分比。
* %MEM：进程占用的内存百分比。
* TIME+：进程自启动以来占用的CPU时间，单位为秒。
* COMMAND：进程的命令名或启动命令。

例如，以下是top命令的输出示例：

```
Tasks: 885 total,   1 running, 884 sleeping,   0 stopped,   0 zombie
  Mem:    11072M total,    10758M used,      314M free,        5M buffers
 Swap:     4095M total,     3130M used,      965M free,     4686M cached
800%cpu  15%user   0%nice  23%sys 758%idle   0%iow   4%irq   1%sirq   0%host
  PID USER         PR  NI VIRT  RES  SHR S[%CPU] %MEM     TIME+ ARGS
 8460 u0_a417      20   0  41G 849M 308M S 22.6   7.6  23:58.65 com.netease.cloudmusic
 8803 u0_a417      16  -4  22G 299M 188M S 10.6   2.6   7:54.15 com.netease.cloudmusic:play
 1494 system       20   0  12G  33M  22M S  7.0   0.2 375:26.62 surfaceflinger
18551 shell        20   0  12G 6.0M 4.0M R  2.3   0.0   0:00.24 top
 9643 u0_a417      20   0  20G 170M 119M S  2.3   1.5   1:46.99 com.netease.cloudmusic:pushservice
27119 root         20   0    0    0    0 I  2.0   0.0   1:55.01 [kworker/u16:2-bwmon_wq]
16823 root         20   0    0    0    0 I  1.6   0.0   0:07.72 [kworker/u16:11-memlat_wq]
 5852 u0_a232      20   0  20G 118M  82M S  1.0   1.0  39:20.27 com.sonymobile.gameenhancer
  316 root         RT   0    0    0    0 S  0.6   0.0  24:06.41 [irq/38-190b6400]
   14 root         20   0    0    0    0 S  0.6   0.0  35:29.60 [rcuog/0]
16817 root         20   0    0    0    0 I  0.3   0.0   0:12.96 [kworker/u16:4-bwmon_wq]
14833 root         20   0    0    0    0 I  0.3   0.0   0:00.54 [kworker/4:3-mm_percpu_wq]
14414 root         20   0    0    0    0 I  0.3   0.0   0:01.00 [kworker/3:0-mm_percpu_wq]
 7476 u0_a422      16  -4  23G 120M  76M S  0.3   1.0   4:31.25 com.tencent.wetype
25566 u0_a422      10 -10  30G 203M 110M S  0.3   1.8  11:58.42 com.tencent.wetype:hld
 4513 root         20   0  12G 2.7M 2.6M S  0.3   0.0   6:53.32 msm_irqbalance -f /system/vendor/etc/msm_irqbalance.conf
 3231 network_sta+ 20   0  19G  94M  54M S  0.3   0.8   5:16.52 com.android.networkstack.process
 1718 system       20   0  12G 3.3M 3.2M S  0.3   0.0   2:20.69 charge_service
```

## 应用启动速度测试
### 三种启动类型
#### 冷启动
设备刚开机，或者应用被杀死后，再次打开应用的场景。
在冷启动开始时，系统有以下三项任务：

1. 加载并启动应用。
1. 在启动后立即显示应用的空白启动窗口。
1. 创建应用进程。

系统一创建应用进程，应用进程就负责后续阶段：
1. 创建应用对象。
1. 启动主线程。
1. 创建主 activity。
1. 膨胀视图。
1. 创建屏幕布局。
1. 执行初步绘制。

当应用进程完成第一次绘制时，系统进程就会换掉显示的后台窗口，将其替换为主 activity。此时，用户可以开始使用应用。

**application创建**

进程生成后，到Application创建，执行完onCreate方法，这个方法一般执行全局配置和第三方库的初始化，也是冷启动优化的重点目标之一。执行完后，即AMS的bindApplication方法走完，开始创建主线程，准备进入到Activity的流程。

**activity 创建**

在应用进程创建 activity 后，activity 将执行以下操作：

1. 初始化值。
1. 调用构造函数。
1. 根据 activity 的当前生命周期状态，相应地调用回调方法，如 
Activity.onCreate()。

通常，onCreate() 方法对加载时间的影响最大，因为它执行工作的开销最高：加载和膨胀视图，以及初始化运行 activity 所需的对象。

#### 温启动
温启动，比如在退出应用后又重新启动应用。进程可能继续运行，但应用必须通过调用 onCreate() 从头开始重新创建 activity。
或者系统将您的应用从内存中逐出，然后用户又重新启动它。进程和 activity 需要重启，但传递到 onCreate() 的已保存实例 state bundle 对于完成此任务有一定助益。

#### 热启动
Activity还在后台，如果应用的所有 activity 仍驻留在内存中，则应用可以避免重复执行对象初始化、布局膨胀和呈现。
但是，如果一些内存为响应内存整理事件（如 onTrimMemory()）而被完全清除，则需要为了响应热启动事件而重新创建相应的对象。

![start_up](/assets/img/blog/blogs_startup.png){:height="200" width="400" load="lazy"}

### 服务类app添加窗口View
在Android中,将View初次添加到Window，之后再次添加的主要区别在于它们发生的时机和可能的影响。

#### 冷启动-初次添加
当View第一次被添加到Window时,它会经历完整的布局和绘制刷流程这包括测量、布局和绘制阶段。

初次添加View时,系统会为其分配一个唯一的Window ID,并将其放置在Window的视图层次结构中。
#### 热启动-再次添加
如果View已经被添加到Window，然后被移除。例如，通过调用removeView()或ViewGone()，再次添加它时，系统可能会尝试重用之前的Window ID。

再次添加View时，它可能不会经历完整的布局和绘制流程，特别是如果它的尺寸和位置没有改变。系统可能会尝试优化性能，只进行必要的更新。

如果View的状态(如可见性、尺寸、位置等)在移除和再次添加之间发生了变化，系统会相应地更新这些状态。

### 启动时间指标
Android 使用初步显示所用时间 (TTID) 和完全显示所用时间 (TTFD) 指标来优化冷应用启动和温应用启动。Android 运行时 (ART) 使用这些指标的数据来高效地预编译代码，以优化未来启动。

更快的启动速度可以促进用户与应用的持续互动，从而减少过早退出、重启实例或前往其他应用的情况。

#### TTID指标
获取初步显示时间TTID，直接在logcat中搜索"Displayed"：显示为1s470ms。

注意：在所有资源加载并显示之前，Logcat 输出中的 Displayed 指标不一定会捕获时间。它会省去布局文件中未引用的资源或被应用作为对象初始化一部分创建的资源。它之所以排除这些资源，是因为加载它们是一个内嵌进程，并且不会阻止应用的初步显示。

有时候，打印后面还有有一个附加的字段：

```
ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms (total +1m22s643ms)
```

total 时间测量值是从应用进程启动时开始计算，并且可以包含首次启动但未在屏幕上显示任何内容的另一个 activity。total 时间测量值仅在单个 activity 的时间和总启动时间之间存在差异时才会显示。

#### TTFD指标
如果有其他的异步操作影响了界面交互，需要在所有控件及数据状态加载完毕，确认可交互状态时，主动调用 reportFullyDrawn方法，以获取最高可达 TTFD 的信息。例如测试Demo中，填入一个长度为1000的recyclerView，完全显示后，主动调用此方法，打印出来的时间为 `2s728ms` ，比上面看的TTID要长不少。

填列表的代码如下：

```kotlin
 MainScope().launch {
            val testList = mutableListOf<String>()
            repeat(1000) {
                delay(1L)
                testList.add(it.toString())
            }
            binding.rvTestteste.apply {
                layoutManager =
                    LinearLayoutManager(this@MainActivity, LinearLayoutManager.VERTICAL, false)
                adapter = SimpleAdapter(testList)
            }
            reportFullyDrawn()
        }
```

### 在trace文件中查指标数据
#### 抓取trace
Android 9开始，系统内默认预制了Perfetto，但是需要手动开启。

```bash
adb shell

setprop persist.traced.enable 1
```

在shell下执行抓取命令，一般只抓取对应单个流程的trace数据，时间10s左右的。

```bash
perfetto -o /data/misc/perfetto-traces/trace_log -t 12s -b 100mb -s 150mb sched freq idle am wm gfx view input
```

参数说明：

* -o trace文件输出路径
* -t 抓取trace的时间
* -b buffer大小
* 追加tags 抓哪些trace的模块

[【谷歌分析Trace 文件的网站：http://ui.perfetto.dev/】](http://ui.perfetto.dev/)

也可以直接用该网站的在线工具来抓取trace。
#### 分析trace
在 Perfetto 中，找到包含“Android App Startups”派生指标的行。如果您没有看到该行，请尝试使用设备上的系统跟踪应用捕获跟踪记录。

![trace1](/assets/img/blog/blogs_trace1.png){:loading="lazy"}

选中这一slice，按m可以显示这一列的纵向区域。点图钉图标固定这一行，再去下面找详细的启动信息：

![trace2](/assets/img/blog/blogs_trace2.png){:loading="lazy"}
