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
# Flutter & RN & CMP三种跨平台方案对比
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

**FLutter Engine**
这是一个纯 C++实现的 SDK，其中囊括了 Skia引擎、Dart运行时、文字排版引擎等。
简单来说它就是一个 dart 运行时，可以以 JIT(动态编译) 或者 AOT(静态编译) 的方式运行 dart 代码。

**Flutter Framework**
最上层应用，我们的应用都是围绕这层来构建，所以该层也是我们打交道最多的层。
改层是一个纯 Dart实现的 SDK，类似于 React在 JavaScript中的作用。它实现了一套基础库， 用于处理动画、绘图和手势。并且基于绘图封装了一套 UI组件库，然后根据 Material 和Cupertino两种视觉风格区分开来。

* 【Foundation】 在最底层，主要定义底层工具类和方法，以提供给其他层使用。
* 【Animation】是动画相关的类，一些动画相关的都在该类中定义。
* 【Painting】封装了 Flutter Engine 提供的绘制接口，例如绘制缩放图像、插值生成阴影、绘制盒模型边框等。
* 【Gesture】提供处理手势识别和交互的功能。
* 【Rendering】是框架中的渲染库。控件的渲染主要包括三个阶段：布局（Layout）、绘制（Paint）、合成（Composite）。
* 【Widget】控件层。所有控件的基类都是 Widget，Widget 的数据都是只读的, 不能改变。
* 【Material】&【Cupertino】这是在 Widget 层之上框架为开发者提供的基于两套设计语言实现的 UI 控件，可以帮助我们的 App 在不同平台上提供接近原生的用户体验。

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

![](/assets/img/blog/pic_flutter_screen_display.png){:width ="500" height="250" loading="lazy"}

由于**最终的图形计算和绘制都是由相应的硬件来完成**，而直接操作硬件的指令通常都会有操作系统屏蔽，应用开发者通常不会直接面对硬件，操作系统屏蔽了这些底层硬件操作后会提供一些封装后的API供操作系统之上的应用调用。

但是对于应用开发者来说，直接调用这些操作系统提供的API是比较复杂和低效的，因为操作系统提供的API往往比较基础，直接调用需要了解API的很多细节。
正是因为这个原因，几乎所有用于开发GUI程序的编程语言都会在操作系统之上再封装一层，将操作系统原生API封装在一个编程框架和模型中，然后定义一种简单的开发规则来开发GUI应用程序。

例如:
> Android SDK 正是封装了Android操作系统API，提供了一个“UI描述文件XML+Java操作DOM”的UI系统。iOS的UIKit 对View的抽象也是一样的，他们都将操作系统API抽象成一个基础对象（如用于2D图形绘制的Canvas），然后再定义一套规则来描述UI，如UI树结构，UI操作的单线程原则等。

Flutter只关心**向 GPU 提供视图数据**，GPU的 VSync信号同步到 UI 线程，UI线程使用 Dart 来构建抽象的视图结构，这份数据结构在 GPU 线程进行图层合成，视图数据提供给 Skia 引擎渲染为 GPU 数据，这些数据通过 OpenGL或者 Vulkan提供给 GPU。

所以 Flutter 并不关心显示器、视频控制器以及 GPU 具体工作，它只关心 GPU发出的 VSync 信号，尽可能快地在两个 VSync 信号之间计算并合成视图数据，并且把数据提供给GPU。Flutter的原理正是如此，它提供了一套Dart API，然后在底层通过skia这种跨平台的绘制库（内部会调用操作系统API）实现了一套代码跨多端。因此，组件更新（例如，iOS 16）对 Flutter 应用程序没有任何影响，但对 React Native 应用程序有影响。

Google官网的渲染流程示意图：

![](/assets/img/blog/blogs_flutter_implement_principle.png){:width ="300" height="470" loading="lazy"}

#### Flutter的测量布局
Flutter 采用约束进行单次测量布局. 整个布局过程中只需要深度遍历一次，极大的提升效能。

