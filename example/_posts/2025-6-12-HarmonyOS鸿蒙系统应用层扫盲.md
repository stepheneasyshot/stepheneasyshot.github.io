---
layout: post
description: > 
  本文针对鸿蒙系统的基础特性，应用层开发所需基础知识，进行一个简单的调研，对比一下和其他平台的差异。
image: 
  path: /assets/img/blog/blogs_harmonyos_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_harmonyos_cover.png
    960w:  /assets/img/blog/blogs_harmonyos_cover.png
    480w:  /assets/img/blog/blogs_harmonyos_cover.png
accent_image: /assets/img/blog/blogs_harmonyos_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# HarmonyOS鸿蒙系统应用层扫盲
## JS & TS & ArkTS
在HarmonyOS中，应用开发使用的是ArkTS语言。了解到它是基于 `TypeScript` 的扩展，旨在为开发者提供更高效、更安全的开发体验。

TypeScript 又是什么东西呢，对没有做过前端的开发来说，这几个语言的概念和特性都会有点疑惑。
### JavaScript (JS)
JavaScript是一种脚本语言，最初用于网页交互，现广泛用于前端、后端（Node.js）、移动端（React Native）等，现在已成为Web开发的主要语言。它是由网景公司（Netscape）在1995年开发的，最初被称为LiveScript，但后来改名为JavaScript。

JavaScript具有以下特点：
* 动态类型：变量类型在运行时确定，无需显式声明。
* 解释执行：浏览器或引擎（如V8）直接解释运行，无需编译。
* 灵活但易出错：开发快速，但大型项目中维护困难（如类型错误需运行时才发现）。

```javascript
// 动态类型
let num = 10; 
num = "hello"; // 合法，但易引发后续问题

// 函数式编程
const sum = (a, b) => a + b;
```

适用于网页动态交互（DOM操作）；快速原型开发或小型项目；与HTML/CSS配合的前端开发（如React/Vue）。

### TypeScript (TS)
TypeScript是JavaScript的超集，由微软开发，添加了静态类型系统，编译为JavaScript运行。

它具有以下特点：
* 静态类型：需在代码中声明变量类型，编译时检查错误。
* 编译为JavaScript：与JS兼容，可直接运行在浏览器或Node.js。
* 面向对象支持：类、接口、泛型等高级特性。
* 工具链完善：VS Code深度集成，提供智能提示和重构。

```typescript
// 类型注解
let num: number = 10;
num = "hello"; // 编译时报错

// 接口与泛型
interface User { id: number; name: string; }
const getUser = <T>(id: T): T => id;
```

适用于大型前端项目（如Angular默认使用TS），以及需要高可维护性和团队协作的场景，还有后端开发（如NestJS框架）。

### ArkTS
ArkTS是HarmonyOS生态的专属语言，由HarmonyOS官方提供，融合了TS的静态类型系统和HarmonyOS的声明式UI、状态管理等特性。

它具有以下特点：
* 静态类型：与TS一致，编译时检查错误。
* 声明式UI：提供类似React/Vue的声明式API，简化UI构建。
* 分布式能力：直接调用HarmonyOS API（如分布式软总线）。
* 高性能字节码：基于Ark运行时（类似JVM），性能优异。

```typescript
// 声明式UI
@Entry
@Component
struct MyComponent {
  build() {
    Column() {
      Text("Hello, HarmonyOS!");
      Button("Click Me", () => { console.log("Button Clicked"); });      
    }
  }
}
```

适用于HarmonyOS应用开发，特别是需要高性能、分布式能力的场景，如智能穿戴设备、智能车等。
## 应用运行环境
### Android
在Android平台上，每个应用程序都运行在一个独立的进程中，由Android Runtime（ART）虚拟机提供执行环境。

ART是基于寄存器架构的高级虚拟机，其设计继承自Dalvik虚拟机但进行了全面重构，采用AOT 预先编译技术将DEX字节码预先编译为本地机器码。

这是一个 **类JVM环境** ，通过优化过的指令集执行编译后的代码，同时保持与Java SE标准库的部分兼容性（通过Android SDK提供的精简实现）。

在系统架构层面，应用进程通过 **Binder** 这个 IPC 机制与Android系统服务进行进程间通信。Binder作为基于Linux内核的高效通信框架，采用内存映射和引用计数技术实现跨进程方法调用（RPC），其传输性能比传统IPC方式提升5-10倍。

例如 **AMS(ActivityManagerService)** 通过Binder向应用进程发送生命周期控制指令，而应用则通过代理接口（如IActivityManager.Stub）回调系统服务，形成完整的进程间协作机制。

