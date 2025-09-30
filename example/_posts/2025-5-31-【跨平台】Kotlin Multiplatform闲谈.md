---
layout: post
description: > 
  本文介绍了Kotlin Multiplatform框架过去现在和未来的一些讨论主题。
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
# 【跨平台】Kotlin Multiplatform闲谈
论题来自于 **霍丙乾(Benny Huo)** 在B站上的答网友问。基于其描述，详细扩展开来。
## 有浏览器H5套壳，为什么还要用Kotlin跨端
H5 主要是使用 Web 技术（HTML、CSS、JavaScript）来构建应用程序，然后通过 WebView（一个内嵌的浏览器组件）在不同平台的原生应用中运行。常见的 H5 跨平台框架包括 Ionic、PhoneGap (Cordova)、以及一些基于小程序（如微信小程序）的开发方式。

实现上是将 Web 应用打包成原生应用，通过 WebView 渲染界面和执行逻辑。由于运行在 WebView 中，性能通常不如原生应用，尤其是在处理复杂动画、大量数据或需要高性能计算的场景。启动速度也不如原生应用。对原生功能的访问需要借助插件。

Kotlin Multiplatform (KMP) 是 JetBrains 推出的一项技术，它允许开发者使用 Kotlin 语言编写共享的业务逻辑（如数据处理、网络请求、商业规则等），并将其编译成适用于不同平台的原生代码。UI 层通常仍然使用各平台的原生技术（Android 的 Jetpack Compose/XML，iOS 的 SwiftUI/UIKit）来实现，但也支持使用 Compose Multiplatform 实现共享 UI。

Kotlin编译器会将面向不同平台的Kotlin代码直接 **编译成对应平台的Native代码实现** ，可以说在每个平台上都是 **原生应用** ，从性能上来说媲美原生。如果进一步使用Compose Multiplatform，则可以在各个平台上共享一套UI代码，使用 Skia 跨端渲染引擎，其性能也远高于 H5 开发。在开发周期上比 H5 长一点。体量大一点的应用一般都会追求更长远，性能更好的技术

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

对于桌面端，`Compose Desktop` 仍然是运行在JVM上。Windows、MACOS、Linux等桌面系统底层的差异性，想抹平是非常困难的。

目前桌面端最火的 `Electron` 框架为例，Electron 应用通常需要独立运行，不能依赖用户本地环境的 Chromium 或 Node.js 版本，都会自己打包一个 V8 引擎，负责解析、编译和执行应用中的 JavaScript 代码（包括前端页面逻辑和 Node.js 后端代码）。正是因为有这个附带的引擎，安装包体积高达几百M，但是其能提供的开发体验和跨平台生态是更应该关注的点。

同理，Compose Desktop没有选择直接对接操作系统，而是选择运行于 JVM 上，也是由于JVM已经把平台适配给做完了，并且提供的接口和性能已经过多年发展验证，可以很好地支持快速开发和功能达成。包括前段时间的Rust，甚至是基于浏览器来运行。

如果不谈Compose UI界面，Kotlin本身其实是可以通过Kotlin/Native直接跑在各个桌面平台上的，比如Windows端通过 Mingw（Windows系统api封装中间层）。另外也可以通过GTK来实现UI界面。

**GTK（GIMP Toolkit）** 是一个开源的跨平台 图形用户界面（GUI）工具库，最初为图像处理软件 GIMP 开发，现广泛用于 Linux 桌面环境（如 GNOME）以及其他操作系统（Windows、macOS）。GTK 本身是用 C 编写的，和 Kotlin 无法直接交互，Binding 充当了“桥梁”，将 GTK 的 C 语言接口通过某种方式（如 JNI、FFI 或原生库）暴露给 Kotlin，使 Kotlin 开发者能直接调用 GTK 的功能来构建 GUI 应用。

## Compose IOS 的一些坑
Kotlin 最开始是和OC互调用的，后来的2.1版本才开始转向和 Swift `互调用。IOS开发端，Cocapods` 处于不维护的存量状态，整体在往Swift生态迁移，后面会全面转向Swift了。

**腾讯视频** 在IOS上自研了一套渲染引擎，因为要兼容大量的原生存量代码，前期只能小范围替换，省掉了一个渲染层的内存。在鸿蒙端是纯以来CMP的Skia了。

* 混合开发的时候，内存，画布开销。
* 单独的View容器需要自己管理生命周期。
* 列表组件单独嵌入到原生容器，可能开发上会比较麻烦，处理手势。

和 OC 互操作方面，类的导出有很多限制。和Swift应该差不多，也会有这些限制。编译的时候不会发现，运行时才知道，

霍老师在腾讯云开发者平台上发布的Kotlin/Native的文章，详细列出了这些限制的问题。

