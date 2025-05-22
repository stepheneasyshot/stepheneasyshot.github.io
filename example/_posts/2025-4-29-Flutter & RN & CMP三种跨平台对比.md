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
首先应该了解Kotlin Multiplatform.

Kotlin Multiplatform是一种跨平台开发技术，它允许开发者使用Kotlin语言编写代码，并在多个平台上运行，包括iOS、Android、Web、桌面等。不同的平台可以共享相同的代码库，从而减少了开发成本和维护成本。

![](/assets/img/blog/blogs_kmp_process.webp){:width ="300" height="250" loading="lazy"}

Kotlin 编译器在Android和IOS上生成对应平台特有文件的流程，它包含以下两部分：
* 前端- 它将 Kotlin 代码转换为 IR（中间表示）。该 IR 能够通过下文所述的后端转换为机器可执行的原生代码。
* 后端- 它将 IR 转换为机器可执行的原生代码。这得益于 JetBrains 构建的 Kotlin/Native 基础架构。对于 Android，它将 IR 转换为 Java 字节码；对于 iOS，它将 IR 转换为 iOS 原生机器可执行代码。

后续参考文档：
https://www.thedroidsonroids.com/blog/what-is-kotlin-multiplatform