---
layout: post
description: > 
  本文介绍了JNI调用的运行流程
image: 
  path: /assets/img/blog/blogs_jni.png
  srcset: 
    1920w: /assets/img/blog/blogs_jni.png
    960w:  /assets/img/blog/blogs_jni.png
    480w:  /assets/img/blog/blogs_jni.png
accent_image: /assets/img/blog/blogs_jni.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】JNI调用是如何运行的
去年学习了整个Android平台的JNI实现流程以及基础类型，引用，多线程，核心指针和JavaVM的相关知识。

[Android JNI开发](./2024-10-21-Android%20JNI开发.md)

现在从架构层面，了解一下底层的运行，调用链路。

主要分析目标是 `Android` 平台。先简单回顾一下 JNI 的开发流程和动态库的生成流程。

首先，理解 **JNI (Java Native Interface)** 的核心作用至关重要。JNI 并不是一种编程语言，而是一套**规范 (Specification)**。它定义了：

* **JVM (Java Virtual Machine) 如何调用 Native 代码：** 包括函数命名约定、参数传递、返回值处理等。
* **Native 代码如何与 JVM 交互：** 比如创建 Java 对象、调用 Java 方法、访问 Java 字段、抛出 Java 异常等。
* **数据类型映射：** Java 类型（`int`, `String`, `Object` 等）与 C/C++ 类型（`jint`, `jstring`, `jobject` 等）之间的转换规则。

这套规范保证了不同 JVM 实现（如 Android 的 ART/Dalvik）和不同操作系统（Linux/Windows/macOS）上的 Java 代码都能以相同的方式与 Native 代码交互。
## so 动态库生成
一个典型的 JNI 项目模组结构是下面这样的：

![](/assets/img/blog/blogs_jni_module_template.png)

现在很多原生C++项目都采用CMake工具来构建编译系统，android平台开发 JNI 也是。CMake 本身并不是一个编译工具，而是一个跨平台的、开源的自动化构建系统。它不直接编译你的代码，而是根据你的 CMakeLists.txt 文件来生成特定平台和编译工具（如 Makefiles、Ninja 或 Visual Studio 项目文件）的构建脚本。

想象一下，如果你的项目需要在 Windows、Linux、macOS 甚至 Android 上编译，每种平台可能有不同的编译器和构建工具。手动为每种环境编写构建脚本会非常复杂且容易出错。CMake 就是为了解决这个问题而生，它提供了一套统一的语法来描述你的项目。
### CMakeLists.txt
`CMakeLists.txt` 文件是什么呢？它是一个构建系统的配置文件。 形象地来概括，`CMakeLists.txt` 就是你告诉 CMake **"我的项目长这样，你需要用这些源文件，链接这些库，编译成这种类型（可执行文件或库），并且用这些特殊的编译选项"** 的说明书。它极大地简化了跨平台项目的构建管理。

它的主要作用包括 **定义项目结构和属性** ， **管理依赖和库** ， **设置编译选项和宏** ， **查找外部包和组件** ， **配置和生成构建文件** ， **定义安装规则**。

例如：

```cmake
# CMake 最低版本要求
cmake_minimum_required(VERSION 3.10.2)

# 定义项目名称
project("my_native_lib")

# 查找 Android Log 库
find_library(log-lib log)

# 添加一个共享库目标，命名为 "my_native_lib"
# 并指定它的源文件
add_library(                  # 添加一个库
              my_native_lib   # 库的名称
              SHARED          # 共享库
              src/main/cpp/native-lib.cpp ) # 源文件路径

# 链接日志库到你的目标库中
target_link_libraries(        # 指定需要链接的库
                       my_native_lib  # 你的目标库
                       ${log-lib} )   # Android Log 库

# 可选：设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

# 可选：添加包含目录
target_include_directories(my_native_lib PUBLIC
                           src/main/cpp/include)
```

