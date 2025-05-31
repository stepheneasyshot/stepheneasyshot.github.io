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

Kotlin编译器会将面向不同平台的Kotlin代码直接 **编译成对应平台的Native代码实现** ，可以说在每个平台上都是 **原生应用** ，从性能上来说媲美原生。如果进一步使用Compose Multiplatform，则可以在各个平台上共享一套UI代码，使用 Skia 跨端渲染引擎，其性能也远高于 H5 开发。在开发周期上比 H5 长一点。

总结起来就是：
* H5 跨平台 更像是“将网页打包成应用”，优势在于开发速度快、Web 开发者门槛低，但性能和原生体验是其短板。
* Kotlin Multiplatform 更像是“用 Kotlin 编写原生应用的一部分”，优势在于性能接近原生、能充分利用原生特性，并且可以灵活选择共享业务逻辑或 UI，但对开发者有一定的 Kotlin 基础要求，且 UI 部分（如果选择原生）仍需单独开发。

## Kotlin/Native性能对比其他语言如何
Kotlin/Native 旨在将 Kotlin 代码编译为可以在没有虚拟机 (VM) 的情况下运行的本地二进制文件，使其适用于嵌入式设备或 iOS 等平台。Kotlin/Native 代码直接编译为机器码，因此它的执行速度通常非常快，可以与原生应用媲美。对于 CPU 密集型任务，其性能通常远优于 JVM 上的 Kotlin 或 JavaScript。

Kotlin/Native 包含一个现代的跟踪垃圾回收器。虽然自动内存管理简化了开发，但在某些情况下，GC 可能会引入微小的暂停，这可能会影响对实时性要求极高的应用。不过，JetBrains 正在不断优化其内存管理器。

同Swift相比，Kotlin不会自动回收，内存消耗会高一点，这使得其运行起来反而更快。

与 C/C++ 相比，它在以下几个方面通常存在差异。 

**C/C++：** 提供对内存的直接控制（通过指针）、更细粒度的硬件优化（如 SIMD 指令、CPU 缓存优化）以及手动内存管理。这使得 C/C++ 成为需要极致性能和资源控制的场景（如操作系统、驱动程序、游戏引擎、高性能计算）的首选。

**Kotlin/Native：** 虽然性能接近原生，但它仍然是高级语言，抽象层高于 C/C++。它提供了垃圾回收机制，简化了内存管理，但也意味着开发者对内存布局和生命周期的直接控制较少。在需要极度细致的内存布局和手动优化以榨取每一丝性能的场景下，C/C++ 仍然更具优势。