渲染对象树中的每个对象都会在布局过程中接受父 对象的 Constraints 参数,决定自己的大小, 然后父对象 就可以按照自己的逻辑决定各个子对象的位置,完成布局过程.
子对象不存储自己在容器中的位置， 所以在它的位置发生改变时并不需要重新布局或者绘制. 子对象的位 置信息存储在它自己的 parentData 字段中,但是该字段由它的父对象负责维护,自身并不关心该字段的内容。
同时也因为这种简单的布局逻辑， Flutter 可以在某些节 点设置布局边界 (Relayout boundary)， 即当边界内的任 何对象发生重新布局时， 不会影响边界外的对象， 反之亦然。

### React Native
#### DOM
文档对象模型（Document Object Model，DOM）是针对 HTML 和 XML 文档的一个编程接口。它将网页文档呈现为结构化的对象树，让程序和脚本能够动态地访问、修改文档的内容、结构和样式。 

DOM 把整个文档看作是由节点（Node）构成的树形结构。每个节点代表文档中的一个部分，比如元素、属性、文本等，这些节点相互关联，形成了一个层次分明的树状结构。

在浏览器环境中，可以使用 JavaScript 来操作 DOM。以下是一些常见的 DOM 操作示例：

```html
<!DOCTYPE html>
<html>
<head>
    <title>DOM 操作示例</title>
</head>
<body>
    <h1 id="heading">原始标题</h1>
    <button id="changeBtn">修改标题</button>
    <script>
        // 获取元素节点
        const heading = document.getElementById('heading');
        const changeBtn = document.getElementById('changeBtn');

        // 为按钮添加点击事件
        changeBtn.addEventListener('click', function() {
            // 修改元素的文本内容
            heading.textContent = '修改后的标题';
            // 修改元素的样式
            heading.style.color = 'red';
        });
    </script>
</body>
</html>
```

在这个示例里，借助 `document.getElementById` 方法获取元素节点，再用 `textContent` 修改元素文本内容，style `修改元素样式，addEventListener` 绑定点击事件。这些都是典型的 DOM 操作。

#### React库原理
先简单了解下 `React` 的工作原理。React 是一个用于构建用户界面的 JavaScript 库，核心工作原理可概括为组件化开发、虚拟 DOM 和协调算法三个方面。

React 采用组件化的开发方式，开发者能将用户界面拆分成多个可复用的组件。每个组件都有独立的状态（state）和属性（props），并且可以管理自身的逻辑和渲染。

**虚拟 DOM（Virtual DOM）** 是 React 的核心概念之一，它是真实 DOM 的轻量级副本，以 JavaScript 对象的形式存在于内存中。当组件的状态或属性发生变化时，React 会先在虚拟 DOM 上进行修改，计算出与之前虚拟 DOM 的差异（Diff）。React 利用 **协调算法** 对比新旧虚拟 DOM 的差异，找出需要更新的最小 DOM 操作集合，然后只对真实 DOM 进行这些必要的更新。这样可以**减少直接操作真实 DOM 的次数**，提高性能。

### React Native架构
#### **基于Bridge的架构**
示意图：
![](/assets/img/blog/blogs_rn_bridge.png){:width ="200" height="280" loading="lazy"}

* 在开发阶段仍然是聚焦于React Components的开发，Babel会将代码编译成浏览器可识别的表达式，并打包成jsbundle文件存储于App设备本地或者存储于服务器（热更新机制）
* 打开App后，加载并解析jsbundle文件，在JavascriptCore中进行运行，这个地方Android和IOS的差异就是，IOS原生就带有一个JavascriptCore，而Android中需要重新加载，所以这也造成了Android的初始化过程会比IOS慢一些。
* 运行时需要将前端的组件渲染成Native端的视图，首先如同React中的虚拟DOM一样，在Bridge中也会构造出一个Shadow Tree，然后通过Yoga布局引擎将Flex布局转换为Native的布局，最终交由UIManager在Native端完成组件的渲染。
* Bridge架构对于开发者来说很好的屏蔽了各个平台之间的差异，相对于WebView也能够提供不错的近原生操作体验。但是Javascript与Native之间的通信过度的依赖Bridge，当交互频繁或数据量很大的时候可能造成白屏或事件阻塞。而且JSON的序列化操作的效率也比较低。

