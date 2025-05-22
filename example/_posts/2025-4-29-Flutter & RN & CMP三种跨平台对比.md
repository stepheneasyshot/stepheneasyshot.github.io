---
layout: post
description: > 
  本文介绍了三种比较热门的跨平台技术的横向对比，涉及性能，易用性，开发成本等方面
image: 
  path: /assets/img/blog/blogs_cover_multiplatform.png
  srcset: 
    1920w: /assets/img/blog/blogs_cover_multiplatform.png
    960w:  /assets/img/blog/blogs_cover_multiplatform.png
    480w:  /assets/img/blog/blogs_cover_multiplatform.png
accent_image: /assets/img/blog/blogs_cover_multiplatform.png
excerpt_separator: <!--more-->
sitemap: false
---
# Flutter & RN & CMP三种跨平台对比
## 历史背景
### Flutter
Flutter的历史最早可以追溯到2014年10月，其前身是Google内部孵化的Sky项目。其是一款跨平台移动应用开发框架，它允许开发者使用单一代码库同时构建iOS和Android应用。Flutter采用了Dart编程语言，这是一种面向对象的、类型安全的编程语言，与JavaScript非常相似。Flutter的主要优势在于其快速的开发速度和流畅的用户体验。

具体的：
* 2014.10 - Flutter的前身Sky在GitHub上开源；
* 2015.10 - 经过一年的开源，Sky正式改名为Flutter；
* 2017.5 - Google I/O正式向外界公布了Flutter，这个时候Flutter才正式进去大家的视野；
* 2018.6 - 距5月Google I/O 1个月的时间，Flutter1.0预览版；
* 2018.12 - Flutter1.0发布，它的发布将大家对Flutter的学习和研究推到了一个新的起点；
* 2019.2 - Flutter1.2发布主要增加对web的支持。

### React Native
React Native是Facebook于2015年发布的一款跨平台移动应用开发框架，它允许开发者使用JavaScript和React来构建iOS和Android应用。React Native的主要优势在于其灵活的组件化开发方式和丰富的第三方库支持。
#### js语言和React
JavaScript是一种动态类型的、解释型的、基于原型的、多范式的编程语言。它最初由Netscape公司开发，后来被许多公司采用，包括Google、Microsoft、Facebook等。JavaScript的主要优势在于其跨平台的特性和丰富的第三方库支持。

React是一种用于构建用户界面的JavaScript库，它采用了组件化的开发方式，使得开发者可以将用户界面分解为多个可重用的组件。React的主要优势在于其高效的渲染机制和灵活的组件化开发方式。
### Compose Multiplatform
Compose Multiplatform是JetBrains于2021年发布的一款跨平台移动应用开发框架，它允许开发者使用Kotlin和Jetpack Compose来构建iOS和Android应用。Compose Multiplatform的主要优势在于其简洁的语法和强大的UI组件库。
#### Kotlin语言
Kotlin是一种静态类型的、基于JVM的编程语言，它与Java非常相似，但是它的语法更加简洁和灵活。Kotlin的主要优势在于其静态类型的特性和空安全的特性。Kotlin最强大的实际上是他的编译器，可以将Kotlin代码编译为Java字节码，从而可以在Java虚拟机上运行，也可以编译成js代码，从而可以在浏览器上运行等等。

## 开发流程
### Flutter
Flutter的开发流程，开发者需要使用Dart语言编写应用程序，然后使用Flutter SDK进行编译和打包。

Flutter的开发流程包括以下几个步骤：
* 编写Dart代码：开发者使用Dart语言编写应用程序的业务逻辑和界面。
* 编译和打包：开发者使用Flutter SDK进行编译和打包，生成iOS和Android应用程序。
* 运行应用程序：开发者可以使用模拟器或真机运行应用程序。

### React Native
React Native的开发流程相对复杂，开发者需要使用JavaScript和React编写应用程序，然后使用React Native CLI进行编译和打包。

React Native的开发流程包括以下几个步骤：
* 编写JavaScript代码：开发者使用JavaScript和React编写应用程序的业务逻辑和界面。
* 编译和打包：开发者使用React Native CLI进行编译和打包，生成iOS和Android应用程序。
* 运行应用程序

### Compose Multiplatform
Compose Multiplatform的开发流程相对简单，开发者只需要使用Kotlin和Jetpack Compose编写应用程序，然后使用Compose Multiplatform CLI进行编译和打包。Compose Multiplatform的开发流程包括以下几个步骤：
* 编写Kotlin代码：开发者使用Kotlin和Jetpack Compose编写应用程序的业务逻辑和界面。
* 编译和打包：开发者可以选择使用Android Studio或者IDEA ItelliJ进行编译和打包，生成iOS和Android应用程序。
* 运行应用程序

## 实现原理
### Flutter
Flutter的框架图如下：

![](/assets/img/blog/blogs_flutter_framework.png)

FLutter Engine
这是一个纯 C++实现的 SDK，其中囊括了 Skia引擎、Dart运行时、文字排版引擎等。
简单来说它就是一个 dart 运行时，可以以 JIT(动态编译) 或者 AOT(静态编译) 的方式运行 dart 代码。


Flutter Framework
最上层应用，我们的应用都是围绕这层来构建，所以该层也是我们打交道最多的层。
改层是一个纯 Dart实现的 SDK，类似于 React在 JavaScript中的作用。它实现了一套基础库， 用于处理动画、绘图和手势。并且基于绘图封装了一套 UI组件库，然后根据 Material 和Cupertino两种视觉风格区分开来。