在这个例子中：
* `project()` 定义了项目的名称。
* `find_library()` 查找了 Android NDK 提供的 `log` 库。
* `add_library()` 定义了一个名为 `my_native_lib` 的**共享库**，并指定了它的源文件 `native-lib.cpp`。这是在 Android 上生成 `.so` 文件的关键。
* `target_link_libraries()` 将 `log-lib` 链接到 `my_native_lib` 中，这样你的 C++ 代码就可以使用 `__android_log_print` 等函数了。
* `set(CMAKE_CXX_STANDARD 17)` 设置了 C++ 编译标准为 C++17。

### 构建流程
动态库的构建流程本质上是 C++ 代码编译和链接的特定于 Android 平台的实现，主要依赖于 **Android NDK (Native Development Kit)**。NDK 是一套工具集，让你能够在 Android 平台上使用 C/C++ 等原生语言开发应用。它包含了下面这些组件：
* **Clang (LLVM):** 用于编译 C/C++ 代码的编译器。
* **Linker (ld):** 链接器，用于生成 `.so` 文件。
* **GCC (已逐渐被 Clang 替代):** 另一种编译器。
* **Standard C/C++ Libraries:** 如 `libc++` 或 `gnustl` (旧版本)。
* **Android Specific APIs:** 比如 `android/log.h`（用于日志输出）和 `jni.h`（用于 JNI 接口）。
* **构建系统：** 目前主流都是使用 **CMake** 。

我们使用NDK来构建C++代码的目标是生成一个`.so`文件，这个文件可以在Android平台上被加载和调用。这个过程又分为哪些阶段？
#### C++ 代码编译
C++ 是一种**编译型语言**，这意味着你的源代码在执行之前必须经过一个或多个阶段的转换。这个过程通常包括以下几个主要步骤。
##### **1. 预处理 (Preprocessing)**
**预处理器 (preprocessor)** 会处理源代码中以 `#` 开头的指令，这些指令被称为**预处理指令**。常见的预处理操作包括：
* **头文件包含 (`#include <file>` 或 `#include "file"`)：** 将指定文件的内容插入到当前文件中。这就像把多个代码片段拼接起来。
* **宏替换 (`#define`)：** 将宏定义替换为对应的文本。例如，`#define PI 3.14159` 会将代码中所有的 `PI` 替换为 `3.14159`。
* **条件编译 (`#ifdef`, `#ifndef`, `#if`, `#else`, `#endif`)：** 根据判断条件，选择性地编译或忽略代码块。这在针对不同平台或配置编译代码时非常有用。

预处理阶段的输出是一个**纯 C++ 源文件**，其中所有的预处理指令都已经被处理完毕，宏也已经展开，并且包含了所有引用的头文件内容。
##### **2. 编译 (Compilation)**
预处理后的源文件（通常以 `.i` 或 `.ii` 结尾，但在实际开发中你可能很少直接看到）会被**编译器 (compiler)** 处理。编译器的主要任务是将 C++ 源代码翻译成**汇编代码 (assembly code)**。

在这个阶段，编译器会执行以下工作：
* **词法分析 (Lexical Analysis)：** 将源代码分解成一系列的**词法单元 (tokens)**，如关键字、标识符、运算符、常量等。
* **语法分析 (Syntax Analysis)：** 根据 C++ 语法规则，将词法单元组织成一个**抽象语法树 (Abstract Syntax Tree - AST)**。如果代码存在语法错误，编译器会在这里报错。
* **语义分析 (Semantic Analysis)：** 检查代码的语义正确性，例如类型匹配、变量声明和初始化、函数调用是否正确等。
* **中间代码生成 (Intermediate Code Generation)：** 将 AST 转换为一种更接近机器语言但仍然独立于具体机器的**中间表示 (Intermediate Representation - IR)**，这有助于后续的优化。
* **代码优化 (Code Optimization)：** 对中间代码进行各种优化，以提高程序的执行效率、减少代码大小。这可能包括死代码消除、常量折叠、循环优化等。
* **目标代码生成 (Target Code Generation)：** 将优化后的中间代码转换成特定处理器架构的**汇编代码**。

编译阶段的输出是一个或多个**汇编文件**（通常以 `.s` 结尾）。
##### **3. 汇编 (Assembly)**
汇编器 (assembler) 的作用是将汇编代码翻译成**机器代码 (machine code)**，也就是由二进制指令组成的**目标文件 (object file)**。目标文件通常以 `.o` (Linux/macOS) 或 `.obj` (Windows) 结尾。

