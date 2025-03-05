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
# Android性能优化
## 奶酪模型
从产品设计到上线，每一个流程都像一片奶酪，令人不愉快的bug就像正好穿透了每一片奶酪的孔，到达了用户那里。比如开发逻辑考虑不全，测试漏测，环境不一致，验收不严格，发布的人员配置错了包。。。

做性能优化时，要往前一步，加强各个环节的管理，尽可能早的捕获异常。

## 开发自测工具
### Leakcanary
大名鼎鼎的内存泄露检测工具，可以检测到泄露之后创建一个桌面图标，弹出一个提示，可以查看堆栈信息。

[【github: leak_canary】](https://github.com/square/leakcanary)

### Dokit
滴滴开源的工具，可以轻松在设备上查看各种信息，进行快捷测试。甚至还有ios，小程序，web端的。

[【github: dokit】](https://github.com/didi/DoKit)

### Flipper
桌面端工具，facebook团队开源。可以实时查看日志，cpu频率，网络，数据库等。可以支持开发者自定义插件。

[【github: flipper】](https://github.com/facebook/flipper)

## 各个测试项目
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

上面的回复数据，第一行加了7个描述
* user:用户态时间
* nice:通过nice修改优先级之后的进程的用户态时间
* sysetm:内核态时间
* idle:空闲时间
* iwait:等待IO完成的时间
* irq:硬件中断的时间
* softirq:软件中断的时间

数据单位是jiffies，表示时钟中断次数，一般为1/100s

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
启动和卡顿测试，还要关注文件读写情况。

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

## 启动速度优化
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
如果View已经被添加到Window,然后被移除。例如,通过调用removeView()或ViewGone(),再次添加它时,系统可能会尝试重用之前的Window ID。
再次添加View时,它可能不会经历完整的布局和绘制流程,特别是如果它的尺寸和位置没有改变。系统可能会尝试优化性能,只进行必要的更新。
如果View的状态(如可见性、尺寸、位置等)在移除和再次添加之间发生了变化,系统会相应地更新这些状态。

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
如果有其他的异步操作影响了界面交互，需要在所有控件及数据状态加载完毕，确认可交互状态时，主动调用 reportFullyDrawn方法，以获取最高可达 TTFD 的信息。例如Demo中，填入一个长度为1000的recyclerView，完全显示后，主动调用此方法，打印出来的时间如下：
2s728ms，比上面看的TTID要长不少。

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
Android 9开始默认预制了Perfetto，需要手动开启。

```bash
setprop persist.traced.enable 1
perfetto -o /data/misc/perfetto-traces/trace_log -t 120s -b 100mb -s 150mb sched freq idle am wm gfx view input
```

参数说明：
• -o trace文件输出路径
• -t 抓取trace的时间
• -b buffer大小
• -catecategories 抓trace的模块

[【谷歌分析Trace 文件的网站】](http://ui.perfetto.dev/)

也可以直接用网站来抓取trace。

#### 分析trace
在 Perfetto 中，找到包含“Android App Startups”派生指标的行。如果您没有看到该行，请尝试使用设备上的系统跟踪应用捕获跟踪记录。

![trace1](/assets/img/blog/blogs_trace1.png){:loading="lazy"}

选中这一slice，按m可以显示这一列的纵向区域。点图钉图标固定这一行，再去下面找详细的启动信息：

![trace2](/assets/img/blog/blogs_trace2.png){:loading="lazy"}

#### 优化手段
1. 所有和界面无关的操作移到工作线程，主线程上刷新数据的逻辑可以移到协程中解决，采用非阻塞的挂起式数据流转方法。
2. 简化View的层级结构，尽可能地减少嵌套减少View的层级。

## 内存泄漏
### 几种引用类型

![referance_type](/assets/img/blog/blogs_jvm_referance_type.png){:loading="lazy"}

* 强引用：Object a=new object(); 创建对象就是一种强引用。对于强引用的对象，就算是出现了OOM也不会对该对象进行回收。在Java中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，它是不可能被垃圾回收机制回收的，即使该对象以后永远都不会被用到JVM也不会回收。因此强引用是造成Java内存泄漏的主要原因之一。 对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为null，一般认为就是可以被垃圾收集的了。
* 软引用：SoftReference<Object> softReference=new SoftReference<>(o1);对于只有软引用的对象来说， 当系统内存充足时它不会被回收，当系统内存不足时它会被回收。软引用通常用在对内存敏感的程序中，比如高速缓存就有用到软引用，内存够用的时候就保留，不够用就回收！
* 弱引用：对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管JVM的内存空间是否足够，都会回收该对象占用的内存。需要使用WeakReference来实现。
* 虚引用：正式学习JVM再看

在 Android 开发中，内存泄漏是一个需要特别关注的问题。以下是一些常见的内存泄漏场景：

### 一、单例模式引起的内存泄漏
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

### 二、非静态内部类引起的内存泄漏
非静态内部类会隐式地持有外部类的引用。如果在外部类（如 Activity）的生命周期内，非静态内部类的实例一直存在，就可能导致外部类无法被回收。

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

### 三、Handler 引起的内存泄漏
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

1. 在 Activity 销毁时，及时移除 Handler 中的所有消息。

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null);
}
```

2. 将Handler改为静态内部类 + WeakReference 来避免内存泄漏。

```java
public class MainActivity extends AppCompatActivity {
    private static class MyHandler extends Handler {
        private WeakReference<MainActivity> activityWeakReference;
        public MyHandler(MainActivity activity) {
            activityWeakReference = new WeakReference<>(activity);
        }
        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = activityWeakReference.get();
            if (activity != null) {
                // 处理消息
            }
        }
    }
    private MyHandler handler = new MyHandler(this);
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

### 四、资源未关闭引起的内存泄漏
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

如果在 Activity 销毁时没有关闭数据库连接，database 对象将一直存在，导致 MainActivity 无法被回收。

解决方案：
1. 在 Activity 销毁时，及时关闭数据库连接。

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    if (database != null) {
        database.close();
    }
}
```

### 五、注册的监听器未注销引起的内存泄漏
如果在 Activity 中注册了监听器，如广播接收器、系统服务的监听器等，在 Activity 销毁时没有注销这些监听器，就会导致 Activity 无法被回收。

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

当 MainActivity 销毁时，由于没有注销广播接收器，导致 MainActivity 无法被回收。

### 六、线程未停止引起的内存泄漏
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

### 七、集合类存储长生命周期的对象导致泄露
集合类使用不当导致的内存泄漏，这里分两种情况来讨论：

1）集合类添加对象后不移除的情况
对于所有的集合类，如果存储了对象，如果该集合类实例的生命周期比里面存储的元素还长，那么该集合类将一直持有所存储的短生命周期对象的引用，那么就会产生内存泄漏，尤其是使用static修饰该集合类对象时，问题将更严重，我们知道static变量的生命周期和应用的生命周期是一致的，如果添加对象后不移除，那么其所存储的对象将一直无法被gc回收。解决办法就是根据实际使用情况，存储的对象使用完后将其remove掉，或者使用完集合类后清空集合，原理和操作都比较简单，这里就不举例了。

2）根据hashCode的值来存储数据的集合类使用不当造成的内存泄漏
以HashSet为例子，当一个对象被存储进HashSet集合中以后，就不能再修改该对象中参与计算hashCode的字段值了，否则，原本存储的对象将无法再找到，导致无法被单独删除，除非清空集合，这样内存泄漏就发生了。

### 八、第三方库使用不当造成的内存泄漏
使用第三方库的时候，务必要按照官方文档指定的步骤来做，否则使用不当也可能产生内存泄漏，比如：

（1）EventBus，也是使用观察者模式实现的，同样注册和反注册要成对出现。

（2）Rxjava中，上下文销毁时，Disposable没有调用dispose()方法。

（3）Glide中，在子线程中大量使用Glide.with(applicationContext)，可能导致内存溢出

### 分析方法
首先，遇到一个内存泄漏问题，如果是必现，我们可以直接根据必现流程，执行了哪些代码块，正向运行排查。如果是偶现的，需要先按照日志显示的流程，尝试去复现问题。

其次，我们可以使用Android Studio的Memory Profiler来分析内存使用情况，查看内存使用的对象和引用关系，从而定位内存泄漏的原因。

Profiler入口：

![memory_profiler](/assets/img/blog/blogs_android_studio_profiler.png)

Live Telemetry 可以实时查看应用的内存使用情况，包括内存分配、内存释放等信息。可以通过点击 Live Telemetry 按钮来打开 Live Telemetry 窗口。

![memory_profiler_1](/assets/img/blog/blogs_android_live_telemetry.png)

还可以直接在shell中使用top或者ps命令来查看应用的内存使用情况，边操作边观察，看看是哪里导致的内存泄漏，再去分析可能的原因来解决。

![memory_top](/assets/img/blog/blogs_top_performance.png)
## 应用内存溢出 OOM
内存溢出是指应用程序试图分配比系统可用内存更多的内存空间。在 Android 中，每个应用程序都有一个特定的内存限制，这个限制取决于设备的硬件和操作系统版本。当应用程序尝试分配的内存超过这个限制时，就会发生内存溢出错误。

可能的原因：

1. 加载大图片
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

## 内存抖动
不再使用的内存被回收是好事，但也会产生一定的负面影响。

在 Android Android 2.2及更低版本上，当发生垃圾回收时，应用的线程会停止，这会导致延迟，从而降低性能。

在Android 2.3开始添加了并发垃圾回收功能，也就是有独立的GC线程来完成垃圾回收工作。

但即便如此，系统执行GC的过程中，仍然会占用一定的cpu资源。频繁地分配和回收内存空间，可能会出现内存抖动现象。

> 内存抖动是指在短时间内内存空间大量地被分配和回收，内存占用率马上升高到较高水平，然后又马上回收到较低水平，然后再次上升到较高水平...这样循环往复的现象。体现在代码中，就是短时间内大量创建和销毁对象。内存抖动严重时会造成肉眼可见的卡顿，甚至造成内存溢出（内存回收不及时也会造成内存溢出），导致app崩溃。

内存抖动的原因：

当调用Sytem.gc()时，程序只会显示地通知系统需要进行垃圾回收了，但系统并不一定会马上执行gc，系统可能只会在后续某个合适的时机再做gc操作。所以对于开发者来说，无法控制对象的回收，所以在做优化时可以从对象的创建上入手，这里提几点避免发生内存抖动的建议：

* 尽量避免在较大次数的循环中创建对象，应该把对象创建移到循环体外。
避免在绘制view时的onDraw方法中创建对象，实际上Android官方也是这样建议的。
* 如果确实需要使用到大量某类对象，尽量做到复用，这一点可以考虑使用设计模式中的享元模式，建立对象池。

举例：

遍历一个String list，对每一个元素进行拼接，返回结果。

```java
public static String changeListToString(List<String> list) {
        String result = "";
        for (String str : list) {
            result += (str + ";");
        }
        return result;
    }
```

String的底层实现是数组，不能进行扩容，拼装字符串的时候会重新生成一个String对象，所以第4行代码执行一次就会生成一个新的String对象，这段代码执行完成后就会产生list.size()个对象。

采用StringBuilder优化：

```java
public static String changeListToString(List<String> list) {
        StringBuilder result = new StringBuilder();
        for (String str : list) {
            result.append(str).append(";");
        }
        return result.toString();
    }
```

StringBuilder执行append方法时，会在原有实例基础上操作，不会生成新的对象，所以上述代码执行完成后就只会产生一个StringBuilder对象。

> 字符串的加号“+” 方法， 虽然现在编译器对其做了优化，使用StringBuilder的append方法进行追加，但是在循环场景，每循环一次都会创建一个StringBuilder对象，且都会调用toString方法转换成字符串，所以开销很大。

## 掉帧优化
Android 系统每隔16ms发出VSYNC信号，触发对UI进行渲染。如果UI渲染的时间超过16ms，就会导致掉帧，从而影响用户体验。

doFrame源码：

```java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
        }
    }
    ...
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
}
```

查找问题的经典log：

```
"Skipped xx frames! The application may be doing too much work on its main thread"
```

大概率是应用的主线程里面安排的任务太多，耗时过长，导致没有在vSYNC信号到来时进行渲染，从而导致掉帧。

实际工作中，我的座椅面板，一个浮窗类的app在退场时也有这个日志打印，每次关闭浮窗时的掉帧都是50帧左右。

分析执行的代码块后，发现pag动效的回收操作耗时过长。

因为pagView的动画显示，需要用到系统的编解码器，座椅app界面内的动效数量又名列前茅，如果没有及时地释放，导致编解码器耗尽，其他媒体类应用打开后就可能会一片黑屏，无法正常播放视频。所以每次窗口remove之前，都会停止动效，回收资源，避免掉帧。

将pagview改为了ImageView加帧动画的方案，在前台的资源占用略有增加，但是无需考虑动效资源释放问题。

改帧动画方案的话，特别注意的是，帧动画不可以后台更新，如果数量过多，会在缓存里堆积大量的Drawable对象，严重的话甚至导致内存溢出。