应用运行时，ART虚拟机通过以下核心机制实现资源管理：
* 内存管理采用**分代垃圾回收**（Generational GC）策略，配合Android特有的 **Low Memory Kill** 机制
* UI线程使用**消息队列**（MessageQueue）和**Looper**实现事件驱动模型
* 图形渲染通过 **RenderThread** 与 **SurfaceFlinger** 服务协同，采用 VSync 信号同步的帧调度机制
* 资源加载通过 **Resources** 类实现多维度（dpi/语言/方向等）资源匹配系统

这种架构设计使得Android应用在性能受限的移动设备环境下，仍能实现高效的**进程隔离、资源调度和用户体验保障**。最新的ART改进（如从Android 12引入的压缩引用指针技术）进一步将内存占用降低20%，体现了持续优化的技术演进路径。
### IOS
在iOS平台上，每个应用程序都运行在一个**独立的沙盒环境**中，由Objective-C/Swift运行时（Runtime）提供核心执行能力，并基于Mach-O可执行格式直接在ARM架构上运行原生代码。不同于Android的虚拟机机制，iOS采用AOT（Ahead-Of-Time）全量编译，通过LLVM编译器链 **将代码预先优化为机器指令** ，结合苹果自研的A系列芯片的定制指令集（如Apple Silicon的AMX矩阵加速指令），实现接近裸金属的执行效率。
#### **系统架构与进程通信**
iOS通过XNU混合内核（融合Mach微内核与BSD特性）实现系统级隔离，应用与系统服务的交互主要依赖以下机制：
* Mach IPC：基于端口（port）的轻量级进程通信，延迟低于1微秒，用于底层服务调用（如进程生命周期通知）
* XPC：高层抽象通信框架，采用序列化对象（NSXPCConnection）实现沙盒间的安全数据交换
* Darwin Notify：基于notifyd服务的跨进程事件广播系统，用于处理系统状态变更（如低内存警告）

#### **核心运行机制**
##### **内存管理**  
* 采用自动引用计数（ARC）编译时内存管理，通过插入retain/release调用替代垃圾回收  
* 使用Jetsam机制主动终止内存超限进程，配合内存压缩（iOS 13+）降低OOM概率

##### **UI渲染流水线**  
* Core Animation合成器直接驱动GPU渲染（Metal API），通过CADisplayLink实现120Hz ProMotion自适应帧同步  
* UIKit主线程遵循严格的RunLoop事件模型（NSDefaultRunLoopMode），所有UI操作必须通过dispatch_async(dispatch_get_main_queue())提交

##### **资源管理**  
* Asset Catalog编译时自动生成多分辨率@2x/@3x图像集，并支持按设备型号动态加载  
* dyld共享缓存将系统库预链接为内存映射文件，减少应用启动时的符号解析开销

#### 性能优化特性
* Swift Runtime使用值类型（Value Types）减少堆分配，方法派发采用直接调用（非ObjC消息转发）  
* GCD（Grand Central Dispatch）：基于线程池的优先级队列（QoS级别），支持硬件线程感知的任务调度  
* MetricKit：实时监控CPU/内存/电池消耗，通过signpost日志实现纳秒级性能分析  

这种架构使得iOS应用在保持严格安全隔离的同时，仍能实现亚毫秒级响应（如相机启动仅需650ms）。随着Swift 6引入并发模型和actor隔离机制，系统进一步强化了多核环境下的线程安全能力，体现了苹果软硬协同设计的持续进化。

### HarmonyOS系统架构
回顾完Android和IOS的应用层架构，我们再来看下鸿蒙系统的应用层架构。

学习转载自阿里巴巴技术团队: [渲染引擎分析 - 鸿蒙(OpenHarmony) JS UI 源码阅读笔记](https://juejin.cn/post/7005093811383140388)

这套架构主体分为应用、框架、引擎以及跨平台适配这几部分，应用层就是透出给开发者的语法，有好几种模式，下文详解。框架层实现了前端框架常见的组件化、MVVM 能力，能够响应式的更新 UI。下面是 JS 引擎，使用的是 QuickJS，应该也支持 V8。再向下是渲染引擎，包含了核心的渲染管线、动画、事件和各种布局绘制算法。
最下面的 porting layer 是适配多平台的关键，定义了平台无关的 layer 数据结构，可以提交给不同的合成器(Compositor)合成渲染，从代码上看，是支持 Flutter Engine 的。

![](/assets/img/blog/blogs_harmonyos_arch.png)

首先鸿蒙应用是打包成 HAP (Harmony Ability Package) 格式分发的，和安卓 APK 一样都是压缩包，包结构大同小异。分发到端上之后，统一由鸿蒙(概念)的 API 承接，然后就分了不同的模式。在华为手机上推送的鸿蒙版本，可以无感兼容安卓应用，肯定是依赖了 AOSP 的。