目标文件包含：
* 编译后的机器指令。
* 数据（全局变量、静态变量）。
* 符号表：记录了程序中定义和引用的函数、变量等符号的信息。
* 调试信息（如果开启了调试选项）。

此时的目标文件是独立的编译单元，它可能包含对其他文件或库中定义的函数和变量的引用（这些引用被称为**未解析的符号**）。
##### **4. 链接 (Linking)**
链接是整个编译过程的最后一个阶段，由**链接器 (linker)** 完成。链接器的主要任务是**将一个或多个目标文件以及程序所需要的库文件组合在一起**，生成一个可执行文件或一个共享库（如 `.so` 文件在 Android 上）。

链接器会完成以下工作：
* **符号解析 (Symbol Resolution)：** 解决目标文件中所有未解析的符号引用。例如，如果你的代码调用了 `printf` 函数，链接器会在标准库中找到 `printf` 的实现，并将这个引用解析到实际的函数地址。
* **重定位 (Relocation)：** 调整代码和数据段的地址。因为每个目标文件都是独立编译的，它们内部的地址都是相对地址。链接器会将它们合并到一个统一的地址空间中，并修正所有需要调整的地址。
* **库链接 (Library Linking)：**
    * **静态链接 (Static Linking)：** 将所有需要的库代码直接复制到可执行文件中。优点是可执行文件独立，不依赖外部库；缺点是文件较大，且库更新时需要重新编译链接。
    * **动态链接 (Dynamic Linking) / 共享链接 (Shared Linking)：** 可执行文件只包含对共享库（如 `.so` 文件）的引用，而不是实际的代码。在程序运行时，操作系统的**动态链接器/加载器**会加载这些共享库。优点是可执行文件较小，多个程序可以共享同一个库实例，节省内存，且库更新时无需重新编译程序；缺点是依赖外部库，如果库缺失或版本不兼容，程序可能无法运行。Android NDK 开发中通常使用动态链接生成 `.so` 文件。

链接阶段的输出是一个**可执行文件**（如 Linux 上的 `a.out` 或 Windows 上的 `.exe` 文件）或者一个**共享库**（如 Linux/Android 上的 `.so` 文件，Windows 上的 `.dll` 文件）。
#### Gradle 构建过程
在 Android Studio 中，我们通常不需要手动调用 CMake 或 ndk-build。`Android Gradle Plugin` 会为你自动化这些任务。例如我们配置了：

```gradle
android {
    // ...
    defaultConfig {
        // ...
        externalNativeBuild {
            cmake {
                // 指向 CMakeLists.txt 的路径
                    path file('src/main/cpp/CMakeLists.txt')
            }
            // 或者 ndkBuild { path file('src/main/cpp/Android.mk') }
        }
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a', 'x86' // 指定需要构建的 ABI
        }
    }
    // ...
}
```

当我们 Sync 项目时，Gradle 会检查 `externalNativeBuild` 配置。当你点击 "Build" 或 "Run" 时，Gradle 会执行下面这些操作：
* 根据 `build.gradle` 中配置的 ABI (`abiFilters`)，为每个目标架构调用相应的 NDK 工具链。
* **调用 CMake 或 ndk-build:** 你的 `CMakeLists.txt` 或 `Android.mk` 文件会被解析。
* **编译 (Compilation):** NDK 工具链中的 Clang 编译器将你的 C++ 源代码编译成对应架构的汇编代码，然后汇编成目标文件 (`.o` 文件)。
* **链接 (Linking):** NDK 工具链中的链接器将这些目标文件与 NDK 提供的系统库（如 `liblog.so`）和你的其他依赖库链接起来。最终，它会生成对应每个 ABI 的 **`.so` 动态库文件**（例如 `libnative-lib.so`）。
* **打包到 APK:** APK 文件本身是一个 ZIP 格式的归档文件。生成的 `.so` 文件会被 Gradle 打包到 APK 的特定目录下，通常是 `lib/<ABI>/`，例如 `lib/arm64-v8a/libnative-lib.so`。