##### **Bridge**
Bridge 顾名思义就是 JS 和 Native 通信的一个桥梁, 所有的本地存储、图片资源访问、图形绘制、3D加速、网络访问、震动效果、NFC、原生控件绘制、地图、定位、通知等等很多功能都是由 Bridge 封装成 JS 接口以后注入 JS Engine 供 JS 调用。

每一个支持 RN 的原生功能必须有同一个原生模块和一个 JS 模块, JS 模块方便调用其接口, 原生模块去根据接口调用原生功能实现原生效果。
Bridge 原生代码负责管理原生模块并能够方便的被 React 调用, 每个功能 JS 封装主要是对 React 做适配, JS 和 Native 之间不存在任何指针传递, 所有的参数均由字符串传递。

重要组件MessageQueue

RN 是不用 JS 引擎的 UI 渲染控件的, 但是会用到 JS 引擎的 DOM 操作管理能力来管理所有 UI 节点, 每次在写完 UI 组件代码后会交给 yoga 去做布局排版, 然后调用原生组件绘制
MessageQueue 负责跳出 JS 引擎, 记录原生接口的地址和对应的 JS 函数名, 然后在 JS 调用该函数的时候把调用转发给原生接口

双端差异：JS 和 IOS 通信用的是 JavaScript Core。 JS 和 Android 通信用的是 Hermes。
##### RN 主要有 3 个线程
JS Thread 执行线程, 负责逻辑层面的处理, Metro 将 React 源码打包成 bundle 文件, 然后传给 JS 引擎执行, 现在 IOS 和 Android 统一的是 JSC
UI Thead 主要主责原生渲染 Native UI 和调用原生能力 (Native Module)
Shadow Thead 这个线程主要创建 Shadow Tree 来模拟 React 结构树, RN 使用 Flexbox 布局, 但是原生不支持, Yoga 引擎就是将 Flexbox 布局转换为原生布局的。

基础概念
* UIManager: 在 Native 里只有它才有权限调用客户端UI
* JS Thread: 运行打包好的 bundle 文件, 这个文件就是我们写完代码去进行打包的文件, 包含了业务逻辑, 交互和模块化组件
* Shadow Node: Native 的一个组件树, 可以监听 App 里的 UI 变化, 类似于虚拟 DOM 和 DOM
* Yoga: Fackbook 开源的一个布局引擎, 用来把 Flexbox 的布局转换为 Native 的布局

##### 运行流程
1. 用户点击 App 图标
1. UIManager 线程: 加载所有 Native 库和 Native 组件比如 Images, View, Text
1. 告诉 JS 线程, Native 部分准备好了, JS 侧开始加载 bundle 文件
1. JS 通过 Bridge 发送一条 Json 数据到 Native , 告诉 Native 怎么创建 UI, 所有的 Bridge 通信都是异步的, 为了避免阻塞 UI
1. Shadow 线程最先拿到消息, 创建 UI 树
1. Yoga 引擎获取布局并转为 Native 的布局
1. 之后 UI Manager 执行一些操作展示 Native UI

##### Brige的缺点
有两个不同的领域 JS 和 Native, 他们彼此之间不能相互感知, 也并不能共享相同内存
通信基于 Bridge 的异步通信, 所以并不能保证数据百分百及时传达到另一侧
JSON 传输大数据非常慢, 内存不能共享, 所有的传输都是新的复制
无法同步更新 UI, 比方在渲染列表的时候, 滑动大量加载数据, 屏幕可能会发生卡顿或闪烁
RN 代码仓库很大, 库比较重, 所以修复 Bug 和开源社区贡献代码的效率也相应更慢

#### **引入了JSI的新架构**
上层 JavaScript 代码需要一个运行时环境，在 React Native 中这个环境是 JSC（JavaScriptCore）。不同于之前直接将 JavaScript 代码输入给 JSC，新的架构中引入了一层 JSI（JavaScript Interface），作为 JSC 之上的抽象，用来屏蔽 JavaScript 引擎的差异，允许换用不同的 JavaScript 引擎

RN的新版架构图：
![](/assets/img/blog/blogs_rn_jsi.png){:width ="200" height="380" loading="lazy"}

