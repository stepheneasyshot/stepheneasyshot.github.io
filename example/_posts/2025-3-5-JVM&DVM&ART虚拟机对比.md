---
layout: post
description: > 
  本文介绍了JVM虚拟机，Dalvik虚拟机还有ART虚拟机三者之间不同特点的对比
image: 
  path: /assets/img/blog/blogs_cover_android_vitual_machine.png
  srcset: 
    1920w: /assets/img/blog/blogs_cover_android_vitual_machine.png
    960w:  /assets/img/blog/blogs_cover_android_vitual_machine.png
    480w:  /assets/img/blog/blogs_cover_android_vitual_machine.png
accent_image: /assets/img/blog/blogs_cover_android_vitual_machine.png
excerpt_separator: <!--more-->
sitemap: false
---
# JVM&DVM&ART虚拟机对比
去年通读了深入理解Java虚拟机，对JVM的一系列特性有了系统的了解，然而作为Android开发，却还没有对Android平台特有的两代虚拟机做更为细致的学习，属实有点说不过去。在车机Android系统app的开发中，几乎涉及不到apk的动态安装卸载，一般是参与系统集成编译完就发布镜像，烧写后一直运行。所以对此专题，了解甚少。

JVM跨平台特性的原理是 **字节码** ，字节码是一种中间代码，它是一种 **面向虚拟机** 的代码，而不是面向硬件的代码。每个JRE编译的时候针对每个平台编译，因此下载JRE（JVM、Java核心类库和支持文件）的时候是分平台的，JVM的作用是把平台无关的.class里面的字节码翻译成平台相关的机器码，来实现跨平台。 
## 为何Android不直接使用JVM
从问题寻找答案，总是一个有效的学习方法，为什么Android平台当年不直接使用JVM作为硬件抽象之上的虚拟机供应用运行呢？

在早期（Android 4.4之前），Android使用的是Dalvik虚拟机。不直接使用JVM，主要有以下几个原因:

* 资源限制: 早期的移动设备内存和存储空间都很有限。 **JVM需要较大的内存和存储空间** ，不适合资源受限的移动设备。Dalvik虚拟机针对移动设备进行了优化,更加轻量级。

* 性能考虑: JVM主要针对服务器和桌面环境优化，移动设备上，可能没有桌面和服务器上那么强大的性能，故JVM在移动设备上性能表现可能不佳。 **Dalvik虚拟机采用了寄存器架构** ，相比 **JVM的栈架构** 在移动设备上有更好的性能表现。

* 许可证问题: JVM受Oracle公司的许可证限制，Android为了避免潜在的法律风险，选择开发自己的虚拟机。这也是一个为人乐道的历史原因。

* 定制化需求: Android需要一个 **更加定制化的解决方案** 来满足移动平台的特殊需求，如电源管理、内存管理等。Dalvik虚拟机可以更好地与Android系统集成。

* 字节码格式: Dalvik使用的.dex格式比Java的.class文件更紧凑，更适合移动设备有限的存储空间。

* 多进程架构: Android的应用运行在独立的进程中，每个进程有自己的Dalvik虚拟机实例，这种架构更适合移动操作系统的需求。

* 安全性考虑: Dalvik虚拟机在安全性方面也有一定的优势，如对内存访问的控制和对异常的处理。

事实证明，Android使用Dalvik虚拟机的选择是正确的，如果直接使用JVM，在以上这些因素的影响下，Android系统可能真的发展不起来。
## Dalvik虚拟机
JVM虚拟机里面，解释运行的是class文件，而在Dalvik虚拟机里面，解释运行的是dex（即“Dalvik Executable”）文件。
### dex文件
回顾一下class文件的组成：

* 魔数：用于标识文件类型，通常为0xCAFEBABE。

* 版本号：用于标识class文件的版本号，包括主版本号和次版本号。

* 常量池：用于存储常量，包括字符串、类名、字段名、方法名等。

* 访问标志：用于标识类的访问权限，如public、final等。

* 类信息：包括类名、父类名、接口名等。