## Java虚拟机如何运行的C++代码
JVM （或者说Android平台的Dalvik或者Art）本身并不能提供运行环境，直接 **运行** C++ 代码。相反，它通过一套**协调机制**来将执行权交给操作系统和底层的 CPU，由它们来执行 C++ 代码。这个协调机制其实就是 **JNI (Java Native Interface)**。
### JNI 的桥梁作用
JVM 并不理解 C++ 代码，它只运行字节码（Java 代码编译后的中间产物）。当你调用了一个 `native` 方法时，JVM 知道这个方法不是由 Java 实现的，而是由外部的本地代码提供。

JNI 规范了 Java 类型和 C++ 类型之间的映射（例如 `int` 对应 `jint`，`String` 对应 `jstring`），以及 Java 方法签名如何转换为 C++ 函数名。并且提供了一套 C 函数接口（通过 `JNIEnv*` 指针），允许 C++ 代码反过来操作 Java 对象、调用 Java 方法、访问 Java 字段等。

当你调用 `System.loadLibrary("mylib");` 时，发生的事情远不止简单地把文件读进内存：

#### **操作系统层面的加载** 
JVM 会请求操作系统（在 Android 上是 Linux 内核）的**动态链接器 (Dynamic Linker)** 来加载这个 `.so` 共享库文件。

动态链接器首先会在文件系统中找到 `libmylib.so`。然后将 `.so` 文件的内容 **内存映射 (Memory Map)** 到当前进程的虚拟地址空间。这并不是把整个文件一次性读入 RAM，而是建立一种映射关系，只有当程序真正访问到某个内存页时，才会从磁盘加载对应的页到物理内存。

映射完成后，执行解析符号。这是关键一步。`.so` 文件内部有一个**符号表 (Symbol Table)**，记录了它所包含的函数（如 `Java_com_example_MyClass_myNativeMethod`）和变量的名称及其在文件内的相对地址。同时，它也记录了自身所依赖的外部函数（如 `printf`、`__android_log_print` 等）的名称。动态链接器会查找这些外部依赖，在其他已经加载的系统库（如 `libc.so`、`liblog.so`）中找到它们的实际内存地址，并更新 `.so` 库内部的**重定位表 (Relocation Table)**，将对外部函数的引用替换为它们的实际地址。

最后，如果你的 `.so` 库中定义了 `JNI_OnLoad` 函数，动态链接器在加载并完成基本解析后，会**通知 JVM**。JVM 随后会调用这个 `JNI_OnLoad` 函数。你可以在这里执行动态注册，将 Java 方法直接映射到 C++ 函数的内存地址，或者进行其他初始化工作。
#### Java 方法与 Native 函数的绑定
在调用 `native` 方法之前，JVM 需要知道这个方法对应的 C++ 函数的具体入口点（也就是它在内存中的地址）。
* **静态注册：** 如果你没有使用 `JNI_OnLoad` 进行动态注册，那么当 JVM 第一次遇到一个 `native` 方法的调用时，它会：
    1.  根据 JNI 的命名规则（`Java_包名_类名_方法名`）构造一个字符串。
    2.  在已经加载的 `.so` 库的**符号表**中查找这个字符串对应的函数。
    3.  一旦找到，JVM 就会将这个 Java 方法与其对应的 C++ 函数的内存地址**绑定 (Binding)** 起来。这样，后续再次调用这个 Java 方法时，就可以直接跳转到那个 C++ 函数的地址，无需再次查找。
* **动态注册：** 如果你在 `JNI_OnLoad` 中使用了 `RegisterNatives`，那么绑定工作在库加载时就已经完成了。JVM 会直接获得 Native 函数的地址，效率更高，且不依赖于严格的命名约定。

### JNI 调用过程
当 JVM 首次尝试调用一个 `native` 方法时，它需要知道这个 Java 方法对应的 Native 函数在 `.so` 库中的具体地址。这种寻址过程就是根据上文提到的**符号表**来查找。