* JSI（Javascript Interface）：JSI的作用就是让Javascript可以持有C++对象的引用，并调用其方法，同时Native端（Android、IOS）均支持对于C++的支持。从而避免了使用Bridge对JSON的序列化与反序列化，实现了Javascript与Native端直接的通信。
JSI还屏蔽了不同浏览器引擎之间的差异，允许前端使用不同的浏览器引擎，因此Facebook针对Android 需要加载JavascriptCore的问题，研发了一个更适合Android的开源浏览器引擎Hermes。

* CodeGen：作为一个工具来自动化的实现Javascript和Native端的兼容性，它可以让开发者创建JS的静态类，以便Native端（Fabric和Turbo Modules）可以识别它们，并且避免每次都校验数据，将会带来更好的性能，并且减少传输数据出错的可能性。

新的 Bridge 层被划分成 Fabric 和 TurboModules 两部分

* Fabric：相当于之前的UIManager的作用，不同之处在于旧架构下Native端的渲染需要完成一系列的”跨桥“操作，即React -> Native -> Shadow Tree -> Native UI，新的架构下UIManager可以通过C++直接创建Shadow Tree大大提高了用户界面体验的速度。

* TurboModules：旧架构下由于端与端之间的隔阂，运行时即便没有使用的模块也会被加载初始化，TurboModules允许Javascript代码仅在需要的时候才去加载对应的Native模块并保留对其直接的引用缩短了应用程序的启动时间。

**新架构的核心改变就是避免了通过Bridge将数据从JavaScript序列化到Native.**

新架构下，打开 App 会发生什么

1. 点击 App 图标
1. Fabric 加载 Native 侧
1. 然后通知 JS 线程 Native 侧准备好了, JS 侧会加载所有的 bundle JS 文件, 里面包含了所有的 JS 和 React 逻辑组件
1. JS 通过一个 Native 函数的引用, 调用到 Fabric, 同时 Shadow Node 创建一个和以前一样的 UI 树
1. Yoga 进行布局计算, 把基于 Flexbox 的布局转化为 Native 端的布局
1. Fabric 执行操作并显示 UI

没有了 Bridge 提升了性能,可以用同步的方式进行操作, 启动时间也快, App 也将更小。
### Compose Multiplatform
#### KMP
首先应该了解Kotlin Multiplatform.

Kotlin 在 Android 世界中广受欢迎，但它并非专为 Android 设计的技术。

Kotlin 的初衷是创建一种通用语言，能够与其他编程语言兼容 ，从而用于构建不同平台（而非仅限于 Android）的应用程序。

所以，Kotlin 从设计上来说就是一门多平台语言。

Kotlin Multiplatform是一种跨平台开发技术，它允许开发者使用Kotlin语言编写代码，并在多个平台上运行，包括iOS、Android、Web、桌面等。不同的平台可以共享相同的代码库，从而减少了开发成本和维护成本。

![](/assets/img/blog/blogs_kmp_process.webp){:width ="360" height="250" loading="lazy"}

Kotlin 编译器在Android和IOS上生成对应平台特有文件的流程，它包含以下两部分：
* 前端- 它将 Kotlin 代码转换为 IR（中间表示）。该 IR 能够通过下文所述的后端转换为机器可执行的原生代码。
* 后端- 它将 IR 转换为机器可执行的原生代码。这得益于 JetBrains 构建的 Kotlin/Native 基础架构。对于 Android，它将 IR 转换为 Java 字节码；对于 iOS，它将 IR 转换为 iOS 原生机器可执行代码。

#### 支持转译成哪些语言？
Kotlin 编译器将源代码作为输入，并生成一组特定于平台的二进制文件。在编译多平台项目时，它可以从同一份代码生成多个二进制文件。例如，编译器可以.class从同一个 Kotlin 文件生成 JVM 文件和原生可执行文件。

目前，其中三种语言的支持最为成熟， JetBrains 正在不断努力扩展支持范围。

* Java：这就是 Kotlin 在Android设备上运行的方式，转为class文件，在JVM平台上运行，但我们也可以在桌面或服务器应用程序中使用它。
* JavaScript ：对js语言的支持，使我们能够在Web应用程序中使用 Kotlin ，包括前端和后端应用程序。
* C / Objective-C：这样，我们就可以访问所有基于Linux的平台和 Apple 操作系统，例如iOS设备、iPadOS、macOS、tvOS和watchOS 。而且由于 Objective-C 可以与 Swift 兼容，因此我们也可以在Swift项目中使用 Kotlin 。