【Foundation】 在最底层，主要定义底层工具类和方法，以提供给其他层使用。
【Animation】是动画相关的类，一些动画相关的都在该类中定义。
【Painting】封装了 Flutter Engine 提供的绘制接口，例如绘制缩放图像、插值生成阴影、绘制盒模型边框等。
【Gesture】提供处理手势识别和交互的功能。
【Rendering】是框架中的渲染库。控件的渲染主要包括三个阶段：布局（Layout）、绘制（Paint）、合成（Composite）。
【Widget】控件层。所有控件的基类都是 Widget，Widget 的数据都是只读的, 不能改变。
【Material】&【Cupertino】这是在 Widget 层之上框架为开发者提供的基于两套设计语言实现的 UI 控件，可以帮助我们的 App 在不同平台上提供接近原生的用户体验。

#### Dart内存分配机制
DartVM的内存分配策略非常简单，创建对象时只需要在现有堆上移动指针，内存增长始终是线形的,省去了查找可用内存段的过程。

Dart中类似线程的概念叫做Isolate，每个Isolate之间是无法共享内存的，所以这种分配策略可以让Dart实现无锁的快速分配。

#### Dart单线程异步原理

对于移动端的交互来说，大多数情况下都是在等待状态，等待网络请求，等待用户输入等.那么设想一下，发起一个网络请求只在一个线程中可以进行吗？当然网络请求肯定是异步的（注意这里说的异步而多线程并非一个概念.），事实验证是可以的，Flutter就采用了Dart这种单线程机制，省去了多线程上下文切换带来的性能损耗.（对于高耗时操作，也同样支持多线程操作，通过Isolate开启，不过注意这里的多线程，内存是无法共享的.）

当一个Dart的方法开始执行时，他会一直执行直至达到这个方法的退出点。换句话说Dart的方法是不会被其他Dart代码打断的。
当一个Dart应用开始的标志是它的main isolate执行了main方法。当main方法退出后，main isolate的线程就会去逐一处理消息队列中的消息。

![](/assets/img/blog/pic_flutter_eventqueue.png){:width ="500" height="180" loading="lazy"}

有了消息队列，然后有了循环去读取消息队列中的消息，就可以有单线程去执行异步消息的能力。一般的消息使用dart:async中使用Future来支持异步消息。

#### Flutter绘制
一般地来说，计算机系统中，CPU、GPU和显示器以一种特定的方式协作：CPU将计算好的显示内容提交给 GPU，GPU渲染后放入帧缓冲区，然后视频控制器按照 VSync信号从帧缓冲区取帧数据传递给显示器显示。

由于**最终的图形计算和绘制都是由相应的硬件来完成**，而直接操作硬件的指令通常都会有操作系统屏蔽，应用开发者通常不会直接面对硬件，操作系统屏蔽了这些底层硬件操作后会提供一些封装后的API供操作系统之上的应用调用。

但是对于应用开发者来说，直接调用这些操作系统提供的API是比较复杂和低效的，因为操作系统提供的API往往比较基础，直接调用需要了解API的很多细节。
正是因为这个原因，几乎所有用于开发GUI程序的编程语言都会在操作系统之上再封装一层，将操作系统原生API封装在一个编程框架和模型中，然后定义一种简单的开发规则来开发GUI应用程序。

例如:
Android SDK 正是封装了Android操作系统API，提供了一个“UI描述文件XML+Java操作DOM”的UI系统。iOS的UIKit 对View的抽象也是一样的，他们都将操作系统API抽象成一个基础对象（如用于2D图形绘制的Canvas），然后再定义一套规则来描述UI，如UI树结构，UI操作的单线程原则等。

![](/assets/img/blog/pic_flutter_screen_display.png){:width ="500" height="250" loading="lazy"}

Flutter只关心**向 GPU 提供视图数据**，GPU的 VSync信号同步到 UI 线程，UI线程使用 Dart 来构建抽象的视图结构，这份数据结构在 GPU 线程进行图层合成，视图数据提供给 Skia 引擎渲染为 GPU 数据，这些数据通过 OpenGL或者 Vulkan提供给 GPU。

所以 Flutter 并不关心显示器、视频控制器以及 GPU 具体工作，它只关心 GPU发出的 VSync 信号，尽可能快地在两个 VSync 信号之间计算并合成视图数据，并且把数据提供给GPU。Flutter的原理正是如此，它提供了一套Dart API，然后在底层通过skia这种跨平台的绘制库（内部会调用操作系统API）实现了一套代码跨多端。

Google官网的渲染流程示意图：

![](/assets/img/blog/blogs_flutter_implement_principle.png){:width ="400" height="600" loading="lazy"}

#### Flutter的测量布局
Flutter 采用约束进行单次测量布局. 整个布局过程中只需要深度遍历一次，极大的提升效能。

渲染对象树中的每个对象都会在布局过程中接受父 对象的 Constraints 参数,决定自己的大小, 然后父对象 就可以按照自己的逻辑决定各个子对象的位置,完成布局过程.
子对象不存储自己在容器中的位置， 所以在它的位置发生改变时并不需要重新布局或者绘制. 子对象的位 置信息存储在它自己的 parentData 字段中,但是该字段由它的父对象负责维护,自身并不关心该字段的内容。
同时也因为这种简单的布局逻辑， Flutter 可以在某些节 点设置布局边界 (Relayout boundary)， 即当边界内的任 何对象发生重新布局时， 不会影响边界外的对象， 反之亦然。