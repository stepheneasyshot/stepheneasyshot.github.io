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

1.  **Systrace (旧版)**
    * **概念**：Systrace 是 Android 早期最常用的性能分析工具，它通过 Ftrace（Linux 内核中的一个跟踪工具）收集系统事件，并结合用户空间事件（由应用程序或系统服务通过 `Trace` 类或 `ATrace` 宏记录）生成 HTML 报告。
    * **优点**：可以直接从命令行使用，无需修改代码（针对系统事件）。
    * **缺点**：生成的 HTML 报告可视化能力有限，对大型 trace 文件分析效率不高。现在逐渐被 Perfetto 取代。

2.  **Perfetto (新版 & 推荐)**
    * **概念**：Perfetto 是 Google 开发的新一代系统级性能分析工具，它旨在替代和增强 Systrace。它提供了更丰富的数据源（包括 Ftrace、Perf、ftrace-events、Android events 等），更灵活的查询能力，以及更强大的 Web UI (UI 网址：[ui.perfetto.dev](https://ui.perfetto.dev))。
    * **优点**：
        * **更全面**：收集的数据类型更多，覆盖面更广。
        * **更灵活**：可以通过 protobuf 配置数据源。
        * **更强大**：Web UI 交互性强，支持 SQL 查询，方便深度分析。
        * **可编程**：可以通过 Python SDK 进行自动化分析。
    * **生成方式**：
        * 使用 `adb shell perfetto` 命令。
        * 使用 Android Studio 的 **CPU Profiler** (推荐，它在后台调用了 Perfetto)。
        * 通过 Android Developers Studio 的 **Perfetto SDK** 在应用中集成。

3.  **Android Studio CPU Profiler (最常用)**
    * **概念**：这是 Android Studio 中集成的性能分析工具，它集成了 CPU、内存、网络和电量分析功能。其中的 CPU Profiler 实际上在幕后使用了 Perfetto 或 ART (Android Runtime) 的采样/插桩机制来生成 trace 文件。
    * **优点**：
        * **集成度高**：与开发环境无缝集成，操作简便。
        * **可视化强**：提供了图形化的界面来展示 CPU 使用率、线程状态、方法调用栈等。
        * **多种记录模式**：支持 Sampled (采样)、Instrumented (插桩)、System Trace (系统跟踪，即 Perfetto)。
    * **生成方式**：在 Android Studio 中点击 Run -> Profile 'your app'，然后选择 CPU Profiler 并开始录制。

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

## 分析流程
### 冷启动