* 字段信息：包括字段名、字段类型、访问标志等。

* 方法信息：包括方法名、方法类型、访问标志等。

* 属性信息：用于存储类的附加信息，如注解、泛型信息等。

为了减小执行文件的体积，安卓的Dalvik虚拟机选择dex文件来替代class文件。

class文件合并为dex文件这个过程，主要由 `Android SDK` 中的dx工具（在较新的Android Gradle插件中被d8工具替代）完成。通过这个转换过程，多个class文件被合并转换成一个更紧凑、更适合移动设备的dex文件，从而减小了应用的大小，提高了加载和执行效率。

class文件转换成dex文件的过程称为"dexing"，主要通过以下步骤完成：

* 收集所有的class文件： 首先，系统会收集所有需要转换的class文件。这些文件通常来自于你的Java源代码编译后的结果，以及你项目中使用的库。

* 解析class文件： 每个class文件都会被解析，提取其中的常量池、方法、字段等信息。

* 合并常量池： dex格式的一个主要优化是合并所有class文件的常量池。相同的字符串、类型和方法引用只会在dex文件中出现一次，大大减少了重复数据。

* 重新编码数据： class文件中的数据会被重新编码为dex格式。这包括将Java字节码转换为Dalvik字节码，以及重新组织类、方法和字段的结构。

* 优化： 在转换过程中，还会进行一些优化，例如删除未使用的字段和方法，内联一些简单的方法调用等。

* 生成dex文件： 最后，所有处理后的数据被写入到一个或多个dex文件中。一个dex文件可以包含多个class的内容。

* 验证： 生成的dex文件会经过验证，确保其格式正确，可以被Dalvik虚拟机（或ART运行时）正确加载和执行。

格式对比如下：

![](/assets/img/blog/blogs_dalvik_class2dex.png)

### JVM和Dalvik运行流程
#### JVM基于栈操作
每个Java线程都有自己的 **Java虚拟机栈** ，用于存储方法调用的信息，包括局部变量、部分结果和方法调用/返回等。

当一个方法被调用时，会在栈上创建一个新的 **栈帧（Stack Frame）** 。栈帧包含局部变量表，操作数栈，指向运行时常量池的引用，方法返回地址等信息。

JVM使用基于栈的指令集，这意味着大多数操作都是通过压栈和出栈来完成的。例如：

```java
int foo(int a, int b) {
  return (a + b) * (a - b);
}
```

转换后的字节码指令为：

```
  int foo(int, int);
    Code:
       0: iload_1 // 将局部变量a压入栈
       1: iload_2 // 将局部变量b压入栈
       2: iadd    // 弹出栈顶的两个元素，相加，结果压入栈
       3: iload_1 // 将局部变量a压入栈
       4: iload_2 // 将局部变量b压入栈
       5: isub    // 弹出栈顶的两个元素，相减，结果压入栈
       6: imul    // 弹出栈顶的两个元素，相乘，结果压入栈
       7: ireturn // 返回栈顶元素作为方法结果
```

#### Dalvik基于寄存器操作
在Dalvik虚拟机中，寄存器是虚拟的，不同于物理CPU中的硬件寄存器。这些虚拟寄存器实际上是存储在内存中的，用于存储局部变量、方法参数、中间计算结果等。

Dalvik虚拟机使用基于寄存器的指令集，这意味着大多数操作都是通过寄存器来完成的。还是上面同样的计算方法：

```java
int foo(int a, int b) {
  return (a + b) * (a - b);
}
```

转换后的字节码指令

```
0000: add-int  v0, v3, v4
0002: sub-int  v1, v3, v4
0004: mul-int/2addr  v0, v1
0005: return  v0
```

> add-int是一个需要两个操作数的指令，其指令格式是：add-int vAA, vBB, vCC。其指令的运算过程，是将后面两个寄存器中的值进行（加）运算，然后将结果放在（第一个）目标寄存器中。其余类似

计算流程如下图：

![](/assets/img/blog/blogs_dalvik_calculate.png)

