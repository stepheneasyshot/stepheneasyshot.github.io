---
layout: post
description: > 
  本文介绍了在分析Android性能问题过程中，常用的trace文件的分析方法。
image: 
  path: /assets/img/blog/blogs_trace_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_trace_cover.png
    960w:  /assets/img/blog/blogs_trace_cover.png
    480w:  /assets/img/blog/blogs_trace_cover.png
accent_image: /assets/img/blog/blogs_trace_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【性能优化】Android trace文件分析
**Android trace 文件**（也常被称为 **Systrace 文件**或 **Perfetto trace 文件**）是 Android 系统生成的一种包含详细性能事件数据的文件。它记录了设备在特定时间段内 **CPU、线程、进程、函数调用、Binder 通信、I/O 操作、SurfaceFlinger 帧渲染等** 各个层面的活动。

可以把它想象成一个高性能的“黑匣子记录仪”，它在系统运行时不断记录各种事件，当出现性能问题时，我们可以回放这些记录，了解系统当时到底发生了什么。

一般用来分析性能相关的问题：
1.  **识别性能瓶颈**：找出导致应用卡顿、响应慢、启动慢、耗电、UI 渲染不流畅等问题的根本原因。
2.  **分析系统行为**：深入了解应用与系统服务、框架层、硬件之间的交互。
3.  **优化代码逻辑**：定位到具体耗时函数或线程阻塞点，从而优化算法或并行处理。
4.  **调试复杂问题**：对于一些难以复现的性能问题，trace 文件能提供宝贵的线索。

## 采集方式
随着 Android 版本的迭代，trace 文件的生成和分析工具也在不断发展。

### **Systrace (旧版)**
Systrace 是 Android 早期最常用的性能分析工具，它通过 Ftrace（Linux 内核中的一个跟踪工具）收集系统事件，并结合用户空间事件（由应用程序或系统服务通过 `Trace` 类或 `ATrace` 宏记录）生成 HTML 报告。可以配置一系列参数，如：

```
gfx：图形 (Graphics) 相关事件；
input：输入 (Input) 事件，例如触摸、按键等；
view：视图系统 (View System) 事件；
wm：窗口管理器 (Window Manager) 事件;
am：Activity 管理器 (Activity Manager) 事件;
audio：音频 (Audio) 事件，与音频播放和录制相关;
...
```

可以直接从命令行使用，无需修改代码（针对系统事件）。但是其生成的报告可视化能力有限，且没有包名，只有进程id，对大型 trace 文件分析效率不高。现在逐渐被 Perfetto 取代。
### **Perfetto (新版 & 推荐)**
Perfetto 是 Google 开发的新一代系统级性能分析工具，它旨在替代和增强 Systrace。它是atrace的超集，除了应用层还可以抓取内核等底层的一些信息，提供了更丰富的数据源（包括 Ftrace、Perf、ftrace-events、Android events 等）。