静态注册时，JVM 会根据 **JNI 的命名约定**来查找 Native 函数。直接在已加载的 `.so` 库的符号表中查找符合命名规范的函数。这个查找过程在第一次调用该 Native 方法时发生，并将 Java 方法与 Native 函数的地址进行**绑定 (Binding)**。后续再次调用时，就可以直接跳转到 Native 函数的地址，提高了效率。如果找不到对应的函数，会抛出 `UnsatisfiedLinkError`。

动态注册是通过 `JNI_OnLoad` 函数，具体的是 `RegisterNatives` JNI 函数手动将 Java 方法和 Native 函数进行关联。这种方式可以不遵循 JNI 的严格命名约定，Native 函数名可以更简洁。可以批量注册多个方法更高效。还有助于防止函数名过长导致某些旧系统（如 Windows）上的问题。

#### 参数传递与栈帧
当 Java 代码调用 Native 方法时：
1.  **保存 Java 运行上下文：** JVM 第一步会保存当前 Java 方法的执行上下文（如局部变量、操作数栈状态等）。
2.  **JNIEnv 指针：** JVM 会将一个 `JNIEnv*` 指针作为第一个参数传递给 Native 函数。这个指针提供了访问 JVM 功能的接口。
3.  **JObject/JClass 参数：**
    * 对于非静态 Native 方法，`jobject` 参数代表调用该方法的 Java 对象的引用。
    * 对于静态 Native 方法，`jclass` 参数代表调用该方法的 Java 类的引用。
4.  **参数转换：** Java 基本类型（如 `int`, `boolean`）会直接映射到对应的 JNI 类型（`jint`, `jboolean`）。对于 Java 对象类型（如 `String`, `Object`），JNI 会传递一个 `jobject` 或其子类型的引用。这些引用是指向 JVM 内部 Java 对象的指针。
5.  **切换栈帧：** JVM 的执行流会从 Java 栈帧切换到 Native 栈帧。CPU 会开始执行 Native `.so` 文件中的机器码。

#### Native 代码执行和返回
JVM 将执行流程**跳转**到 Native C++ 函数在内存中的起始地址。此时，CPU 开始执行 `.so` 库中的 C++ 机器码。**JVM 不再直接“运行”C++ 代码，而是将控制权交给了 CPU，由 CPU 直接执行预编译好的机器指令。**

Native C++ 代码在运行时，可以直接使用 JNIEnv 指针来与 JVM 进行交互。例如：

* `env->NewStringUTF()`: 从 C 字符串创建 Java `String` 对象。
* `env->GetStringUTFChars()`: 将 Java `String` 转换为 C 字符串。
* `env->CallIntMethod()`: 调用 Java 对象的某个 int 类型返回值的成员方法。
* `env->ThrowNew()`: 在 Java 层抛出异常。

当 Native 函数执行完毕并通过 `return` 语句返回结果时。Native 函数返回的 **C/C++ 类型结果会被 JNI 自动转换回对应的 Java 类型**。

然后要恢复 Java 上下文，执行流会**从 Native 栈帧切换回 Java 栈帧**。如果 Native 代码中设置了待抛出的 Java 异常，JVM 会在返回后立即抛出该异常。如果无异常，Java 调用方会接收到 Native 方法的返回值，并继续其后续操作。
### 内存管理与垃圾回收
一个重要的底层细节是内存管理：
* **Java 堆：** Java 对象存储在 Java 堆上，由 JVM 的垃圾回收器管理。
* **Native 堆：** C/C++ 代码可以通过 `malloc/free` 或 `new/delete` 在 Native 堆上分配内存。这部分内存不受 JVM 垃圾回收器的管理，需要 Native 代码自己负责释放，否则会导致内存泄漏。
* **JNI 引用：** 当 Native 代码获取到 Java 对象的引用时（`jobject`, `jstring`, `jbyteArray` 等），这些引用被称为 **局部引用 (Local Reference)**。它们在 Native 方法返回后会自动被 JVM 释放。如果你需要在 Native 方法返回后继续持有这些引用，你需要将它们提升为 **全局引用 (Global Reference)**，并手动管理其生命周期。
