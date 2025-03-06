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
去年通读了深入理解Java虚拟机，对JVM的一系列特性有了系统的了解，然而作为Android开发，却还没有对Android平台特有的两代虚拟机做更为细致的学习，属实有点说不过去。

JVM跨平台特性的原理是 **字节码** ，字节码是一种中间代码，它是一种 **面向虚拟机** 的代码，而不是面向硬件的代码。每个JRE编译的时候针对每个平台编译，因此下载JRE（JVM、Java核心类库和支持文件）的时候是分平台的，JVM的作用是把平台无关的.class里面的字节码翻译成平台相关的机器码，来实现跨平台。 

## 为何Android不直接使用JVM
从问题寻找答案，总是一个有效的学习方法，为什么Android平台当年不直接使用JVM作为硬件抽象之上的虚拟机供应用运行呢？在早期（Android 4.4之前），Android使用的是Dalvik虚拟机。

不直接使用JVM，主要有以下几个原因:

* 资源限制: 早期的移动设备内存和存储空间都很有限。 **JVM需要较大的内存和存储空间** ，不适合资源受限的移动设备。Dalvik虚拟机针对移动设备进行了优化,更加轻量级。

* 性能考虑: JVM主要针对服务器和桌面环境优化，移动设备上，可能没有桌面和服务器上那么强大的性能，故JVM在移动设备上性能表现可能不佳。 **Dalvik虚拟机采用了寄存器架构** ，相比 **JVM的栈架构** 在移动设备上有更好的性能表现。

* 许可证问题: JVM受Oracle公司的许可证限制，Android为了避免潜在的法律风险，选择开发自己的虚拟机。这也是一个为人乐道的历史原因。

* 定制化需求: Android需要一个 **更加定制化的解决方案** 来满足移动平台的特殊需求，如电源管理、内存管理等。Dalvik虚拟机可以更好地与Android系统集成。

* 字节码格式: Dalvik使用的.dex格式比Java的.class文件更紧凑，更适合移动设备有限的存储空间。

* 多进程架构: Android的应用运行在独立的进程中，每个进程有自己的Dalvik虚拟机实例，这种架构更适合移动操作系统的需求。

* 安全性考虑: Dalvik虚拟机在安全性方面也有一定的优势，如对内存访问的控制和对异常的处理。

事实证明，Android使用Dalvik虚拟机的选择是正确的，如果直接使用JVM，在以上这些因素的影响下，Android系统可能真的发展不起来。

## Dalvik虚拟机
JVM虚拟机里面，解释运行的是class文件，而在Dalvik虚拟机里面，解释运行的是dex文件。
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

为了减小执行文件的体积，安卓的Dalvik虚拟机选取的dex文件来替代class文件。

class文件合并为dex文件这个过程，主要由 `Android SDK` 中的dx工具（在较新的Android Gradle插件中被d8工具替代）完成。通过这个转换过程，多个class文件被合并into一个更紧凑、更适合移动设备的dex文件，从而减小了应用的大小，提高了加载和执行效率。

class文件转换成dex文件的过程称为"dexing"，主要通过以下步骤完成：

* 收集所有的class文件： 首先，系统会收集所有需要转换的class文件。这些文件通常来自于你的Java源代码编译后的结果，以及你项目中使用的库。

* 解析class文件： 每个class文件都会被解析，提取其中的常量池、方法、字段等信息。

* 合并常量池： dex格式的一个主要优化是合并所有class文件的常量池。相同的字符串、类型和方法引用只会在dex文件中出现一次，大大减少了重复数据。

* 重新编码数据： class文件中的数据会被重新编码为dex格式。这包括将Java字节码转换为Dalvik字节码，以及重新组织类、方法和字段的结构。

* 优化： 在转换过程中，还会进行一些优化，例如删除未使用的字段和方法，内联一些简单的方法调用等。

* 生成dex文件： 最后，所有处理后的数据被写入到一个或多个dex文件中。一个dex文件可以包含多个class的内容。

* 验证： 生成的dex文件会经过验证，确保其格式正确，可以被Dalvik虚拟机（或ART运行时）正确加载和执行。