还有更灵活的查询能力，以及更强大的 Web UI (UI 网址：[ui.perfetto.dev](https://ui.perfetto.dev))。

* **更全面**：收集的数据类型更多，覆盖面更广。
* **更灵活**：可以通过 protobuf 配置数据源。
* **更强大**：Web UI 交互性强，支持 SQL 查询，方便深度分析。
* **可编程**：可以通过 Python SDK 进行自动化分析。

可以使用 `adb shell perfetto` 命令来采集：

```
emu64xa:/ $ perfetto -h

Usage: perfetto
  --background     -d      : Exits immediately and continues in the background.
                             Prints the PID of the bg process. The printed PID
                             can used to gracefully terminate the tracing
                             session by issuing a `kill -TERM $PRINTED_PID`.
...
```

比较重要的参数有：

```
-t <duration> ：指定持续时间，例如 -t 10s 表示持续 10 秒。
-b <buffer-size> ：指定缓冲区大小，单位为 MB，例如 -b 128 表示 128 MB。
-c <config> ：指定配置文件，例如 -c my_config.pb 表示使用 my_config.pb 作为配置文件。
--output <file> ：指定输出文件，例如 --output my_trace.pb 表示将结果输出到 my_trace.pb 文件。
```

可以抓取的tag配置有如下：

```
emu64xa:/sys/kernel/tracing/events $ ls
alarmtimer    devfreq       gadget        irq_vectors  mt76            printk        spi        virtio_gpu
asoc          devlink       gpio          jbd2         mt76_usb        qdisc         spmi       vmalloc
avc           dma_fence     gpu_mem       kmem         mt76x02         ras           swiotlb    vmscan
binder        drm           header_event  kvm          napi            raw_syscalls  synthetic  vsock
block         dwc3          header_page   kvmmmu       neigh           rcu           task       vsyscall
bpf_test_run  enable        huge_memory   kyber        net             regmap        tcp        watchdog
bpf_trace     erofs         i2c           lock         netlink         regulator     thermal    wbt
bridge        error_report  initcall      mac80211     nmi             rpm           thp        workqueue
cfg80211      exceptions    interconnect  maple_tree   notifier        rtc           timer      writeback
cgroup        ext4          io_uring      mdio         nvme            sched         tlb        x86_fpu
cma           f2fs          iocost        migrate      oom             scsi          ucsi       xdp
compaction    fib           iomap         mmap         page_isolation  sd            udp        xhci-hcd
cpuhp         fib6          iommu         mmap_lock    page_pool       signal        ufs
csd           filelock      ipi           mmc          pagemap         skb           uvcg
damon         filemap       irq           module       percpu          smbus         v4l2
dev           ftrace        irq_matrix    msr          power           sock          vb2
```

我们还可以在新版的perfetto网站上直接采用图形化的方式去生成配置文件的代码：

![](/assets/img/blog/blogs_record_new_trace.png)

![](/assets/img/blog/blogs_record_new_trace_2.png)

选取要抓取的信息之后，到 `cmdline` tab那里复制下来：

```
buffers {
  size_kb: 65536
  fill_policy: DISCARD
}
buffers {
  size_kb: 4096
  fill_policy: DISCARD
}
data_sources {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_process_exit"
      ftrace_events: "sched/sched_process_free"
      ftrace_events: "task/task_newtask"
      ftrace_events: "task/task_rename"
      ftrace_events: "sched/sched_switch"
      ftrace_events: "power/suspend_resume"
      ftrace_events: "sched/sched_blocked_reason"
      ftrace_events: "sched/sched_wakeup"
      ftrace_events: "sched/sched_wakeup_new"
      ftrace_events: "sched/sched_waking"
      ftrace_events: "sched/sched_process_exit"
      ftrace_events: "sched/sched_process_free"
      ftrace_events: "task/task_newtask"
      ftrace_events: "task/task_rename"
      ftrace_events: "power/cpu_frequency"
      ftrace_events: "power/cpu_idle"
      ftrace_events: "power/suspend_resume"
      symbolize_ksyms: true
      disable_generic_events: true
    }
  }
}
data_sources {
  config {
    name: "linux.process_stats"
    process_stats_config {
      scan_all_processes_on_start: true
    }
  }
}
data_sources {
  config {
    name: "linux.sys_stats"
    sys_stats_config {
      stat_period_ms: 250
      stat_counters: STAT_CPU_TIMES
      stat_counters: STAT_FORK_COUNT
      cpufreq_period_ms: 250
    }
  }
}
duration_ms: 10000
```

直接粘贴到本地 txt 文档里，更名为 `pbt` 后缀，推送到设备中，就可以使用命令来使用这个配置文件采集对应的trace数据。
### **Android Studio CPU Profiler**
Android Studio 中已经自带了一个 Profiler 性能分析工具，它集成了 CPU、内存、网络和电量分析功能。其中的 CPU Profiler 实际上在幕后使用了 Perfetto 或 ART (Android Runtime) 的采样/插桩机制来生成 trace 文件。

* **集成度高**：与开发环境无缝集成，操作简便。
* **可视化强**：提供了图形化的界面来展示 CPU 使用率、线程状态、方法调用栈等。
* **多种记录模式**：支持 Sampled (采样)、Instrumented (插桩)、System Trace (系统跟踪，即 Perfetto)。

可以直接在 Android Studio 中点击 Run -> Profile 'your app'，然后选择 CPU Profiler 并开始录制。
## Trace中的重要信息
在分析 trace 文件时，通常需要关注以下几个核心信息：
1.  **CPU 使用率 (CPU Usage)**：显示每个 CPU 核的负载情况，以及进程和线程在 CPU 上的调度。
2.  **线程状态 (Thread States)**：每个线程在时间轴上的状态，如 Running (运行中)、Sleeping (休眠)、Runnable (可运行，等待 CPU)、Blocked (阻塞)。这对于识别线程阻塞和死锁非常关键。
3.  **函数调用 (Method Calls)**：如果使用采样或插桩模式，可以看到函数调用栈，帮助你识别耗时函数。
4.  **Binder Transactions**：进程间通信的事件，显示 Binder 调用的发起和接收，以及耗时。这是你刚才提到的重点。
5.  **I/O 操作 (Disk I/O)**：文件读写操作，过多的 I/O 会导致性能下降。
6.  **SurfaceFlinger & VSync**：显示帧的渲染过程，对于分析 UI 卡顿 (Jank) 至关重要。你可以看到 VSync 信号、应用绘制耗时、GPU 渲染耗时等。
7.  **内存事件 (Memory Events)**：虽然 CPU trace 主要关注 CPU，但有时也会包含一些内存分配/回收事件，帮你发现内存抖动。
8.  **自定义事件 (Custom Trace Events)**：你可以在自己的代码中插入 `Trace.beginSection()` / `Trace.endSection()` 或 `ATrace` 宏，在 trace 文件中标记出特定代码块的执行时间，这对于追踪应用内部逻辑的耗时非常有用。

注意在采集时，选择合适的 TAG 对于生成有效且不过大的 trace 文件至关重要。你需要根据你想要分析的性能问题来选择：
* UI 卡顿 / 渲染问题：gfx, view, wm, sched，以及你的 app 类别（用于自定义事件）。
* App 启动耗时：am, dalvik (或 ART), app, sched, disk, binder_driver。
* 耗电问题：power, sched, network, audio, video, camera, disk。
* 内存抖动 / GC 问题：dalvik (或 ART), app。
* 文件 I/O 性能：disk, sched, app。
* Binder IPC 问题：binder_driver, binder_lock, app, sched。

选择的类别越多，生成的 trace 文件就越大，分析起来也可能越慢，所以建议只选择你真正需要关注的类别。

## 分析流程
trace文件里面记录的信息是非常详细的，但是如果直接看这些信息，可能很难分析出问题所在。所以，我们需要分析trace文件里面的信息，得到我们想要的信息。并且根据所分析的问题不同，入手的地方也都不一样。常见的需要分析trace文件的场景有以下几个。
### 冷启动分析
Perfetto在线网站比较智能了，有冷启动发生的话，在 `startup` 一行里就会显示出来了。

首先，可以到 `system_server` 下面，找到**iq(Incoming Queue)** 事件。

> system_server 进程中的 iq 事件是 Binder 请求进入 system_server 传入队列的标记。它是衡量 system_server 处理 Binder 请求的负载和效率的关键指标。

再搜索 `launching` 事件，就可以找到应用启动的起始点。

从iq到整个launching，就是应用的整体启动耗时。