### 内存对比
一个JVM进程所处的Java虚拟机的内存可以用下面这张热门图片概括：

![jvm_ram](/assets/img/blog/blogs_jvm_ram_simple.png){:width ="500" height="280" loading="lazy"}

Dalvik虚拟机的内存模型略有不同。

#### Dalvik内存模型
Dalvik虚拟机用来分配对象的堆划分为两部分，一部分叫做Active Heap，另一部分叫做Zygote Heap。

Android系统启动后，会有一个Zygote进程创建第一个Dalvik虚拟机，它只维护了一个堆。以后启动的所有应用程序进程是被Zygote进程fork出来的，并都持有一个自己的Dalvik虚拟机。在创建应用程序的过程中，Dalvik虚拟机采用 **COW策略** 复制Zygote进程的地址空间。

> COW策略，即Copy-On-Write，一开始的时候（未复制Zygote进程的地址空间的时候），应用程序进程和Zygote进程共享了同一个用来分配对象的堆。当Zygote进程或者应用程序进程对该堆 **进行写操作时，内核就会执行真正的拷贝操作** ，使得Zygote进程和应用程序进程分别拥有自己的一份拷贝，这就是所谓的COW。因为copy是十分耗时的，所以必须尽量避免copy或者尽量少的copy。

为了实现这个目的，当创建第一个应用程序进程时，会将已经使用了的那部分堆内存划分为一部分作为 **Zygote堆** ，还没有使用的堆内存划分为另外一部分称为 **Active堆** 。

Zygote Heap 堆内存中, 有一部分区域的内存是只读的, 如系统相关的库, 共享库, 预置库, 这些内存数据所有应用公用。这些预加载的类、资源和对象可以在Zygote进程和应用程序进程中做到 **长期共享** 。这样既能减少拷贝操作，还能减少对内存的需求。

这样只需把zygote堆中的内容复制给应用程序进程就可以了。以后无论是Zygote进程，还是应用程序进程，当 **它们需要分配对象的时候，都在Active堆上进行** 。这样就可以使得Zygote堆尽可能少地被执行写操作，因而就可以减少执行写时拷贝的操作。

### 垃圾回收算法
总的来说，Dalvik的垃圾回收算法相比JVM更简单，但更专注于移动设备的需求。它追求更小的内存占用、更快的应用启动时间和更低的电量消耗。

JVM垃圾回收大概包括标记清除，分代收集，复制算法等，而Dalvik虚拟机的垃圾回收主要采用并发标记-清除（Concurrent-Mark-Sweep）算法。

![](/assets/img/blog/blogs_dvm_gc.png)

#### Mark 阶段
通过递归，从 GC Roots 开始标记被引用的对象。为了避免 Stop-The-World，采用 GC 线程和其他线程并发执行，分为两步：

* 只标记 GC Roots 对象（在 GC 过程开始的时刻，被全局变量、栈变量、寄存器对象引用的对象）。这个阶段会短时间 Stop-The-World，防止这些 GC Roots 对象在此过程中再去引用其他对象。
* 通过这些 GC Roots 对象的引用关系，找到并标记其他正在使用的对象。这个阶段并发执行，需要把其他线程对对象的修改记录到 Card Table（未修改为 clean，修改过为 dirty）。执行结束后，需要再次使用 GC 线程对这些发生修改的对象再次标记（Stop-The-World，量少，速度快）。

DVM内存模型图

![](/assets/img/blog/blogs_dvm_mem_model.png)

#### Sweep 阶段
GC 线程和用户线程同时并发执行，同时 GC 线程开始对为标记的区域做清扫，回收所有的垃圾对象。

清除 Live Bitmap 中有，Mark Bitmap 中没有的对象，即为垃圾对象。

#### 算法缺点
对 CPU 资源敏感：在并发阶段，它虽然不会导致用户线程停顿，但会因为占用了一部分线程（或者说 CPU 资源）而导致应用程序变慢，总吞吐量会降低。
无法处理浮动垃圾：在并发清除时，用户线程新产生的垃圾，称为浮动垃圾。