并非所有 Kotlin 代码都能编译到所有平台。Kotlin 编译器会阻止您在通用代码中使用特定于平台的函数或类，因为这些代码无法编译到其他平台。

例如，您无法使用java.io.File公共代码中的依赖项。它是 JDK 的一部分，而公共代码也会编译为本机代码，而 JDK 类在本机代码中不可用。

![](/assets/img/blog/blogs_kmp_build_targets.svg)

Kotlin的业务逻辑代码向目标平台的代码转换，是在编译器中进行的。所以在应用程序上架之前，用于分发的软件包里面，实际上和目标平台的Native应用程序没有任何区别。

#### KMP开发模式
如果某些功能无法在通用代码中实现，可以使用KMP独特的 **expect / actual** 声明机制，在commonMain中声明需要实现的功能，在原生平台的代码中使用系统特有的实现。

如果我们想使用某些系统 API 或原生工具，我们也可以直接到原生文件夹中写native平台代码。这种高度灵活性使得 KMP 的风险比其他解决方案更低。

Kotlin官方文档推荐的几种usecases

![](/assets/img/blog/blogs_kmp_usecases.png)

比较流行KMP应用模式一般是将网络请求，数据库存储等使用KMP改写，在UI逻辑上仍然使用之前的代码来实现，这样可以减少重复代码的编写，提高开发效率。有最大限度保留旧的用户交互逻辑和功能。
#### Skia引擎
**Skia**是一个用C++编写的开源高性能二维图形库。它本质上是一个图形引擎，为在各种硬件和软件平台上绘制文本、几何图形（形状）和图像提供了通用的 API。

Skia 库的主要特点有：
* 二维图形：专门绘制二维图形。
* 跨平台：Skia 可在各种操作系统上无缝运行，包括 Windows、macOS、iOS、Android、Linux（Ubuntu、Debian、openSUSE、Fedora），甚至网络浏览器（通过 WebAssembly）。
* 开源：由谷歌赞助和管理，Skia 采用 BSD 自由软件许可证，允许任何人使用和贡献。
* 核心图形引擎：它是许多流行产品的基本图形引擎，包括
    * 谷歌浏览器和 ChromeOS
    * 安卓系统
    * Flutter（谷歌用于构建本地应用程序的用户界面工具包）
    * 火狐浏览器
    * LibreOffice（从 7.0 版开始）
    * 以及其他各种应用程序和框架。

Skia 的**核心优势**在于它能在各种硬件上高效地渲染这些图形。它通过支持各种后端渲染技术来实现这一目标：
* GPU 加速：** 对于现代设备，Skia 可以利用图形处理器（GPU）进行硬件加速渲染。为此，Skia 可将其内部绘图命令转换为对 GPU 应用程序接口的调用，例如：
    * OpenGL ES / OpenGL：** 一种广泛应用于 2D 和 3D 图形的 API。
    * AGLE：** 兼容性层，可将 OpenGL ES 调用转换为特定供应商的本地 API（如 Windows 上的 Direct3D 或 macOS 上的 Metal），以获得一致的性能。
    * Vulkan：** 一种现代高性能图形和计算 API。
    * Metal：** 苹果用于 iOS 和 macOS 的底层图形 API。
* CPU 软件光栅化：** 在 GPU 加速不可用或不可取的情况下（如某些服务器或特定的渲染需求），Skia 可以退回到 CPU 上的软件渲染。这包括直接在 CPU 上将矢量图形光栅化为像素。
 * PDF/SVG 输出：** Skia 还可以渲染成 PDF 或 SVG 等格式，这些格式基于矢量，可按比例缩放而不会降低质量。

#### CMP的绘制
这是 Skia 发挥关键作用的地方。Compose Multiplatform **不会直接使用每个平台原生的 UI 组件（例如 Android 上的 View 或 iOS 上的 UIKit 控件）来绘制**。相反，它采取了**像素渲染 (pixel-painting)** 的方式，这与 Flutter 的工作方式类似。

JetBrains 开发了一个名为 **Skiko** 的 Kotlin Multiplatform 库。Skiko 是 Skia 的 Kotlin 包装器，它提供了 Kotlin API 来与底层的 Skia 图形库进行交互。

