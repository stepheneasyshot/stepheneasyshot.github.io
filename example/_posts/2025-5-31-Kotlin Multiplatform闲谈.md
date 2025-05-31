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
## 有浏览器H5套壳，为什么还要用Kotlin跨端
H5 主要是使用 Web 技术（HTML、CSS、JavaScript）来构建应用程序，然后通过 WebView（一个内嵌的浏览器组件）在不同平台的原生应用中运行。常见的 H5 跨平台框架包括 Ionic、PhoneGap (Cordova)、以及一些基于小程序（如微信小程序）的开发方式。

实现上是将 Web 应用打包成原生应用，通过 WebView 渲染界面和执行逻辑。由于运行在 WebView 中，性能通常不如原生应用，尤其是在处理复杂动画、大量数据或需要高性能计算的场景。启动速度也不如原生应用。对原生功能的访问需要借助插件。

Kotlin Multiplatform (KMP) 是 JetBrains 推出的一项技术，它允许开发者使用 Kotlin 语言编写共享的业务逻辑（如数据处理、网络请求、商业规则等），并将其编译成适用于不同平台的原生代码。UI 层通常仍然使用各平台的原生技术（Android 的 Jetpack Compose/XML，iOS 的 SwiftUI/UIKit）来实现，但也支持使用 Compose Multiplatform 实现共享 UI。

Kotlin编译器会将面向不同平台的Kotlin代码直接 **编译成对应平台的Native代码实现** ，可以说在每个平台上都是 **原生应用** ，从性能上来说媲美原生。如果进一步使用Compose Multiplatform，则可以在各个平台上共享一套UI代码，使用 Skia 跨端渲染引擎，其性能也远高于 H5 开发。在开发周期上比 H5 长一点。

总结起来就是：
* H5 跨平台 更像是“将网页打包成应用”，优势在于开发速度快、Web 开发者门槛低，但性能和原生体验是其短板。
* Kotlin Multiplatform 更像是“用 Kotlin 编写原生应用的一部分”，优势在于性能接近原生、能充分利用原生特性，并且可以灵活选择共享业务逻辑或 UI，但对开发者有一定的 Kotlin 基础要求，且 UI 部分（如果选择原生）仍需单独开发。