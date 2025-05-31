---
layout: post
description: > 
  本文介绍了java线程池相关知识
image: 
  path: /assets/img/blog/blogs_kmp_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_kmp_cover.png
    960w:  /assets/img/blog/blogs_kmp_cover.png
    480w:  /assets/img/blog/blogs_kmp_cover.png
accent_image: /assets/img/blog/blogs_kmp_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Kotlin Multiplatform闲谈
论题来自于霍丙乾Benny Huo 在B站上的答网友问。基于其描述，详细扩展开来。
## 有浏览器H5套壳，为什么还要用Kotlin跨端
H5 主要是使用 Web 技术（HTML、CSS、JavaScript）来构建应用程序，然后通过 WebView（一个内嵌的浏览器组件）在不同平台的原生应用中运行。常见的 H5 跨平台框架包括 Ionic、PhoneGap (Cordova)、以及一些基于小程序（如微信小程序）的开发方式。

实现上是将 Web 应用打包成原生应用，通过 WebView 渲染界面和执行逻辑。由于运行在 WebView 中，性能通常不如原生应用，尤其是在处理复杂动画、大量数据或需要高性能计算的场景。启动速度也不如原生应用。对原生功能的访问需要借助插件。

Kotlin Multiplatform (KMP) 是 JetBrains 推出的一项技术，它允许开发者使用 Kotlin 语言编写共享的业务逻辑（如数据处理、网络请求、商业规则等），并将其编译成适用于不同平台的原生代码。UI 层通常仍然使用各平台的原生技术（Android 的 Jetpack Compose/XML，iOS 的 SwiftUI/UIKit）来实现，但也支持使用 Compose Multiplatform 实现共享 UI。

Kotlin编译器会将面向不同平台的Kotlin代码直接 **编译成对应平台的Native代码实现** ，可以说在每个平台上都是 **原生应用** ，从性能上来说媲美原生。如果进一步使用Compose Multiplatform，则可以在各个平台上共享一套UI代码，使用 Skia 跨端渲染引擎，其性能也远高于 H5 开发。在开发周期上比 H5 长一点。体量大一点的应用都会追求更长远，性能更好的技术

总结起来就是：
* H5 跨平台 更像是“将网页打包成应用”，优势在于开发速度快、Web 开发者门槛低，但性能和原生体验是其短板。
* Kotlin Multiplatform 更像是“用 Kotlin 编写原生应用的一部分”，优势在于性能接近原生、能充分利用原生特性，并且可以灵活选择共享业务逻辑或 UI，但对开发者有一定的 Kotlin 基础要求，且 UI 部分（如果选择原生）仍需单独开发。

## Kotlin/Native性能对比其他语言如何
Kotlin/Native 旨在将 Kotlin 代码编译为可以在没有虚拟机 (VM) 的情况下运行的本地二进制文件，使其适用于嵌入式设备或 iOS 等平台。Kotlin/Native 代码直接编译为机器码，因此它的执行速度通常非常快，可以与原生应用媲美。对于 CPU 密集型任务，其性能通常远优于 JVM 上的 Kotlin 或 JavaScript。

Kotlin/Native 包含一个现代的跟踪垃圾回收器。虽然自动内存管理简化了开发，但在某些情况下，GC 可能会引入微小的暂停，这可能会影响对实时性要求极高的应用。不过，JetBrains 正在不断优化其内存管理器。

同Swift相比，Kotlin不会自动回收，内存消耗会高一点，这使得其运行起来反而更快。对象在内存管理级别做了池化，创建和使用都会比Swift更快。

与 C/C++ 相比，它在以下几个方面通常存在差异。 

**C/C++：** 提供对内存的直接控制（通过指针）、更细粒度的硬件优化（如 SIMD 指令、CPU 缓存优化）以及手动内存管理。这使得 C/C++ 成为需要极致性能和资源控制的场景（如操作系统、驱动程序、游戏引擎、高性能计算）的首选。

**Kotlin/Native：** 虽然性能接近原生，但它仍然是高级语言，抽象层高于 C/C++。它提供了垃圾回收机制，简化了内存管理，但也意味着开发者对内存布局和生命周期的直接控制较少。在需要极度细致的内存布局和手动优化以榨取每一丝性能的场景下，C/C++ 仍然更具优势。

## 非Android平台为什么不使用Kotlin/Native替代JVM
以Android平台为例，如果使用 `Kotlin/Native` 去绕过Android虚拟机，那么开发上，Google的api都不可以直接使用了，仍然需要套一层壳。

在移动端的另外两大平台上，KMP在开发时，是采用了类似RN的桥接调用的，腾讯在将Kotlin/Native移植到鸿蒙系统时，就是使用ArkTS包装鸿蒙的系统api，给Kotlin调用，包括IOS上也是如此实现。

对于桌面端，`Compose Desktop` 仍然是运行在JVM上。Windows,MACOS,Linux等桌面系统，其底层的差异性，想抹平是非常困难的。

目前桌面端最火的 `Electron` 框架为例，Electron 应用通常需要独立运行，不能依赖用户本地环境的 Chromium 或 Node.js 版本，都会自己打包一个 V8 引擎，负责解析、编译和执行应用中的 JavaScript 代码（包括前端页面逻辑和 Node.js 后端代码）。正是因为有这个附带的引擎，安装包体积高达几百M，但是其能提供的开发体验和跨平台生态是更应该关注的点。

同理，Compose Desktop没有选择直接对接操作系统，而是选择运行于 JVM 上，也是由于JVM已经把平台适配给做完了，并且提供的接口和性能已经过多年发展验证，可以很好地支持快速开发和功能达成。包括前段时间的Rust，甚至是基于浏览器来运行。

如果不谈Compose UI界面，Kotlin本身其实是可以通过Kotlin/Native直接跑在各个桌面平台上的，比如Windows端通过 Mingw（Windows系统api封装中间层）。另外也可以通过GTK来实现UI界面。

**GTK（GIMP Toolkit）** 是一个开源的跨平台 图形用户界面（GUI）工具库，最初为图像处理软件 GIMP 开发，现广泛用于 Linux 桌面环境（如 GNOME）以及其他操作系统（Windows、macOS）。GTK 本身是用 C 编写的，和 Kotlin 无法直接交互，Binding 充当了“桥梁”，将 GTK 的 C 语言接口通过某种方式（如 JNI、FFI 或原生库）暴露给 Kotlin，使 Kotlin 开发者能直接调用 GTK 的功能来构建 GUI 应用。

## Compose IOS 的一些坑
Kotlin 最开始是和OC互调用的，后来的2.1版本才开始转向和 Swift `互调用。IOS开发端，Cocapods` 处于不维护的存量状态，后面会全面转向Swift了。