在绘制阶段，Compose Multiplatform 会将布局阶段计算出的 UI 元素的形状、颜色、文本、图片等信息，转化为一系列 **Skia 绘制命令**。这些命令包括：
* 绘制矩形、圆形、线条、路径等几何图形。
* 绘制文本（包括字体、大小、颜色等）。
* 绘制图片。
* 应用变换（平移、旋转、缩放）和滤镜效果。

无论是在桌面、iOS 还是 Web 上，Compose Multiplatform 都会将这些 Skia 绘制命令传递给 Skia 库。Skia 再根据目标平台的不同，选择最合适的底层图形 API 进行渲染：
    
**桌面 (Windows, macOS, Linux):**， Skia 可以直接利用 OpenGL、Direct3D 或 Vulkan (如果可用) 等 GPU API 进行硬件加速渲染。

**iOS:** ，Compose Multiplatform 在 iOS 上也使用 Skia 进行画布渲染。这意味着它不会使用 iOS 原生的 UIKit 视图，而是直接通过 Skia 绘制到屏幕上。Skia 会利用 Metal (Apple 的图形 API) 或 OpenGL ES (较旧的 API) 进行渲染。

**Android:** ，Jetpack Compose (Compose Multiplatform 的 Android 部分) 本身就使用 Skia 作为其底层的渲染引擎。所以，这部分是无缝衔接的。

**Web:** ，在 Web 平台上，Compose Multiplatform 通常会利用 WebAssembly 和 HTML Canvas 元素。Skia 编译为 WebAssembly 并在 Canvas 上绘制像素。

**CMP特有性能优化:**
* **增量渲染:** Compose 只有在 UI 状态发生变化时才会重新执行受影响的 Composable 函数，并只更新屏幕上发生变化的部分。
* **GPU 加速:** 通过 Skia 及其对底层图形 API 的支持，Compose Multiplatform 能够充分利用 GPU 进行硬件加速渲染，从而实现流畅的动画和高性能的 UI。
* **缓存:** Skia 会在内部进行各种优化，例如图形指令的缓存，以减少重复计算。

**总结来说，Compose Multiplatform 绘制组件的流程是：**

1.  开发者通过 Kotlin 的 Composable 函数声明 UI。
2.  Compose 运行时构建 UI 元素的组合树。
3.  Compose 进行测量和布局，确定每个元素的尺寸和位置。
4.  将 UI 元素转换为 Skia 绘制命令。
5.  通过 Skiko 库，这些 Skia 绘制命令被传递给底层的 Skia 图形库。
6.  Skia 根据目标平台的特性，利用 GPU (通过 OpenGL/Direct3D/Vulkan/Metal 等) 或 CPU 进行像素渲染，最终将 UI 呈现在屏幕上。

这种方法使得 Compose Multiplatform 能够提供一致的 UI 外观和行为，无论应用程序运行在哪个平台上，同时也能利用平台原生的图形性能。

## 性能
在比较 Flutter、React Native 和 Compose Multiplatform (CMP) 的性能时，需要考虑它们各自的架构和设计哲学，因为这直接影响了它们的运行时性能。以下是这三者在性能方面的对比：

### 1. Flutter

**架构核心:** Flutter 使用 **Dart 语言**，并拥有自己的渲染引擎，该引擎直接通过 **Skia** 图形库（在最新版本中，桌面和移动端已逐步转向 **Impeller** 渲染引擎）绘制 UI。这意味着 Flutter 不依赖于平台原生的 UI 组件。Dart 代码在发布时会被编译为**原生机器码 (ahead-of-time, AOT)**。

**性能特点:**

* **接近原生性能:** 由于直接编译为机器码并使用自己的渲染引擎，Flutter 在 UI 渲染和动画方面通常能达到与原生应用非常接近的性能。它能够以 60 FPS (甚至 120 FPS) 的流畅度运行复杂动画和高负载 UI。
* **无桥接开销:** Flutter 消除了 JavaScript 桥接的开销，因为 Dart 代码直接与底层平台通信，避免了在 JavaScript 和原生代码之间进行序列化和反序列化的性能瓶颈。
* **启动时间:** 相对于原生应用，Flutter 应用的启动时间可能会略长，因为它需要初始化 Flutter 引擎。然而，Google 正在不断优化这方面。
* **内存占用:** 在某些基准测试中，Flutter 应用的内存占用可能略高于原生应用，因为它捆绑了自己的引擎和渲染器。
* **包大小:** Flutter 应用的包大小通常比原生应用大，因为它包含了 Flutter 引擎和 Dart 运行时。