[深入理解Kotlin/Native](https://cloud.tencent.com/developer/article/2448428)

互操作尽可能少，导出一些工具函数等。

## 能做到三大移动端一把梭哈吗
前段时间 Jetbrains 发布通知，CMP的IOS端已经稳定。

在鸿蒙端，腾讯视频团队已经实现了比较稳定的方案，在和Jetbrains沟通能否贡献到官方代码中。

所以技术上Android、IOS、鸿蒙三端共用是没有问题的。

## CMP和RN和Flutter的对比
此前已经对比过一次三者的易用性，性能，渲染方式上的差异。
差异。

[Kotlin Multiplatform 对比 React Native 和 Flutter](./2025-4-29-Flutter%20&%20RN%20&%20CMP三种跨平台方案对比.md)

CMP最大的优势还是来自于Kotlin，在各个平台上，编译完后都是Native实现。另外在UI渲染上采用Skia，性能优于RN，持平Flutter。

**能否替代Flutter？**

有潜力，但是CMP的潜力不止于替换Flutter，Kotlin编译器编译完的代码，在框架上和原生开发无异，胃口大一点问问是否可以替代原生。

每个平台都有最合适的原生代码，CMP三段一码的开发成本，对大型app诱惑还是很大，成本和收益都是需要考虑的。

## CMP前景如何
CMP前景依托于Kotlin跨端的前景。

Kotlin跨平台是语言特性，而不是框架，并没有做一个中间层来转换。

Kotlin中可以直接访问C的结构体。

以Java生态要求Kotlin，还是有很长的路要走。

Compose的WebAssembly，也是 Jetbrains 官方下一阶段的重点，IOS端已经稳定。

相比其他的框架，其问题在于正是因为到处都是Native，就需要 **对这个target的原生环境有一定了解** 。知道如何去调用原生的API，比如处理内存，处理生命周期，处理手势等。

在IOS上写代码，就需要对Swift有一定了解，依赖原生api去配置。

## ovCompose
腾讯视频团队推出的跨平台开发框架ovCompose以及相关基础库KuiklyBase，旨在解决跨平台开发中的一些痛点问题，并推动Compose跨端生态的发展。
### 项目背景
* 跨平台需求：随着鸿蒙系统的推出，客户端跨平台开发的需求显著增加。传统的UI跨端方案已无法满足业务需求，全跨端APP（覆盖Android、iOS和鸿蒙）的开发成为降低开发成本和提升效率的关键。
* 现有技术的局限性：尽管Kotlin Multiplatform具备高性能和灵活的原生交互能力，但它存在一些问题，如不支持鸿蒙系统、iOS混排能力受限以及GC性能表现一般等。

### ovCompose和KuiklyBase的特性优势
* 鸿蒙高性能：KuiklyBase选择了Kotlin Native（KN）作为鸿蒙适配方案，相比JavaScript（JS）具有更快的执行速度和更好的三端一致性。通过性能优化，如内联优化、ThreadLocal优化等，显著提升了执行效率。
* 鸿蒙三明治架构支持混排：利用Skia渲染方案和XComponent组件，解决了Compose与原生组件的混排问题，支持粘贴按钮等安全组件的混排。
* 三端高一致性：通过Kotlin Native方案解决了跨线程访问问题，保持了逻辑运行的一致性。同时，iOS和鸿蒙平台均采用Skia渲染，确保了UI绘制的一致性。
* iOS多模态渲染：采用指令映射方案，解决了Compose在iOS上的混排难题，并实现了与原生UI的灵活混排。
* Kotlin Native内存优化：包括GC优化（如GC抑制、分段GC、Sweep优化）和堆Dump优化，显著提升了内存管理效率。
* KuiklyBase组件生态：提供了丰富的组件，如异常堆栈还原组件、跨语言互调用组件、资源管理组件、原子操作组件、协程组件、序列化组件等，为开发者提供了强大的支持。

### 实现原理
* KN鸿蒙平台适配：通过在Kotlin IR转LLVM IR时使用苹果的LLVM 11，在LLVM IR生成可执行文件时使用鸿蒙的LLVM 12，解决了鸿蒙平台的适配问题。
* KN性能优化：包括内联优化、ThreadLocal优化、协程性能优化、调试性能优化等，显著提升了Kotlin Native在鸿蒙平台上的性能。
* 鸿蒙绘制不同步问题解决：通过采用XComponent的Texture模式，解决了Compose与ArkUI绘制不同步的问题。
* iOS多模态渲染：设计了基于iOS的PictureRecorder局部更新架构，通过增量hash减少hash计算量，优化了绘制效率。

### 开源信息
开源仓库：ovCompose和KuiklyBase已在GitHub开源，包含5个仓库，地址为：https://github.com/Tencent-TDS

### 未来计划
* 持续优化：重点优化GC在业务场景的表现、Kotlin-Native组件化、开发体验优化以及UIKit渲染模式与Skia的进一步对齐。
* 扩展支持：计划将ovCompose和KuiklyBase扩展到TV和PC端，进一步完善跨平台开发框架。

### 与KuiklyUI的差异
* KuiklyUI：侧重于静态化+动态化双运行模式，采用轻量原生渲染，支持H5和小程序。
* ovCompose：专注于全面对齐Compose Multiplatform标准API，采用自渲染方式实现鸿蒙平台的适配，确保三端高度一致性。