**总结:** Flutter 在 UI 渲染和动画流畅度方面表现出色，适合需要复杂、高度定制 UI 和高性能动画的应用。

### 2. React Native

**架构核心:** React Native 使用 **JavaScript/TypeScript**。它不直接绘制 UI，而是通过一个 **JavaScript 桥接 (Bridge)** 与平台原生的 UI 组件进行通信。当 JavaScript 端更新状态时，通过桥接将指令发送到原生 UI 线程，由原生组件进行渲染。

**性能特点:**

* **有桥接开销:** 传统的 React Native 架构中，JavaScript 线程和原生 UI 线程之间的通信需要通过桥接，这会引入一定的序列化和反序列化开销，尤其是在频繁更新 UI 或进行大量数据传输时，可能导致性能瓶颈和 UI 卡顿。
* **原生组件渲染:** 优势在于使用原生 UI 组件，能够提供原生的外观和感觉，但在需要高度定制的 UI 或跨平台像素级一致性时，可能需要额外的努力。
* **新架构 (Fabric & TurboModules):** React Native 正在积极推广其“新架构”，其中包含 **Fabric 渲染系统**和 **TurboModules**。
    * **Fabric:** 旨在解决旧桥接的性能问题，通过 C++ 层实现 JavaScript 和原生之间的同步通信，减少了桥接开销，提高了 UI 响应速度和动画流畅度。它也支持并发渲染。
    * **TurboModules:** 允许原生模块按需加载，从而改善了应用启动时间。
    * **JSI (JavaScript Interface):** 替换了旧的桥接，允许 JavaScript 直接调用 C++ 代码，从而实现更高效的通信。
* **启动时间:** 相对于原生应用，React Native 应用的启动时间可能较长，尤其是在加载 JavaScript 包时。新架构旨在改善这一点。
* **内存占用:** 内存占用通常介于原生和 Flutter 之间，因为需要 JavaScript 运行时和原生组件。

**总结:** React Native 在性能方面受 JavaScript 桥接的限制，但在新架构（Fabric、TurboModules、JSI）的推动下，其性能正在显著提升，尤其是在复杂 UI 和动画方面。对于需要快速开发、且对绝对原生性能要求不那么极致的应用，React Native 仍是一个强有力的选择。

### 3. Compose Multiplatform (CMP)

**架构核心:** Compose Multiplatform 基于 **Kotlin Multiplatform (KMP)** 技术，使用 **Kotlin** 语言。它在 Android 上复用 Google 的 Jetpack Compose，而在其他平台（iOS、桌面、Web）上，通过 **Skiko** (Skia 的 Kotlin 包装器) 直接调用 **Skia** 图形库进行 UI 绘制，与 Flutter 的像素渲染方式类似。Kotlin 代码会编译为原生二进制文件。

**性能特点:**

* **接近原生性能:**
    * **Android:** 直接使用 Jetpack Compose，其性能与原生 Android UI 相当，并受益于 Android 系统内置的 Skia 库。
    * **iOS/桌面:** 通过 Skiko/Skia 直接绘制 UI，避免了桥接开销，因此在 UI 渲染和动画方面能达到接近原生应用的性能。
    * **AOT 编译:** Kotlin 代码可以编译为原生机器码（AOT 编译），进一步提升了运行时性能。
* **启动时间:** 在 Android 上，与 Jetpack Compose 应用类似，启动时间通常良好。在 iOS 上，由于需要捆绑 Skia 库（不像 Android 可以依赖系统内置），可能会增加一点启动时间，但总体上仍然表现优秀。
* **内存占用:** 通常表现良好，与原生应用或 Jetpack Compose 应用类似。
* **包大小:** 在 Android 上，CMP 应用的包大小与 Jetpack Compose 应用类似。在 iOS 和桌面端，由于需要捆绑 Skia 库，包大小会比原生应用略大，但通常比 Flutter 应用小。

**总结:** Compose Multiplatform 在性能上非常具有竞争力。它在 Android 上直接受益于 Jetpack Compose 的原生整合，而在其他平台则通过 Skia 提供了高性能的像素渲染。对于追求原生性能和统一代码库的 Kotlin 开发者来说，CMP 是一个非常吸引人的选择。


## 适用场景
**对极致性能要求高、或拥有复杂定制 UI 的应用:** **Flutter** 和 **Compose Multiplatform** 通常是更好的选择。它们通过直接绘制像素来绕过原生组件的限制，提供高度优化的渲染管道。

**对开发速度和 Web 开发者友好度有高要求、或希望逐步迁移现有原生应用:** **React Native** (尤其是新架构下) 仍是强有力的竞争者。其庞大的社区和成熟的生态系统也是巨大优势。

**对于 Kotlin 开发者、希望最大化代码共享并获得接近原生性能，同时能够方便地与现有原生代码互操作的项目:** **Compose Multiplatform** 提供了非常吸引人的平衡点。

最终的选择取决于你的 **项目需求、团队技术栈、以及对性能、开发速度和原生体验** 的优先级。随着这三个框架的不断发展和优化，它们之间的 **性能差距也在逐渐缩小** 。

## 外部讨论
在热门论坛Reddit上，某篇帖子如下：
Compose Multiplatform 与 Flutter

> 您好，我正在决定将我的事业重心放在这两者之间：
Dart（Flutter）VS Kotlin（KMP 和 CMP）
因为我也想做独立移动应用程序，但同时也想担任移动工程师一职。
我的工作地点在美国，所以这里的 Flutter 职位比较少。
我知道 CMP 在 iOS 上还不稳定，但它是未来的趋势吗？
我喜欢在 Ktor 的后端也可以使用 Kotlin。
但 Flutter 有生态系统和热重载功能，所以我很纠结到底要继续使用哪一种......

网友1：
> 您正在询问 kotlin Reddit，所以这里会有一些偏见。
但抛开这些不谈，学习 KMP 比学习 Flutter 更接近学习原生 Android。这就是关键优势。安卓开发者可以轻松地将他们的知识迁移到 Kotlin Multiplatform，Compose 与他们在安卓中使用的完全相同，因此可以相互映射 与 KMP 相比，Flutter 有 2-3 年的先发优势，而且拥有更成熟的生态系统。但是，您将在 Android 和 iOS SDK 的基础上学习 Flutter SDK，而原生开发人员的知识迁移学习曲线更大。

网友2：
> 使用 Compose 的 Kotlin 多平台令人惊叹。不过我不确定是否有适合它的工作，如果你正在考虑的话。这是更新颖的技术。但通过它，会有 Android 的机会。

网友3：
> 我使用过 jetpack compose 和 flutter，flutter 非常缺乏优秀的库，而且创建一个小部件需要大量的模板，令人厌恶。
你最好还是学习这两个平台的真正原生程序，我发现 Flutter 从未真正解决开发 iOS 应用程序的痛苦。
从安全角度来看，Flutter 也有点弱，因为你无法真正控制许多东西与操作系统的交互方式（如安全存储）或字符串的永久性，而且大多数代码扫描程序都不包括 dart 或 pub 包。
你也得不到一个合适的集成开发环境，flutter 支持是在 android studio 上附加的。
试着在 flutter 中做一个懒列表，然后再在 jetpack compose 中做，这就是它们生态系统的完美体现。

就我个人的情况，专业为Android开发，对于Kotlin和Jetpack Compose的写法，架构设计，已经是比较熟悉了，如果有做跨平台的需求，在业务不是太复杂的情况下，使用CMP几乎是最佳选择。所以这个跨平台的能力对于熟悉这两个技术的Android开发可以说是买一送一，拿起电脑，稍微看看文档就可以写功能。

国内已经有Bilibili，快手在使用KMP来重构自己的产品，腾讯甚至基于CMP自己改了一套Kuikly，所以站在发展的角度看，我认为CMP日后的成熟度和公司接受度，说不定可以超过Flutter。