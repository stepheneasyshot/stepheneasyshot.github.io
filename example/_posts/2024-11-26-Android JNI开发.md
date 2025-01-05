---
layout: post
description: > 
  介绍了JNI开发的一般流程，以及基础性的知识储备。
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
# Android JNI开发
## 官方文档
[Google官方JNI文档](https://developer.android.com/ndk/guides?hl=zh-cn)

## 项目例程
[\[JniDemo\]](https://github.com/stepheneasyshot/JniDemo)

## 基本开发流程
Android Studio 编译原生库的默认构建工具是 CMake。由于很多现有项目都使用 ndk-build 构建工具包，因此 Android Studio 也支持 ndk-build。不过，如果您要创建新的原生库，则应使用 CMake。新的接口开发全部使用cmake来构建，相比之前的ndk-build的配置方式，使用cmake可以省略掉.h文件声明和android.mk文件来辅助构建，只需要一个CMakeList.txt即可。

开发流程：
1. Java/Kotlin代码里创建好需要的native方法，注意在Cpp文件中对方法名有明确要求。

```kotlin
package com.stephen.jnitest

object JniUtils {

   fun init() {
        System.loadLibrary("jni-test")
    }

    external fun hello(): String
}
```

2. 创建Native代码文件，即C/C++文件

```c++
#include <jni.h>
#include <string>
#include <android/log.h>

#define LOG_TAG "Stephen JNI TEST"

extern "C" JNIEXPORT jstring JNICALL
Java_com_stephen_jnitest_JniUtils_hello(
        JNIEnv *env, jobject) {
    const char *hello = "Hello from C++";

    __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG,
                        "This is my first time using android log in C++");
    __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, "Hello String: [%s]", hello);

    return env->NewStringUTF(hello);
}
```

3. 创建CmakeLists.txt脚本文件

```
cmake_minimum_required(VERSION 3.18.1)

project("jni-test")

add_library(jni-test SHARED
        jni-test.cpp)

# Include libraries needed for lib
target_link_libraries(jni-test
        android
        log)
```

4. 在gradle里配置构建脚本的路径

```groovy
android{
    externalNativeBuild {
        cmake {
            path = file("src/main/cpp/CMakeLists.txt")
        }
    }
}
```
文件结构如下：

![pic](/assets/img/blog/blogs_jni_tree.png){:width="400" height="300" loading="lazy"}

以上是在一个android library里进行的开发，完成后可以打包aar对外提供。
## CMakeList写法
Google原生的提示模板：
```
# Sets the minimum version of CMake required to build your native library.
# This ensures that a certain set of CMake features is available to
# your build.

cmake_minimum_required(VERSION 3.4.1)

# Specifies a library name, specifies whether the library is STATIC or
# SHARED, and provides relative paths to the source code. You can
# define multiple libraries by adding multiple add_library() commands,
# and CMake builds them for you. When you build your app, Gradle
# automatically packages shared libraries with your APK.

add_library( # Specifies the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )
```
第一addLibrary需要制定库的名称，第二可以选择配置为静态库还是动态库方式，第三是源文件。
### 添加原生依赖库
向 CMake 构建脚本添加 find_library() 命令以找到 NDK 库并将其路径存储为一个变量。您可以使用此变量在构建脚本的其他部分引用 NDK 库。
比如引用Android原生的日志库：
```
find_library( # Defines the name of the path variable that stores the
              # location of the NDK library.
              log-lib

              # Specifies the name of the NDK library that
              # CMake needs to locate.
              log )

# Links your native library against one or more other native libraries.
target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the log library to the target library.
                       ${log-lib} )
```
也可以使用add_library()，直接添加原生代码当作依赖，以下命令可以指示 CMake 将 android_native_app_glue.c（负责管理 NativeActivity 生命周期事件和触控输入）构建至静态库，并将其与 native-lib 关联：
```
add_library( app-glue
             STATIC
             ${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c )

# You need to link static libraries against your shared native library.
target_link_libraries( native-lib app-glue ${log-lib} )
```
### 添加头文件
在Android Studio中使用CMake添加头文件，你需要在CMakeLists.txt文件中使用include_directories指令。这个指令告诉CMake在编译时需要包含哪些目录来搜索头文件。
例如，如果你有一个头文件目录位于app/src/main/cpp/include，你可以在CMakeLists.txt中添加如下指令：
```
include_directories(include)
```
这行代码会告诉CMake在编译时需要包含app/src/main/cpp/include目录下的所有头文件。
目录结构如下：
完整的CMakeLists.txt示例如下：
```
cmake_minimum_required(VERSION 3.18.1)

project("terminal-channel")

add_library(terminal-channel SHARED
        common.cpp
        process.cpp
        termExec.cpp)

include_directories(include)

target_link_libraries(terminal-channel
        android
        log)添加预构建库
add_library( imported-lib
             SHARED
             IMPORTED )然后，您需要使用 set_target_properties() 命令指定库的路径：
add_library(...)
set_target_properties( # Specifies the target library.
                       imported-lib

                       # Specifies the parameter you want to define.
                       PROPERTIES IMPORTED_LOCATION

                       # Provides the path to the library you want to import.
                       imported-lib/src/${ANDROID_ABI}/libimported-lib.so )
```
## Android ABI
不同的 Android 设备使用不同的 CPU，而不同的 CPU 支持不同的指令集。CPU 与指令集的每种组合都有专属的应用二进制接口 (ABI)。ABI 包含以下信息：

* 可使用的 CPU 指令集（和扩展指令集）。
* 运行时内存存储和加载的字节顺序。Android 始终是 little-endian。
* 在应用和系统之间传递数据的规范（包括对齐限制），以及系统调用函数时如何使用堆栈和寄存器。
* 可执行二进制文件（例如程序和共享库）的格式，以及它们支持的内容类型。Android 始终使用 ELF。如需了解详情，请参阅 ELF System V 应用二进制接口。
* 如何重整 C++ 名称。如需了解详情，请参阅 Generic/Itanium C++ ABI。

**armeabi-v7a**

此 ABI 适用于 32 位 ARM CPU。它包括 Thumb-2 和 Neon。

**arm64-v8a**

此 ABI 适用于 64 位 ARM CPU。

**x86**

此 ABI 适用于支持通常称为“x86”“i386”或“IA-32”的指令集的 CPU。

**x86_64**

此 ABI 适用于支持通常称为“x86-64”的指令集的 CPU。

### gradle配置
默认情况下，Gradle（无论是通过 Android Studio 使用，还是从命令行使用）会针对所有非弃用 ABI 进行构建。要限制应用支持的 ABI 集，请使用 abiFilters。例如，要仅针对 64 位 ABI 进行构建，请在 build.gradle 中设置以下配置：

```
android {
    defaultConfig {
        ndk {
            abiFilters 'arm64-v8a', 'x86_64'
        }
    }
}
```
### Native方法声明解析
以之前的ndk-build的方式声明的头文件为例：

```c++
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class HelloJNI */

#ifndef _Included_HelloJNI
#define _Included_HelloJNI
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     HelloJNI
 * Method:    sayHello
 * Signature: ()V
 */
JNIEXPORT jstring JNICALL Java_HelloJNI_sayHello(JNIEnv *, jobject);

#ifdef __cplusplus
}
#endif
#endif
```

* extern "C" 告诉 C++ 编译器以 C 的方式来编译这个函数，以方便其他 C 程序链接和访问该函数。C 和 C++ 有着不同的命名协议，因为 C++ 支持函数重载，用了不同的命名协议来处理重载的函数。在 C 中函数是通过函数名来识别的，而在 C++ 中，由于存在函数的重载问题，函数的识别方式通过函数名，函数的返回类型，函数参数列表三者组合来完成的。因此两个相同的函数，经过C，C++编绎后会产生完全不同的名字。所以，如果把一个用 C 编绎器编绎的目标代码和一个用 C++ 编绎器编绎的目标代码进行链接，就会出现链接失败的错误。
* JNIEnv：JNIEnv 内部提供了很多函数，方便我们进行 JNI 编程。
* jobject: 指向 "this" 的 Java 对象
* 如果 java 中的 native 函数是 static 的，那第二个参数是 jclass，代表了 java 中的 Class 类。
* JNIEXPORT、JNICALL 两个宏在 linux 平台的定义如下：

```
//该声明的作用是保证在本动态库中声明的方法 , 能够在其他项目中可以被调用
#define JNIEXPORT  __attribute__ ((visibility ("default")))
//一个空定义
#define JNICALL
```
## JNI_ONLOAD
### 原生库
您可以使用标准 API 从共享库加载原生代码 System.loadLibrary。
事实上，旧版 Android 的 PackageManager 中存在导致安装和 使原生库更新不可靠。ReLinker 项目提供了解决此问题和其他原生库加载问题的解决方法。
从静态类调用 System.loadLibrary（或 ReLinker.loadLibrary） 初始化函数。参数是“未修饰”库名称 因此，要加载 libfubar.so，您需要传入 "fubar"。
如果您只有一个类具有原生方法，则调用 System.loadLibrary 位于该类的静态初始化程序中。否则，您应该从 Application 进行该调用，这样您就知道始终会加载该库，并且始终会提前加载。运行时可以通过两种方式找到您的原生方法。您可以请使用 RegisterNatives 注册它们；也可以让运行时动态查询它们和dlsym。

**RegisterNatives 的优势在于，您可以提前还可以检查这些符号是否存在导出除 JNI_OnLoad 之外的任何内容。这样做的好处是让运行时因为它需要编写的代码略少一些。**

如需使用 RegisterNatives，请执行以下操作：

* 提供 JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) 函数。
* 在 JNI_OnLoad 中，使用 RegisterNatives 注册所有原生方法。
* 使用 -fvisibility=hidden 进行构建，以便仅使用您的 

JNI_OnLoad 。这样可以生成更快、更小的代码，并避免 与加载到您的应用中的其他库发生冲突（但创建的堆栈轨迹没有多大用处） （如果您的应用在原生代码中崩溃）。
### JNI_OnLoad方法
Java JNI有两种方法，一种是通过javah,获取一组带签名函数，然后实现这些函数。这种方法很常用，也是官方推荐的方法。还有一种就是JNI_OnLoad方法。
当Android的VM(Virtual Machine)执行到C组件(即*so档)里的System.loadLibrary()函数时，首先会去执行C组件里的JNI_OnLoad()函数。它的用途有二： 

* 告诉VM此C组件使用那一个JNI版本。如果你的*.so档没有提供JNI_OnLoad()函数，VM会默认该*.so档是使用最老的JNI 1.1版本。由于新版的JNI做了许多扩充，如果需要使用JNI的新版功能，例如JNI 1.4的java.nio.ByteBuffer,就必须藉由JNI_OnLoad()函数来告知VM。
* 由于VM执行到System.loadLibrary()函数时，就会立即先呼叫JNI_OnLoad()，所以C组件的开发者可以藉由JNI_OnLoad()来进行C组件内的初期值之设定(Initialization) 。

JNI_OnLoad方法内容基本固定：

```c++
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    // Find your class. JNI_OnLoad is called from the correct class loader context for this to work.
    jclass c = env->FindClass("com/example/app/package/MyClass");
    if (c == nullptr) return JNI_ERR;

    // Register your class' native methods.
    static const JNINativeMethod methods[] = {
        {"nativeFoo", "()V", reinterpret_cast<void*>(nativeFoo)},
        {"nativeBar", "(Ljava/lang/String;I)Z", reinterpret_cast<void*>(nativeBar)},
    };
    int rc = env->RegisterNatives(c, methods, sizeof(methods)/sizeof(JNINativeMethod));
    if (rc != JNI_OK) return rc;

    return JNI_VERSION_1_6;
}
```
## 数据类型
### 基础数据类型

| Java 类型     | JNI 类型   | C/C++ 类型 |
|:---------|:----------|:----------|
| boolean  | jboolean | unsigned char |
| byte      |  jbyte | signed char |
| char  | jchar |unsigned short |
|short|jshort|signed short|
|int    | jint |  int |
| long|jlong|long|
|float|jfloat|float|
|double|jdouble|double|

以上基础类型可以随意互相转换，直接使用.
Kotlin：
```kotlin
    external fun add(a: Int, b: Int): Int

    external fun calChar(charater: Char): Char
```
C++：
```c++
extern "C" JNIEXPORT jint JNICALL
Java_com_stephen_jnitest_JniUtils_add(JNIEnv *env, jobject, jint a, jint b) {
    return a + b;
}

extern "C" JNIEXPORT jchar JNICALL
Java_com_stephen_jnitest_JniUtils_calChar(JNIEnv *env, jobject, jchar a) {
    return a + 1;
}
```
### 引用类型
jni.h 中定义的非基本数据类型称为引用类型。

| Java 类型|JNI 引用类型 | 类型描述|
|:---------|:----------|:----------|
|java.lang.Object|jobject|表示任何Java的对象|
|java.lang.String|jstring|Java的String字符串类型的对象|
|java.lang.Class|jclass|Java的Class类型对象|
|java.lang.Throwable|jthrowable|Java的Throwable类型|
|byte[]|jbyteArray|Java byte型数组|
|Object[]|jobjectArray|Java任何对象的数组|
|boolean[]|jbooleanArray|Java boolean型数组|
|char[]|jcharArray|Java char型数组|
|short[]|jshortArray|Java short型数组|
|int[]|jintArray|Java int型数组|
|long[]|jlongArray|Java long型数组|
|float[]|jfloatArray|Java float型数组|
|double[]|jdoubleArray|Java double型数组|

这些数据类型在使用时需要互相转换。
native 程序主要做了这么几件事：

1. 接收 JNI 类型的参数
2. 参数类型转换，JNI 类型转换为 Native 类型
3. 执行 Native 代码
4. 创建一个 JNI 类型的返回对象，将结果拷贝到这个对象并返回结果

#### 字符串
为了在 C/C++ 中使用 Java 字符串，需要先将 Java 字符串转换成 C 字符串。用 GetStringChars 函数可以将 Unicode 格式的 Java 字符串转换成 C 字符串，用 GetStringUTFChars 函数可以将 UTF-8 格式的 Java 字符串转换成 C 字符串。这些函数的第三个参数均为 isCopy，它让调用者确定返回的 C 字符串地址指向副本还是指向堆中的固定对象。

```c++
JNIEXPORT jstring JNICALL Java_HelloJNI_sayHello__Ljava_lang_String_2(JNIEnv *env, jobject jobj, jstring str) {
  
    //jstring -> char*
    jboolean isCopy;
    //GetStringChars 用于 unicode 编码
    //GetStringUTFChars 用于 utf-8 编码
    const char* cStr = env->GetStringUTFChars(str, &isCopy);
  
    if (nullptr == cStr) {
        return nullptr;
    }

    if (JNI_TRUE == isCopy) {
        cout << "C 字符串是 java 字符串的一份拷贝" << endl;
    } else {
        cout << "C 字符串指向 java 层的字符串" << endl;
    }

    cout << "C/C++ 层接收到的字符串是 " << cStr << endl;
  
    //通过JNI GetStringChars 函数和 GetStringUTFChars 函数获得的C字符串在原生代码中
    //使用完之后需要正确地释放，否则将会引起内存泄露。
    env->ReleaseStringUTFChars(str, cStr);

    string outString = "Hello, JNI";
    // char* 转换为 jstring
    return env->NewStringUTF(outString.c_str());
}
```
#### 字符串的其他常用操作函数

**GetStringUTFChars/ReleaseStringUTFChars**

Java 默认使用 UTF-16 编码，而 C/C++ 默认使用 UTF-8 编码。GetStringUTFChars 可以把一个 jstring 指针（指向 JVM 内部的 UTF-16 字符序列）转换成一个 UTF-8 编码的 C 风格字符串。
```
// 参数说明：
// * this: JNIEnv 指针
// * string: jstring类型(Java 传递给本地代码的字符串指针)
// * isCopy: 它的取值可以是 JNI_TRUE (值为1)或者为 JNI_FALSE (值为0)。如果值为 JNI_TRUE，表示返回 JVM 内部源字符串的一份拷贝，并为新产生的字符串分配内存空间。如果值为 JNI_FALSE，表示返回 JVM 内部源字符串的指针，意味着可以通过指针修改源字符串的内容，不推荐这么做，因为这样做就打破了 Java 字符串不能修改的规定。但我们在开发当中，并不关心这个值是多少，通常情况下这个参数填 NULL 即可。
const char* (*GetStringUTFChars)(JNIEnv*, jstring, jboolean*);//C环境中的定义
    
const char* GetStringUTFChars(jstring string, jboolean* isCopy)//C++环境中的定义
{ return functions->GetStringUTFChars(this, string, isCopy); }
```
调用完 GetStringUTFChars 之后不要忘记安全检查，因为 JVM 可能需要为新诞生的字符串分配内存空间，当内存空间不够分配的时候，会导致调用失败，失败后 GetStringUTFChars 会返回 NULL，并抛出一个 OutOfMemoryError 异常。JNI的异常和 Java 中的异常处理流程是不一样的，Java 遇到异常如果没有捕获，程序会立即停止运行。而 JNI 遇到未决的异常不会改变程序的运行流程，也就是程序会继续往下走，这样后面针对这个字符串的所有操作都是非常危险的，因此，我们需要用 return 语句跳过后面的代码，并立即结束当前方法。
```
// 参数说明：
// this: JNIEnv 指针
// string: 指向一个 jstring 变量，即是要释放的本地字符串的来源。在当前环境下指向 Java 中传递过来的 String 字符串对应的 JNI 数据类型 jstring
// utf：将要释放的C/C++本地字符串。即我们调用GetStringUTFChars获取的数据的存储指针。
void (*ReleaseStringUTFChars)(JNIEnv*, jstring, const char*);//C中的定义
    
void ReleaseStringUTFChars(jstring string, const char* utf)//C++中的定义
{ functions->ReleaseStringUTFChars(this, string, utf); }
```
ReleaseStringUTFChars 函数用于通知虚拟机 jstring 在 jvm 中对应的内存已经不使用了，可以清除了。

**GetStringChars/ReleaseStringChars**

GetStringChars返回字符串 string 对应的 UTF-16 字符数组的指针。在内存不足时抛出 OutOfMemoryError 异常.
ReleaseStringChars通知虚拟机平台释放 chars 所引用的相关资源，以免造成内存泄漏。参数chars 是一个指针，可通过 GetStringChars() 从 string 获得。
```
const jchar* (GetStringChars)(JNIEnv env, jstring string, jboolean* isCopy);

void ReleaseStringChars (JNIEnv *env, jstring string, const jchar *chars);
```

**NewStringUTF**

利用C风格字符串创建一个新的 java.lang.String 字符串对象。这个新创建的字符串会自动转换成 Java 支持的 UTF-16 编码。在内存不足时抛出 OutOfMemoryError 异常。
```
// 参数说明
// this: JNIEnv 指针
// bytes: 指向一个char * 变量，即要返回给 Java 层的 C/C++ 中字符串。
jstring  (*NewStringUTF)(JNIEnv*, const char*);//C环境中定义

jstring NewStringUTF(const char* bytes)//C++环境中的定义
{ return functions->NewStringUTF(this, bytes); }
```

**NewString**

利用 UTF-16 字符数组构造新的 java.lang.String 对象。在内存不足时抛出 OutOfMemoryError 异常。
```
jstring (NewString)(JNIEnv env, const jchar* unicodeChars, jsize size);
```

**GetStringUTFLength**

返回字符串的 UTF-8 编码的长度，即 C 风格字符串的长度。
```
jsize (GetStringUTFLength)(JNIEnv env, jstring string);
```

**GetStringLength**

返回字符串的 UTF-16 编码的长度，即 Java 字符串长度
```
const jchar* (GetStringChars)(JNIEnv env, jstring string, jboolean* isCopy);
```

**GetStringCritical/ReleaseStringCritical**

此前提到的Get/ReleaseStringChars 和 Get/ReleaseStringUTFChars 这对函数返回的源字符串会后分配内存，如果有一个字符串内容相当大，有 1M 左右，而且只需要读取里面的内容打印出来，用这两对函数就有些不太合适了。
此时用 Get/ReleaseStringCritical 可直接返回源字符串的指针应该是一个比较合适的方式。不过这对函数有一个很大的限制，在这两个函数之间的本地代码不能调用任何会让线程阻塞或等待 JVM 中其它线程的本地函数或 JNI 函数。因为通过 GetStringCritical 得到的是一个指向 JVM 内部字符串的直接指针，获取这个直接指针后会导致暂停 GC 线程，当 GC 被暂停后，如果其它线程触发 GC 继续运行的话，都会导致阻塞调用者。所以在Get/ReleaseStringCritical 这对函数中间的任何本地代码都不可以执行导致阻塞的调用或为新对象在 JVM 中分配内存，否则，JVM 有可能死锁。另外一定要记住检查是否因为内存溢出而导致它的返回值为 NULL，因为 JVM 在执行 GetStringCritical 这个函数时，仍有发生数据复制的可能性，尤其是当 JVM 内部存储的数组不连续时，为了返回一个指向连续内存空间的指针，JVM 必须复制所有数据。
与 GetStringUTFChars 相同，GetStringCritical 也可能在内存不足时抛出 OutOfMemoryError 异常。

**GetStringRegion/GetStringUTFRegion**

分别表示获取 UTF-16 和 UTF-8 编码字符串指定范围内的内容。
这对函数会把源字符串复制到一个预先分配的缓冲区内。

```c++
JNIEXPORT jstring JNICALL Java_HelloJNI_sayHello__Ljava_lang_String_2(JNIEnv *env, jobject jobj, jstring str) {
    char buff[128];
    jsize len = env->GetStringUTFLength(str); // 获取 utf-8 字符串的长度
    // 将虚拟机平台中的字符串以 utf-8 编码拷入C缓冲区,该函数内部不会分配内存空间
    env->GetStringUTFRegion(str,0,len,buff);
}
```

#### 小结
* 对于小字符串来说，GetStringRegion 和 GetStringUTFRegion 这两对函数是最佳选择，因为缓冲区可以被编译器提前分配，而且永远不会产生内存溢出的异常。当你需要处理一个字符串的一部分时，使用这对函数也是不错。因为它们提供了一个开始索引和子字符串的长度值。另外，复制少量字符串的消耗也是非常小的。
* 使用 GetStringCritical 和 ReleaseStringCritical 这对函数时，必须非常小心。一定要确保在持有一个由 GetStringCritical 获取到的指针时，本地代码不会在 JVM 内部分配新对象，或者做任何其它可能导致系统死锁的阻塞性调用。
* 获取 Unicode 字符串和长度，使用 GetStringChars 和 GetStringLength 函数。获取 UTF-8 字符串的长度，使用 GetStringUTFLength 函数。
* 创建 Unicode 字符串，使用NewString，创建UTF-8使用 NewStringUTF 函数。
* 通过 GetStringUTFChars、GetStringChars、GetStringCritical 获取字符串，这些函数内部会分配内存，必须调用相对应的 ReleaseXXXX 函数释放内存。

##### 数组
```c++
JNIEXPORT jdoubleArray JNICALL Java_HelloJNI_sumAndAverage(JNIEnv *env, jobject obj, jintArray inJNIArray) {
    //类型转换 jintArray -> jint*
    jboolean isCopy;
    jint* inArray = env->GetIntArrayElements(inJNIArray, &isCopy);

    if (JNI_TRUE == isCopy) {
        cout << "C 层的数组是 java 层数组的一份拷贝" << endl;
    } else {
        cout << "C 层的数组指向 java 层的数组" << endl;
    }

    if(nullptr == inArray) return nullptr;
    //获取到数组长度
    jsize length = env->GetArrayLength(inJNIArray);

    jint sum = 0;
    for(int i = 0; i < length; ++i) {
        sum += inArray[i];
    }

    jdouble average = (jdouble)sum / length;
    //释放数组
    env->ReleaseIntArrayElements(inJNIArray, inArray, 0); // release resource

    //构造返回数据，outArray 是指针类型，需要 free 或者 delete 吗？要的
    jdouble outArray[] = {sum, average};
    jdoubleArray outJNIArray = env->NewDoubleArray(2);
    if(NULL == outJNIArray) return NULL;
    //向 jdoubleArray 写入数据
    env->SetDoubleArrayRegion(outJNIArray, 0, 2, outArray);
    return outJNIArray;
}
```
使用时需要特别注意item对象的创建与释放。
JNI 中的数组分为基本类型数组和对象数组，它们的处理方式是不一样的，基本类型数组中的所有元素都是 JNI的基本数据类型，可以直接访问。而对象数组中的所有元素是一个类的实例或其它数组的引用，和字符串操作一样，不能直接访问 Java 传递给 JNI 层的数组，必须选择合适的 JNI 函数来访问和设置 Java 层的数组对象。
##### 引用数组

**一维数组**

```c++
JNIEXPORT jobjectArray JNICALL Java_com_xxx_jni_JNIArrayManager_operateStringArrray
  (JNIEnv * env, jobject object, jobjectArray objectArray_in)
{
    //获取到长度信息
    jsize  size = env->GetArrayLength(objectArray_in);

    /*******获取从JNI传过来的String数组数据**********/


    for(int i = 0; i < size; i++)
    {
        jstring string_in= (jstring)env->GetObjectArrayElement(objectArray_in, i);
        char *char_in  = env->GetStringUTFChars(str, nullptr);
    }


    /***********从JNI返回String数组给Java层**************/
    jclass clazz = env->FindClass("java/lang/String");
    jobjectArray objectArray_out;
    const int len_out = 5;
    objectArray_out = env->NewObjectArray(len_out, clazz, NULL);
    char * char_out[]=  { "Hello,", "world!", "JNI", "is", "fun" };

    jstring temp_string;
    for( int i= 0; i < len_out; i++ )
    {   
        temp_string = env->NewStringUTF(char_out[i])；
        env->SetObjectArrayElement(objectArray_out, i, temp_string);
    }
    return objectArray_out;
}
```

**二维数组**

```c++
JNIEXPORT jobjectArray JNICALL Java_com_xxx_jni_JNIArrayManager_operateTwoIntDimArray(JNIEnv * env, jobject object, jobjectArray objectArray_in)
{
    /**********    解析从Java得到的int型二维数组 **********/
    int i, j ;
    const int row = env->GetArrayLength(objectArray_in);//获取二维数组的行数
    jarray array = (jarray)env->GetObjectArrayElement(objectArray_in, 0);
    const int col = env->GetArrayLength(array);//获取二维数组每行的列数

    //根据行数和列数创建int型二维数组
    jint intDimArrayIn[row][col];

    
    for(i =0; i < row; i++)
    {
         array = (jintArray)env->GetObjectArrayElement(objectArray_in, i);
        
         //操作方式一，这种方法会申请natvie memory内存
         jint *coldata = env->GetIntArrayElements((jintArray)array, NULL );        
         for (j=0; j<col; j++) {    
              intDimArrayIn [i] [j] = coldata[j]; //取出JAVA类中int二维数组的数据,并赋值给JNI中的数组  
         }  

          //操作方式二，赋值,这种方法不会申请内存
          //  env->GetIntArrayRegion((jintArray)array, 0, col, (jint*)&intDimArrayIn[i]);         
          
         env->ReleaseIntArrayElements((jintArray)array, coldata,0 );  
    }

    /**************创建一个int型二维数组返回给Java**************/
    const int row_out = 2;//行数
    const int col_out = 2;//列数

    //获取数组的class
    jclass clazz  = env->FindClass("[I");//一维数组的类
    //新建object数组，里面是int[]
    jobjectArray intDimArrayOut = env->NewObjectArray(row_out, clazz, NULL);

    int tmp_array[row_out][col_out] = { { 0,1 }, { 2,3 } };
    for(i = 0; i< row_out; i ++)
    {
        jintArray intArray = env->NewIntArray(col_out);
        env->SetIntArrayRegion(intArray, 0, col_out, (jint*)&tmp_array[i]);
        env->SetObjectArrayElement(intDimArrayOut, i, intArray);
    }
    return intDimArrayOut;
}
```

**GetArrayLength**

```c++
jsize (GetArrayLength)(JNIEnv env, jarray array);
```
返回数组中的元素个数

**NewObjectArray**

```
jobjectArray NewObjectArray (JNIEnv *env, jsize length, jclass elementClass, jobject initialElement);
```
构建 JNI 引用类型的数组，它将保存类 elementClass 中的对象。所有元素初始值均设为 initialElement，一般使用 NULL 就好。如果系统内存不足，则抛出 OutOfMemoryError 异常。

**GetObjectArrayElement和SetObjectArrayElement**

```
jobject GetObjectArrayElement (JNIEnv *env, jobjectArray array, jsize index)
```
返回 jobjectArray 数组的元素，通常是获取 JNI 引用类型数组元素。如果 index 不是数组中的有效下标，则抛出ArrayIndexOutOfBoundsException 异常。
```
void SetObjectArrayElement (JNIEnv *env, jobjectArray array, jsize index, jobject value)
```
设置 jobjectArray 数组中 index 下标对象的值。如果 index 不是数组中的有效下标，则会抛出 ArrayIndexOutOfBoundsException 异常。如果 value 的类不是数组元素类的子类，则抛出 ArrayStoreException 异常。

**New\<PrimitiveType\>Array 函数集**

```
NativeTypeArray New<PrimitiveType>Array (JNIEnv* env, jsize size)
```
用于构造 JNI 基本类型数组对象。
在实际应用中把 PrimitiveType 替换为某个实际的基本类型数据类型，然后再将 NativeType 替换成对应的 JNI Native Type 即可，具体的：
```
函数名                      返回类型
NewBooleanArray()           jbooleanArray
NewByteArray()              jbyteArray
NewCharArray()              jcharArray
NewShortArray()             jshorArray
NewIntArray()               jintArray
NewLongArray()              jlongArray
NewFloatArray()             jfloatArray
NewDoubleArray()            jdoubleArray      
```

**Get/ReleaseArrayElements函数集**

```
NativeType* Get<PrimitiveType>ArrayElements(JNIEnv *env, NativeTypeArray array, jboolean *isCopy)
```
该函数用于将 JNI 数组类型转换为 JNI 基本数据类型数组，在实际使用过程中将 PrimitiveType 替换成某个实际的基本类型元素访问函数，然后再将NativeType替换成对应的 JNI Native Type 即可：
```
函数名                           转换前类型             转换后类型
GetBooleanArrayElements()       jbooleanArray          jboolean*
GetByteArrayElements()          jbyteArray             jbyte*
GetCharArrayElements()          jcharArray             jchar*
GetShortArrayElements()         jshortArray            jshort*
GetIntArrayElements()           jintArray              jint*
GetLongArrayElements()          jlongArray             jlong*
GetFloatArrayElements()         jfloatArray            jfloat*
GetDoubleArrayElements()        jdoubleArray           jdouble*
```
```
void Release<PrimitiveType>ArrayElements (JNIEnv *env, NativeTypeArray array, NativeType *elems,jint mode);
```
该函数用于通知 JVM，数组不再使用，可以清理先关内存了。在实际使用过程中将 PrimitiveType 替换成某个实际的基本类型元素访问函数，然后再将 NativeType 替换成对应的 JNI Native Type 即可：
```
函数名                              NativeTypeArray        NativeType
ReleaseBooleanArrayElements()       jbooleanArray          jboolean
ReleaseByteArrayElements()          jbyteArray             jbyte
ReleaseCharArrayElements()          jcharArray             jchar
ReleaseShortArrayElements()         jshortArray            jshort
ReleaseIntArrayElements()           jintArray              jint
ReleaseLongArrayElements()          jlongArray             jlong
ReleaseFloatArrayElements()         jfloatArray            jfloat
ReleaseDoubleArrayElements()        jdoubleArray  
```

**jdoubleGet/Set\<PrimitiveType\>ArrayRegion**

```
void Set<PrimitiveType>ArrayRegion (JNIEnv *env, NativeTypeArray array, jsize start, jsize len, NativeType *buf);
```
该函数用于将基本类型数组某一区域复制到 JNI 数组类型中。在实际使用过程中将 PrimitiveType 替换成某个实际的基本类型元素访问函数，然后再将 NativeType 替换成对应的 JNI Native Type 即可：

```
函数名                              NativeTypeArray        NativeType
SetBooleanArrayRegion()             jbooleanArray          jboolean
SetByteArrayRegion()                jbyteArray             jbyte
SetCharArrayRegion()                jcharArray             jchar
SetShortArrayRegion()               jshortArray            jshort
SetIntArrayRegion()                 jintArray              jint
SetLongArrayRegion()                jlongArray             jlong
SetFloatArrayRegion()               jfloatArray            jfloat
SetDoubleArrayRegion()              jdoubleArray           jdouble
```
## 防止Native内存泄漏
JNI 层作为 Java 层和 Native 层之间相交互的中间层，它兼具 Native 层和 Java 层的某些特性，尤其在对引用对象的创建和回收上。
* 和 C++ 里的 new 操作符可以创建一个对象类似，JNI 层可以利用 JNI NewObject 等函数创建一个 Java 意义的对象(引用型对象）。这个被 New 出来的对象是局部（Local） 型的引用对象。
* JNI 层可通过 DeleteLocalRef 释放 Local 型的引用对象（等同于Java 层中设置持有这个对象的变量的值为 null)。如果不调用 DeleteLocalRef 的话，根据 JNI 规范，Local 型对象在 JNI 函数返回后，也会由虚拟机根据垃圾回收的逻辑进行标记和回收。
* 除了 Local 型对象外，JNI 层借助JNI Global 相关函数可以将一个 Local 型引用对象转换成一个全局（Global） 型对象。而 Global 型对象的回收只能先由程序显式地调用 Global 相关函数进行删除，然后，虚拟机才能借助垃圾回收机制回收它们。

引用类型针对的是除开基本类型的 JNI 类型，比如 jstring, jclass ,jobject 等。JNI 类型是 java 层与 c 层的中间类型，java 层与 c 层都需要管理他。我们可以将 JNI 引用类型理解为 Java 意义的对象。

JNI 类型根据使用的方式可分为：
* 局部引用
* 全部引用
* 弱全部引用

### 局部引用
#### 什么是局部引用？
通过 JNI 接口从 Java 传递下来或者通过 NewLocalRef 和各种 JNI 接口（FindClass、NewObject、GetObjectClass和NewCharArray等）创建的引用称为局部引用。
#### 局部引用的特点？

* 在函数为执行完毕前，局部引用会阻止 GC 回收所引用的对象
* 局部引用不能在本地函数中跨函数使用，不能跨线程使用，当然也不能直接缓存起来使用
* 函数返回后（未返回局部引用的情况下），局部引用所引用的对象会被 JVM 自动释放，也可在函数结束前通过 DeleteLocalRef 函数手动释放
* 如果 c 函数返回了一个局部引用数据，在 java 层，该类型会转换为对应的 java 类型。当 java 层不存在该对象的引用时，gc 就会回收该对象

#### 释放局部引用
局部引用在本地方法执行完会被自动回收，但是有些场景最好是我们手动回收一次。
1. JNI 会将创建的局部引用都存储在一个局部引用表中，如果这个表超过了最大容量限制，就会造成局部引用表溢出，使程序崩溃。经测试，Android上的 JNI 局部引用表最大数量是 512 个。当我们在实现一个本地方法时，可能需要创建大量的局部引用，如果没有及时释放，就有可能导致 JNI 局部引用表的溢出，所以，在不需要局部引用时就立即调用 DeleteLocalRef 手动删除。
2. 在编写 JNI 工具函数时，工具函数在程序当中是公用的，被谁调用你是不知道的。其内部的局部引用在使用完成后应该立即释放，避免过多的内存占用。
3. 如果你的本地函数不会返回。比如一个接收消息的函数，里面有一个死循环，用于等待别人发送消息过来 while(true) { if (有新的消息) ｛ 处理之。。。。｝ else { 等待新的消息。。。}}。如果在消息循环当中创建的引用你不显示删除，很快将会造成JVM局部引用表溢出。
4. 局部引用使用完了就删除，而不是要等到函数结尾才释放，局部引用会阻止所引用的对象被 GC 回收。比如你写的一个本地函数中刚开始需要访问一个大对象，因此一开始就创建了一个对这个对象的引用，但在函数返回前会有一个大量的非常复杂的计算过程，而在这个计算过程当中是不需要前面创建的那个大对象的引用的。但是，在计算的过程当中，如果这个大对象的引用还没有被释放的话，会阻止 GC 回收这个对象，内存一直占用者，造成资源的浪费。所以这种情况下，在进行复杂计算之前就应该把引用给释放了，以免不必要的资源浪费。
言而总之，当一个局部引用不在使用后，立即将其释放，以避免不必要的内存浪费。
#### 本地方法中局部引用的数量
JNI 的规范指出，JVM 要确保每个 Native 方法至少可以创建 16 个局部引用，经验表明，16 个局部引用已经足够平常的使用了。
但是，如果要与 JVM 中的对象进行复杂的交互计算，就需要创建更多的局部引用了，这时就需要使用 EnsureLocalCapacity 来确保可以创建指定数量的局部引用，如果创建成功返回 0 ，返回返回小于 0 ，如下代码示例：

```c++
  // Use EnsureLocalCapacity
    int len = 20;
    if (env->EnsureLocalCapacity(len) < 0) {
        // 创建失败，out of memory
    }
    for (int i = 0; i < len; ++i) {
        jstring  jstr = env->GetObjectArrayElement(arr,i);
        // 处理 字符串
        // 创建了足够多的局部引用，这里就不用删除了，显然占用更多的内存
    }
```
确保可以创建了足够的局部引用数量，所以在循环处理局部引用时可以不进行删除了，但是显然会消耗更多的内存空间了。

循环中的局部引用，有更好的做法：

PushLocalFrame 与 PopLocalFrame 是两个配套使用的函数对。它们可以为局部引用创建一个指定数量内嵌的空间，在这个函数对之间的局部引用都会在这个空间内，直到释放后，所有的局部引用都会被释放掉，不用再担心每一个局部引用的释放问题了。

```c++
 // Use PushLocalFrame & PopLocalFrame
    for (int i = 0; i < len; ++i) {
        if (env->PushLocalFrame(len)) { // 创建指定数据的局部引用空间
            //out ot memory
        }
        jstring jstr = env->GetObjectArrayElement(arr, i);
        // 处理字符串
        // 期间创建的局部引用，都会在 PushLocalFrame 创建的局部引用空间中
        // 调用 PopLocalFrame 直接释放这个空间内的所有局部引用
        env->PopLocalFrame(NULL); 
    }
```
使用 PushLocalFrame & PopLocalFrame 函数对，就可以在期间放心地处理局部引用，最后统一释放掉。
### 全局引用
全局引用可以跨方法、跨线程使用，直到它被手动释放才会失效。同局部引用一样，也会阻止它所引用的对象被 GC 回收。与局部引用不一样的是，函数执行完后，GC 也不会回收全局引用指向的对象。与局部引用创建方式不同的是，只能通过 NewGlobalRef 函数创建。

```c++
   static jclass cls_string = NULL;
    if (cls_string == NULL) {
        jclass local_cls_string = (*env)->FindClass(env, "java/lang/String");
        if (cls_string == NULL) {
            return NULL;
        }

        // 将java.lang.String类的Class引用缓存到全局引用当中
        cls_string = (*env)->NewGlobalRef(env, local_cls_string);

        // 删除局部引用
        (*env)->DeleteLocalRef(env, local_cls_string);

        // 再次验证全局引用是否创建成功
        if (cls_string == NULL) {
            return NULL;
        }
    }
```
当我们的本地代码不再需要一个全局引用时，应该马上调用 DeleteGlobalRef 来释放它。如果不手动调用这个函数，即使这个对象已经没用了，JVM 也不会回收这个全局引用所指向的对象。
### 弱全局引用
弱全局引用使用 NewGlobalWeakRef 创建，使用 DeleteGlobalWeakRef 释放。下面简称弱引用。与全局引用类似，弱引用可以跨方法、线程使用。但与全局引用很重要不同的一点是，弱引用不会阻止 GC 回收它引用的对象。

```c++
    static jclass myCls2 = NULL;
    if (myCls2 == NULL)
    {
        jclass myCls2Local = (*env)->FindClass(env, "mypkg/MyCls2");
        if (myCls2Local == NULL)
        {
            return; /* 没有找到mypkg/MyCls2这个类 */
        }
        myCls2 = NewWeakGlobalRef(env, myCls2Local);
        if (myCls2 == NULL)
        {
            return; /* 内存溢出 */
        }
    }
    ... /* 使用myCls2的引用 */
```
### 引用之间的比较
IsSameObject 用来判断两个引用是否指向相同的对象。还可以用 isSameObject 来比较弱全局引用所引用的对象是否被 GC 了，返回 JNI_TRUE 则表示回收了，JNI_FALSE 则表示未被回收。
```
env->IsSameObject(obj1, obj2) // 比较两个引用是否指向相同的对象
env->IsSameObject(obj, NULL)  // 比较局部引用或者全局引用是否为 NULL
env->IsSameObject(wobj, NULL) // 比较弱全局引用所引用对象是否被 GC 回收
```
一些疑问：如果 C 层返回给 java 层一个全局引用，这个全局引用何时可以被 GC 回收？
我认为不会被 GC 回收，造成内存泄漏。
所以 JNI 函数如果要返回一个对象，我们应该使用局部引用作为返回值。
## 描述符
描述符即JVM对类，数据，方法等的标记方式。
### 类描述符
在 JNI 的 Native 方法中，我们要使用 Java 中的对象怎么办？即在 C/C++ 中怎么找到 Java 中的类，这就要使用到 JNI 开发中的类描述符了
JNI 提供的函数中有个 FindClass() 就是用来查找 Java 类的，其参数必须放入一个类描述符字符串，类描述符一般是类的完整名称（包名+类名）
一个 Java 类对应的描述符，就是类的全名，其中 . 要换成 / :

```
// 完整类名:   java.lang.String
// 对应类描述符: java/lang/String

jclass intArrCls = env->FindClass(“java/lang/String”)

jclass clazz = FindClassOrDie(env, "android/view/Surface");
```
### 域描述符
ta是 JNI 中对 Java 数据类型的一种表示方法。在 JVM 虚拟机中，存储数据类型的名称时，是使用指定的描述符来存储，而不是我们习惯的 int，float 等。
虽然有类描述符，但是类描述符里并没有说明基本类型和数组类型如何表示，所以在 JNI 中就引入了域描述符的概念。
接着我们通过一个表格来了解域描述符的定义：
|类型标识|Java数据类型|
|:-----|:------|
|Z|boolean|
|B|byte|
|C|char|
|S|short|
|I|int|
|J|long|
|F|float|
|D|double|
|L包名/类名;|各种引用类型|
|V|void|
|\[|数组|
|方法|(参数)返回值|

接着我们来看几个例子：
```
Java类型：  java.lang.String
JNI 域描述符：Ljava/lang/String;  //注意结尾有分号

Java类型：   int[]
JNI域描述符： [I

Java类型：   float[]
JNI域描述符： [F

Java类型：   String[]
JNI域描述符： [Ljava/lang/String;

Java类型：   Object[]
JNI域描述符： [Ljava/lang/Object;

Java类型：   int[][]
JNI域描述符： [[I

Java类型：   float[][]
JNI域描述符： [[F
```
### 方法描述符
方法描述符是 JVM 中对函数（方法）的标记方式，看几个例子就能基本掌握其命名特点了：

```
Java 方法                               方法描述符

String fun()                            ()Ljava/lang/String;
int fun(int i, Object object)           (ILjava/lang/Object;)I
void fun(byte[] bytes)                  ([B)V
int fun(byte data1, byte data2)         (BB)I
void fun()                              ()V
```
## JavaVM
JavaVM（Java Virtual Machine）是 Java 程序的运行环境，负责执行 Java 字节码。
在 JNI 开发中，JavaVM 是一个关键的组件，它提供了许多与 Java 运行环境相关的功能，比如加载类、创建对象等。还有管理Java线程，调用Java方法，访问Java对象等。
JavaVM有以下特点：
* JavaVM 是一个结构体，用于描述 Java 虚拟机。
* 一个 JVM 中只有一个 JavaVM 对象。在 Android 平台上，一个 Java 进程只能有一个 ART 虚拟机，也就是说一个进程只有一个 JavaVM 对象。
* JavaVM 可以在进程中的各线程间共享。
JavaVM实例通常是应用启动时自动创建，在 JNI 开发中，通常需要先获取 JavaVM 接口指针。这可以在 JNI_OnLoad 函数中完成。在动态注册的方式中，JNI_OnLoad 是一个由 JNI 库提供的函数，当 Java 虚拟机加载本地库（包含 JNI 代码的库）时会调用这个函数。

```c++
#include <jni.h>
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    // 验证版本
    if (vm->GetEnv((void**)&env, JNI_VERSION_1_6)!= JNI_OK) {
        return -1;
    }
    // 保存JavaVM指针，方便后续使用
    static JavaVM* savedVm = vm;
    return JNI_VERSION_1_6;
}
```
也可以通过 JNIEnv 的函数获取到 JavaVM:

```c++
JavaVM *gJavaVM;

JNIEXPORT jstring JNICALL Java_HelloJNI_sayHello(JNIEnv *env, jobject obj)
{   
    env->GetJavaVM(&gJavaVM);
    return (*env)->NewStringUTF(env,"Hello from JNI !");
}
```

## JNIEnv
JNIEnv 即 Java Native Interface Environment，Java 本地编程接口环境。JNIEnv 内部定义了很多函数用于简化我们的 JNI 编程。
JNI 把 Java 中的所有对象或者对象数组当作一个 C 指针传递到本地方法中，这个指针指向 JVM 中的内部数据结构(对象用jobject来表示，而对象数组用jobjectArray或者具体是基本类型数组)，而内部的数据结构在内存中的存储方式是不可见的。只能从 JNIEnv 指针指向的函数表中选择合适的 JNI 函数来操作JVM 中的数据结构。
### C
在 C 语言中，JNIEnv 是一个指向 JNINativeInterface_ 结构体的指针。JNINativeInterface_ 结构体中定义了非常多的函数指针，这些函数用于简化我们的 JNI 编程。C 语言中，JNIEnv 中函数的使用方式如下:
```
//JNIEnv * env
// env 的实际类型是 JNINativeInterface_**
(*env)->NewStringUTF(env,"Hello from JNI !");
```
### C++
在 C++ 代码中，JNIEnv 是一个 JNIEnv_ 结构体。JNIEnv_ 结构体中同样定义了非常多的成员函数，这些函数用于简化我们的 JNI 编程。C++ 语言中，JNIEnv 中函数的使用方式如下:
```
//JNIEnv * env
// env 的实际类型是 JNIEnv_*
env->NewstringUTF ( "Hello from JNI ! ");
```
可以将其看成每个线程的独立的工具类，方便进行一系列的操作，简化JNI编程。
使用时要区分单线程和多线程的场景。
### 单线程
可以直接通过JNI方法传入的参数拿到指针对象来使用：
```
// 第一个参数就是 JNIEnv
JNIEXPORT jstring JNICALL Java_HelloJNI_sayHello(JNIEnv *env, jobject obj)
{
    return (*env)->NewStringUTF(env,"Hello from JNI !");
}
```
### 多线程
JNIEnv 是一个线程作用域的变量，不能跨线程传递，不同线程的 JNIEnv 彼此独立。多线程使用之前需要先声明一个指针，再将其和线程绑定，指向这个线程自己的JniEnv实例所在的位置。使用完毕之后再解绑定。

```c++
//定义全局变量
//JavaVM 是一个结构体，用于描述 Java 虚拟机，后面会讲
JavaVM* gJavaVM;

JNIEXPORT jstring JNICALL Java_HelloJNI_sayHello(JNIEnv *env, jobject obj)
{   
    //线程不允许共用env环境变量，但是JavaVM指针是整个jvm共用的，所以可以通过下面的方法保存JavaVM指针，在线程中使用
    env->GetJavaVM(&gJavaVM);
    return (*env)->NewStringUTF(env,"Hello from JNI !");
}

//假设这是一个工具函数，可能被多个线程调用
void util_xxx()
{
    JNIEnv *env;
    //从全局的JavaVM中获取到环境变量
    gJavaVM->AttachCurrentThread(&env,NULL);

    //就可以使用 JNIEnv 了

    //最后需要做清理操作
    gJavaVM->DetachCurrentThread();
}
```
一些函数:

|函数名|功能|
|:------|:------|
|FindClass|用于获取类|
|GetObjectClass|通过对象获取这个类|
|NewGlobalRef|创建 obj 参数所引用对象的新全局引用|
|NewObject|构造新 Java 对象|
|NewString|利用 Unicode 字符数组构造新的 java.lang.String 对象|
|NewStringUTF|利用 UTF-8 字符数组构造新的 java.lang.String 对象|
|New\<Type\>Array|创建类型为Type的数组对象|
|Get\<Type\>Field|获取类型为Type的字段|
|Set\<Type\>Field|设置类型为Type的字段的值|
|GetStatic\<Type\>Field|获取类型为Type的static的字段|
|SetStatic\<Type\>Field|设置类型为Type的static的字段的值|
|Call\<Type\>Method|调用返回类型为Type的方法|
|CallStatic\<Type\>Method|调用返回值类型为Type的static方法|

相关的函数不止上面的这些，这些函数的介绍和使用方法。我们可以在开发过程中参考官方文档：
[Oracle官方JNI文档](https://docs.oracle.com/en/java/javase/11/docs/specs/jni/index.html)

## Native访问Java层
### 访问成员变量
访问一个类成员基本分为三步：
* 获取到类对应的 jclass 对象（对应于 Java 层的 Class 对象），jclss 是一个局部引用，使用完后记得使用 DeleteLocalRef 以避免局部引用表溢出。
* 获取到需要访问的类成员的 jfieldID，jfieldID 不是一个 JNI 引用类型，是一个普通指针，指针指向的内存又 JVM 管理，我们无需在使用完后执行 free 清理操作
* 根据被访问对象的类型，使用 GetxxxField 和 SetxxxField 来获得/设置成员变量的值

Java:

```java
//定义一个被访问的类
public class TestJavaClass {

    private String mString = "Hello JNI, this is normal string !";
    
    private static int mStaticInt = 0;
}
//定义两个 native 方法
public native void accessJavaFiled(TestJavaClass testJavaClass);
public native void accessStaticField(TestJavaClass testJavaClass);
```
C++:

```c++
//访问成员变量
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_accessJavaFiled(JNIEnv *env, jobject thiz,jobject test_java_class) {
    jclass clazz;
    jfieldID mString_fieldID;

    //获得 TestJavaClass 的 jclass 对象
    // jclass 类型是一个局部引用
    clazz = env->GetObjectClass(test_java_class);

    if (clazz == NULL) {
        return;
    }

    //获得 mString 的 fieldID
    mString_fieldID = env->GetFieldID(clazz, "mString", "Ljava/lang/String;");
    if (mString_fieldID == NULL) {
        return;
    }

    //获得 mString 的值
    jstring j_string = (jstring) env->GetObjectField(test_java_class, mString_fieldID);
    //GetStringUTFChars 分配了内存，需要使用 ReleaseStringUTFChars 释放
    const char *buf = env->GetStringUTFChars(j_string, NULL);

    //修改 mString 的值
    char *buf_out = "Hello Java, I am JNI!";
    jstring temp = env->NewStringUTF(buf_out);
    env->SetObjectField(test_java_class, mString_fieldID, temp);

    //jfieldID 不是 JNI 引用类型，不用 DeleteLocalRef
    // jfieldID 是一个指针类型，其内存的分配与回收由 JVM 负责，不需要我们去 free
    //free(mString_fieldID);

    //释放内存
    env->ReleaseStringUTFChars(j_string, buf);
    //释放局部引用表
    env->DeleteLocalRef(j_string);
    env->DeleteLocalRef(clazz);

}

//访问静态成员变量
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_accessStaticField(JNIEnv *env, jobject thiz,
                                                            jobject test_java_class) {
    jclass clazz;
    jfieldID mStaticIntFiledID;

    clazz = env->GetObjectClass(test_java_class);

    if (clazz == NULL) {
        return;
    }

    mStaticIntFiledID = env->GetStaticFieldID(clazz, "mStaticInt", "I");

    //获取静态成员
    jint mInt = env->GetStaticIntField(clazz, mStaticIntFiledID);
    //修改静态成员
    env->SetStaticIntField(clazz, mStaticIntFiledID, 10086);

    env->DeleteLocalRef(clazz);
    
}
```
### 调用Java方法
Native 访问一个 Java 方法基本分为三步：
* 获取到类对应的 jclass 对象（对应于 Java 层的 Class 对象），jclss 是一个局部引用，使用完后记得使用 DeleteLocalRef 以避免局部引用表溢出。
* 获取到需要访问的方法的 jmethodID，jmethodID 不是一个 JNI 引用类型，是一个普通指针，指针指向的内存由 JVM 管理，我们无需在使用完后执行 free 清理操作
* 接着就可以调用 CallxxxMethod/CallStaticxxxMethod 来调用对于的方法，xxx 是方法的返回类型。

Java:

```java
//等待被 native 层访问的 java 类
public class TestJavaClass {

    //......
    private void myMethod() {
        Log.i("JNI", "this is java myMethod");
    }

    private static void myStaticMethod() {
        Log.d("JNI", "this is Java myStaticMethod");
    }

}

//本地方法
public native void accessJavaMethod();

public native void accessStaticMethod();
```
C++:

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_accessJavaMethod(JNIEnv *env, jobject thiz) {

    //获取 TestJavaClass 对应的 jclass
    jclass clazz = env->FindClass("com/yuandaima/myjnidemo/TestJavaClass");
    if (clazz == NULL) {
        return;
    }

    //构造函数 id
    jmethodID java_construct_method_id = env->GetMethodID(clazz, "<init>", "()V");

    if (java_construct_method_id == NULL) {
        return;
    }

    //创建一个对象
    jobject object_test = env->NewObject(clazz, java_construct_method_id);
    if (object_test == NULL) {
        return;
    }

    //获得 methodid
    jmethodID java_method_id = env->GetMethodID(clazz, "myMethod", "()V");
    if (java_method_id == NULL) {
        return;
    }

    //调用 myMethod 方法
    env->CallVoidMethod(object_test,java_method_id);

    //清理临时引用吧  
    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(object_test);
}
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_accessStaticMethod(JNIEnv *env, jobject thiz) {

    jclass clazz = env->FindClass("com/yuandaima/myjnidemo/TestJavaClass");
    if (clazz == NULL) {
        return;
    }

    jmethodID static_method_id = env->GetStaticMethodID(clazz, "myStaticMethod", "()V");
    if(NULL == static_method_id)
    {
        return;
    }

    env->CallStaticVoidMethod(clazz, static_method_id);

    env->DeleteLocalRef(clazz);

}
```
## 异常处理
### JNIEnv 内部函数抛出的异常
很多 JNIEnv 中的函数都会抛出异常，处理方法大体上是一致的：
* 返回值与特殊值（一般是 NULL）比较，知晓函数是否发生异常
* 如果发生异常立即 return
* jvm 会将异常抛给 java 层，我们可以在 java 层通过 try catch 机制捕获异常

JAVA:

```java
public native void exceptionTest();

//调用
try {
     exceptionTest();
} catch (Exception e) {
    e.printStackTrace();
}
```
C++:

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_exceptionTest(JNIEnv *env, jobject thiz) {   
    //查找的类不存在，返回 NULL；
    jclass clazz = env->FindClass("com/yuandaima/myjnidemo/xxx");
    if (clazz == NULL) {
        return; //return 后，jvm 会向 java 层抛出 ClassNotFoundException
    }
}

// result:
java.lang.ClassNotFoundException: Didn't find class "com.yuandaima.myjnidemo.xxx"Native 回调 Java 层方法，被回调的方法抛出异常
```

Native 回调 Java 层方法，被回调的方法抛出异常。这样情况下一般有两种解决办法：
* Java 层 Try catch 本地方法，这是比较推荐的办法。
* Native 层处理异常，异常处理如果和 native 层相关，可以采用这种方式
### Native层不处理异常，Java层来处理异常
java:

```java
//执行这个方法会抛出异常
private static int exceptionMethod() {
    return 20 / 0;
}

//native 方法，在 native 中，会调用到 exceptionMethod() 方法
public native void exceptionTest();

// MainActivity中调用是加上try-catch:
//Java 层调用
try {
    exceptionTest();
} catch (Exception e) {
    //这里处理异常
    //一般是打 log 和弹 toast 通知用户
    e.printStackTrace();
}
```
C++:

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_exceptionTest(JNIEnv *env, jobject thiz) {
    jclass clazz = env->FindClass("com/yuandaima/myjnidemo/TestJavaClass");
    if (clazz == NULL) {
        return;
    }

    //调用 java 层会抛出异常的方法
    jmethodID static_method_id = env->GetStaticMethodID(clazz, "exceptionMethod", "()I");

    if (NULL == static_method_id) {
        return;
    }

    //直接调用，发生 ArithmeticException 异常，传回 Java 层
    env->CallStaticIntMethod(clazz, static_method_id);

    env->DeleteLocalRef(clazz);
}
```
### Native来处理异常
有的异常需要在 Native 处理，这里又分为两类：
* 异常在 Native 层就处理完了
* 异常在 Native 层处理了，还需要返回给 Java 层，Java 层继续处理

java:

```java
//执行这个方法会抛出异常
private static int exceptionMethod() {
    return 20 / 0;
}

//native 方法，在 native 中，会调用到 exceptionMethod() 方法
public native void exceptionTest();

//Java 层调用
try {
    exceptionTest();
} catch (Exception e) {
    //这里处理异常
    //一般是打 log 和弹 toast 通知用户
    e.printStackTrace();
}
```
C++:

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_exceptionTest(JNIEnv *env, jobject thiz) {
    jthrowable mThrowable;
    jclass clazz = env->FindClass("com/yuandaima/myjnidemo/TestJavaClass");
    if (clazz == NULL) {
        return;
    }

    jmethodID static_method_id = env->GetStaticMethodID(clazz, "exceptionMethod", "()I");
    if (NULL == static_method_id) {
        return;
    }

    env->CallStaticIntMethod(clazz, static_method_id);

    //检测是否有异常发生
    if (env->ExceptionCheck()) {
        //获取到异常对象
        mThrowable = env->ExceptionOccurred();
        //这里就可以根据实际情况处理异常了
        //.......
        //打印异常信息堆栈
        env->ExceptionDescribe();
        //清除异常信息
        //如果，异常还需要 Java 层处理，可以不调用 ExceptionClear，让异常传递给 Java 层
        env->ExceptionClear();
        //如果调用了 ExceptionClear 后，异常还需要 Java 层处理，我们可以抛出一个新的异常给 Java 层
        jclass clazz_exception = env->FindClass("java/lang/Exception");
        env->ThrowNew(clazz_exception, "JNI抛出的异常！");

        env->DeleteLocalRef(clazz_exception);
    }

    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(mThrowable);
}
```
## 引用类型的内存分析
### Java 程序使用的内存
从逻辑上可以分为两个部分：

* Java Memory
* Native Memory

Java Memory 就是我们的 Java 程序使用的内存，通常从逻辑上区分为栈和堆。方法中的局部变量通常存储在栈中，引用类型指向的对象一般存储在堆中。Java Memory 由 JVM 分配和管理，JVM 中通常会有一个 GC 线程，用于回收不再使用的内存。

Java 程序的执行依托于 JVM ，JVM 一般使用 C/C++ 代码编写，需要根据 Native 编程规范去操作内存。如：C/C++ 使用 malloc()/new 分配内存，需要手动使用 free()/delete 回收内存。这部分内存我们称为 Native Memory。

Java 中的对象对应的内存，由 JVM 来管理，他们都有自己的数据结构。当我们通过 JNI 将一个 Java 对象传递给 Native 程序时，Native 程序要操作这块内存时（即操作这个对象），就需要了解这个数据结构，显然这有点麻烦了，所以 JVM 的设计者在 JNIenv 中定义了很多函数（NewStringUTF，FindClass，NewObject 等）来帮你操作和构造这些对象。同时也提供了引用类型（jobject、jstring、jclass、jarray、jintArray等）来引用这些对象。
### 明确引用类型的范围
引用类型是指针，指向的是 Java 中的对象在 JVM 中对应的内存。引用类型的定义如下：

```c++
#ifdef __cplusplus

class _jobject {};
class _jclass : public _jobject {};
class _jthrowable : public _jobject {};
class _jstring : public _jobject {};
class _jarray : public _jobject {};
class _jbooleanArray : public _jarray {};
class _jbyteArray : public _jarray {};
class _jcharArray : public _jarray {};
class _jshortArray : public _jarray {};
class _jintArray : public _jarray {};
class _jlongArray : public _jarray {};
class _jfloatArray : public _jarray {};
class _jdoubleArray : public _jarray {};
class _jobjectArray : public _jarray {};

typedef _jobject *jobject;
typedef _jclass *jclass;
typedef _jthrowable *jthrowable;
typedef _jstring *jstring;
typedef _jarray *jarray;
typedef _jbooleanArray *jbooleanArray;
typedef _jbyteArray *jbyteArray;
typedef _jcharArray *jcharArray;
typedef _jshortArray *jshortArray;
typedef _jintArray *jintArray;
typedef _jlongArray *jlongArray;
typedef _jfloatArray *jfloatArray;
typedef _jdoubleArray *jdoubleArray;
typedef _jobjectArray *jobjectArray;

#else

struct _jobject;

typedef struct _jobject *jobject;
typedef jobject jclass;
typedef jobject jthrowable;
typedef jobject jstring;
typedef jobject jarray;
typedef jarray jbooleanArray;
typedef jarray jbyteArray;
typedef jarray jcharArray;
typedef jarray jshortArray;
typedef jarray jintArray;
typedef jarray jlongArray;
typedef jarray jfloatArray;
typedef jarray jdoubleArray;
typedef jarray jobjectArray;

#endif
```
不是以上类型的指针就不是 JNI 引用类型，比如容易混淆的 jmethod jfield 都不是 JNI 引用类型。

JNI 引用类型是指针，但是和 C/C++ 中的普通指针不同，C/C++ 中的指针需要我们自己分配和回收内存（C/C++ 使用 malloc()/new 分配内存，需要手动使用 free()/delete 回收内存）。JNI 引用不需要我们分配和回收内存，这部分工作由 JVM 完成。我们额外需要做的工作是在 JNI 引用类型使用完后，将其从引用表中删除，防止引用表满了。
### 局部引用
通过 JNI 接口从 Java 传递下来或者通过 NewLocalRef 和各种 JNI 接口（FindClass、NewObject、GetObjectClass和NewCharArray等）创建的引用称为局部引用。

当从 Java 环境切换到 Native 环境时，JVM 分配一块内存用于创建一个 Local Reference Table，这个 Table 用来存放本次 Native Method 执行中创建的所有局部引用（Local Reference）。每当在 Native 代码中引用到一个 Java 对象时，JVM 就会在这个 Table 中创建一个 Local Reference。比如，我们调用 NewStringUTF() 在 Java Heap 中创建一个 String 对象后，在 Local Reference Table 中就会相应新增一个 Local Reference。

对于开发者来说，Local Reference Table 是不可见的，Local Reference Table 的内存不大，所能存放的 Local Reference 数量也是有限的（在 Android 中默认最大容量是512个）。在开发中应该及时使用 DeleteLocalRef( )删除不必要的 Local Reference，不然可能会出现溢出错误。

很多人会误将 JNI 中的 Local Reference 理解为 Native Code 的局部变量。这是错误的：
* 局部变量存储在线程堆栈中，而 Local Reference 存储在 Local Ref 表中。
* 局部变量在函数退栈后被删除，而 Local Reference 在调用 DeleteLocalRef() 后才会从 Local Ref 表中删除，并且失效，或者在整个 Native Method 执行结束后被删除。
* 可以在代码中直接访问局部变量，而 Local Reference 的内容无法在代码中直接访问，必须通过 JNI function 间接访问。JNI function 实现了对 Local Reference 的间接访问，JNI function 的内部实现依赖于具体 JVM。

### 全局引用
Global Reference 是通过 JNI 函数 NewGlobalRef() 和D eleteGlobalRef() 来创建和删除的。 Global Reference 具有全局性，可以在多个 Native Method 调用过程和多线程中使用。

使用 Global reference时，当 native code 不再需要访问 Global reference 时，应当调用 JNI 函数 DeleteGlobalRef() 删除 Global reference 和它引用的 Java 对象。否则 Global Reference 引用的 Java 对象将永远停留在 Java Heap 中，从而导致 Java Heap 的内存泄漏。

### 弱全局引用
弱全局引用使用 NewWeakGlobalRef() 和 DeleteWeakGlobalRef() 进行创建和删除，它与 Global Reference 的区别在于该类型的引用随时都可能被 GC 回收。

对于 Weak Global Reference 而言，可以通过 isSameObject() 将其与 NULL 比较，看看是否已经被回收了。如果返回 JNI_TRUE，则表示已经被回收了，需要重新初始化弱全局引用。

Weak Global Reference 的回收时机是不确定的，有可能在前一行代码判断它是可用的，后一行代码就被 GC 回收掉了。为了避免这类事情发生，JNI官方给出了正确的做法，通过 NewLocalRef() 获取 Weak Global Reference，避免被GC回收。

### JNI性能优化
* Java 程序中，调用一个 Native 方法相比调用一个 Java 方法要耗时很多，我们应该减少 JNI 方法的调用，同时一次 JNI 调用尽量完成更多的事情。对于过于耗时的 JNI 调用，应该放到后台线程调用。
* Native 程序要访问 Java 对象的字段或调用它们的方法时，本机代码必须调用 FindClass()、GetFieldID()、GetStaticFieldID、GetMethodID() 和 GetStaticMethodID() 等方法，返回的 ID 不会在 JVM 进程的生存期内发生变化。但是，获取字段或方法的调用有时会需要在 JVM 中完成大量工作，因为字段和方法可能是从超类中继承而来的，这会让 JVM 向上遍历类层次结构来找到它们。为了提高性能，我们可以把这些 ID 缓存起来，用内存换性能。

#### 缓存java字段，方法ID
java:

```java
public class TestJavaClass {

    //......
    private void myMethod() {
        Log.i("JNI", "this is java myMethod");
    }
    //......
}

public native void cacheTest();
```
C++:

```c++
extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_cacheTest(JNIEnv *env, jobject thiz) {

    jclass clazz = env->FindClass("com/yuandaima/myjnidemo/TestJavaClass");
    if (clazz == NULL) {
        return;
    }

    static jmethodID java_construct_method_id = NULL;
    static jmethodID java_method_id = NULL;

    //实现缓存的目的，下次调用不用再获取 methodid 了
    if (java_construct_method_id == NULL) {
        //构造函数 id
        java_construct_method_id = env->GetMethodID(clazz, "<init>", "()V");
        if (java_construct_method_id == NULL) {
            return;
        }
    }

    //调用构造函数，创建一个对象
    jobject object_test = env->NewObject(clazz, java_construct_method_id);
    if (object_test == NULL) {
        return;
    }
    //相同的手法，缓存 methodid
    if (java_method_id == NULL) {
        java_method_id = env->GetMethodID(clazz, "myMethod", "()V");
        if (java_method_id == NULL) {
            return;
        }
    }

    //调用 myMethod 方法
    env->CallVoidMethod(object_test, java_method_id);

    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(object_test);
}
```
主要是通过一个全局变量保存 methodid，这样只有第一次调用 native 函数时，才会调用 GetMethodID 去获取，后面的调用都使用缓存起来的值了。这样就避免了不必要的调用，提升了性能。
#### 静态初始化
java:

```java
static {
    System.loadLibrary("myjnidemo");
    initIDs();
}

public static native void initIDs();
```
C++:

```c++
//定义用于缓存的全局变量
static jmethodID java_construct_method_id2 = NULL;
static jmethodID java_method_id2 = NULL;

extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_initIDs(JNIEnv *env, jclass clazz) {

    jclass clazz2 = env->FindClass("com/yuandaima/myjnidemo/TestJavaClass");

    if (clazz == NULL) {
        return;
    }

    //实现缓存的目的，下次调用不用再获取 methodid 了
    if (java_construct_method_id2 == NULL) {
        //构造函数 id
        java_construct_method_id2 = env->GetMethodID(clazz2, "<init>", "()V");
        if (java_construct_method_id2 == NULL) {
            return;
        }
    }

    if (java_method_id2 == NULL) {
        java_method_id2 = env->GetMethodID(clazz2, "myMethod", "()V");
        if (java_method_id2 == NULL) {
            return;
        }
    }
}
```
手法和使用时缓存是一样的，只是缓存的时机变了。如果是动态注册的 JNI 还可以在 Onload 函数中来执行缓存操作。
## 多线程Demo
JNI 环境下，进行多线程编程，有以下两点是需明确的：
* JNIEnv 是一个线程作用域的变量，不能跨线程传递，每个线程都有自己的 JNIEnv 且彼此独立
* 局部引用不能在本地函数中跨函数使用，不能跨线程使用，当然也不能直接缓存起来使用

java:

```java
public void javaCallback(int count) {
    Log.e(TAG, "onNativeCallBack : " + count);
}

public native void threadTest();
```
C++:

```c++
static int count = 0;
JavaVM *gJavaVM = NULL;//全局 JavaVM 变量
jobject gJavaObj = NULL;//全局 Jobject 变量
jmethodID nativeCallback = NULL;//全局的方法ID

//这里通过标志位来确定 两个线程的工作都完成了再执行 DeleteGlobalRef
//当然也可以通过加锁实现
bool main_finished = false;
bool background_finished = false;

static void *native_thread_exec(void *arg) {

    LOGE(TAG, "nativeThreadExec");
    LOGE(TAG, "The pthread id : %d\n", pthread_self());
    JNIEnv *env;
    //从全局的JavaVM中获取到环境变量
    gJavaVM->AttachCurrentThread(&env, NULL);

    //线程循环
    for (int i = 0; i < 5; i++) {
        usleep(2);
        //跨线程回调Java层函数
        env->CallVoidMethod(gJavaObj, nativeCallback, count++);
    }
    gJavaVM->DetachCurrentThread();

    background_finished = true;

    if (main_finished && background_finished) {
        env->DeleteGlobalRef(gJavaObj);
        LOGE(TAG, "全局引用在子线程销毁");
    }

    return ((void *) 0);

}


extern "C"
JNIEXPORT void JNICALL
Java_com_yuandaima_myjnidemo_MainActivity_threadTest(JNIEnv *env, jobject thiz) {
    //创建全局引用，方便其他函数或线程使用
    gJavaObj = env->NewGlobalRef(thiz);
    jclass clazz = env->GetObjectClass(thiz);
    nativeCallback = env->GetMethodID(clazz, "javaCallback", "(I)V");
    //保存全局 JavaVM，注意 JavaVM 不是 JNI 引用类型
    env->GetJavaVM(&gJavaVM);

    pthread_t id;
    if (pthread_create(&id, NULL, native_thread_exec, NULL) != 0) {
        return;
    }

    for (int i = 0; i < 5; i++) {
        usleep(20);
        //跨线程回调Java层函数
        env->CallVoidMethod(gJavaObj, nativeCallback, count++);
    }

    main_finished = true;

    if (main_finished && background_finished && !env->IsSameObject(gJavaObj, NULL)) {
        env->DeleteGlobalRef(gJavaObj);
        LOGE(TAG, "全局引用在主线程销毁");
    }
}
```
示例代码中，我们的子线程需要使用主线程中的 jobject thiz，该变量是一个局部引用，不能赋值给一个全局变量然后跨线程跨函数使用，我们通过 NewGlobalRef 将局部引用装换为全局引用并保存在全局变量 jobject gJavaObj 中，在使用完成后我们需要使用 DeleteGlobalRef 来释放全局引用，因为多个线程执行顺序的不确定性，我们使用了标志位来确保两个线程所有的工作完成后再执行释放操作。
## C++线程安全 Memory Order
### 为什么需要 Memory Order
如果不使用任何同步机制（例如 mutex 或 atomic），在多线程中读写同一个变量，那么，程序的结果是难以预料的。主要原因有一下几点：
简单的读写不是原子操作
CPU 可能会调整指令的执行顺序
在 CPU cache 的影响下，一个 CPU 执行了某个指令，不会立即被其它 CPU 看见
### 非原子操作给多线程编程带来的影响
原子操作说的是，一个操作的状态要么就是未执行，要么就是已完成，不会看见中间状态。
下面看一个非原子操作给多线程编程带来的影响：

```
  int64_t i = 0;     // global variable
Thread-1:              Thread-2:
i++;               std::cout << i;
```
C++ 并不保证 i++ 是原子操作。从汇编的角度看，读写内存的操作一般分为三步：
* 将内存单元读到 cpu 寄存器
* 修改寄存器中的值
* 将寄存器中的值回写入对应的内存单元

进一步，有的 CPU Architecture， 64 位数据（int64_t）在内存和寄存器之间的读写需要两条指令。
这就导致了 i++ 操作在 cpu 的角度是一个多步骤的操作。所以 Thread-2 读到的可能是一个中间状态。

### 指令的执行顺序调整给多线程编程带来的影响
为了优化程序的执行性能，编译器和 CPU 可能会调整指令的执行顺序。为阐述这一点，下面的例子中，让我们假设所有操作都是原子操作：

```
    int x = 0;     // global variable
          int y = 0;     // global variable
  
Thread-1:              Thread-2:
x = 100;               while (y != 200) {}
y = 200;               std::cout << x;
```
如果 CPU 没有乱序执行指令，那么 Thread-2 将输出 100。然而，对于 Thread-1 来说，x = 100; 和 y = 200; 这两个语句之间没有依赖关系，因此，Thread-1 允许调整语句的执行顺序：

```
Thread-1:
y = 200;
x = 100;
```
在这种情况下，Thread-2 将输出 0 或 100。
### CPU CACHE 对多线程程序的影响
CPU cache 也会影响到程序的行为。下面的例子中，假设从时间上来讲，A 操作先于 B 操作发生：

```
     int x = 0;     // global variable
  
Thread-1:                      Thread-2:
x = 100;    // A               std::cout << x;    // B
```
尽管从时间上来讲，A 先于 B，但 CPU cache 的影响下，Thread-2 不能保证立即看到 A 操作的结果，所以 Thread-2 可能输出 0 或 100。
### 同步机制
对于 C++ 程序来说，解决以上问题的办法就是使用同步机制，最常见的同步机制就是 std::mutex和 std::atomic。从性能角度看，通常使用 std::atomic 会获得更好的性能。
C++ 提供了四种 memory ordering ：

* Relaxed ordering
* Release-Acquire ordering
* Release-Consume ordering
* Sequentially-consistent ordering

#### Relaxed ordering
在这种模型下，std::atomic 的 load() 和 store() 都要带上 memory_order_relaxed 参数。Relaxed ordering 仅仅保证 load() 和 store() 是原子操作，除此之外，不提供任何跨线程的同步。
先看看一个简单的例子：

```
         std::atomic<int> x = 0;     // global variable
         std::atomic<int> y = 0;     // global variable
  
Thread-1:                              Thread-2:
//A                                    // C
r1 = y.load(memory_order_relaxed);     r2 = x.load(memory_order_relaxed); 
//B                                    // D
x.store(r1, memory_order_relaxed);     y.store(42, memory_order_relaxed); 
```
执行完上面的程序，可能出现 r1 == r2 == 42。理解这一点并不难，因为编译器允许调整 C 和 D 的执行顺序。如果程序的执行顺序是 D -> A -> B -> C，那么就会出现 r1 == r2 == 42。
如果某个操作只要求是原子操作，除此之外，不需要其它同步的保障，就可以使用 Relaxed ordering。程序计数器是一种典型的应用场景：

```c++
#include <cassert>
#include <vector>
#include <iostream>
#include <thread>
#include <atomic>
std::atomic<int> cnt = {0};
void f()
{
    for (int n = 0; n < 1000; ++n) {
        cnt.fetch_add(1, std::memory_order_relaxed);
    }
}
int main()
{
    std::vector<std::thread> v;
    for (int n = 0; n < 10; ++n) {
        v.emplace_back(f);
    }
    for (auto& t : v) {
        t.join();
    }
    assert(cnt == 10000);    // never failed
    return 0;
}
```
#### Release-Acquire ordering
在这种模型下，store() 使用 memory_order_release，而 load() 使用 memory_order_acquire。这种模型有两种效果，第一种是可以限制 CPU 指令的重排：
在 store() 之前的所有读写操作，不允许被移动到这个 store() 的后面。
在 load() 之后的所有读写操作，不允许被移动到这个 load() 的前面。
除此之外，还有另一种效果：假设 Thread-1 store() 的那个值，成功被 Thread-2 load() 到了，那么 Thread-1 在 store() 之前对内存的所有写入操作，此时对 Thread-2 来说，都是可见的。
下面的例子阐述了这种模型的原理：

```c++
#include <thread>
#include <atomic>
#include <cassert>
#include <string>

std::atomic<bool> ready{ false };
int data = 0;
void producer()
{
    data = 100;                                       // A
    ready.store(true, std::memory_order_release);     // B
}
void consumer()
{
    while (!ready.load(std::memory_order_acquire)){}    // C
    assert(data == 100); // never failed              // D
}
int main()
{
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```
让我们分析一下这个过程：
首先 A 不允许被移动到 B 的后面。
同样 D 也不允许被移动到 C 的前面。
当 C 从 while 循环中退出了，说明 C 读取到了 B store()的那个值，此时，Thread-2 保证能够看见 Thread-1 执行 B 之前的所有写入操作（也即是 A）。
#### Release-Consume ordering
在这种模型下，store() 使用 memory_order_release，而 load() 使用 memory_order_consume。这种模型有两种效果，第一种是可以限制 CPU 指令的重排：
在 store() 之前的与原子变量相关的所有读写操作，不允许被移动到这个 store() 的后面。
在 load() 之后的与原子变量相关的所有读写操作，不允许被移动到这个 load() 的前面。
除此之外，还有另一种效果：假设 Thread-1 store() 的那个值，成功被 Thread-2 load() 到了，那么 Thread-1 在 store() 之前对与原子变量相关的内存的所有写入操作，此时对 Thread-2 来说，都是可见的。
下面的例子阐述了这种模型的原理：

```c++
#include <thread>
#include <atomic>
#include <cassert>
#include <string>
 
std::atomic<std::string*> ptr;
int data;
 
void producer()
{
    std::string* p  = new std::string("Hello");  //A
    data = 42;
    //ptr依赖于p
    ptr.store(p, std::memory_order_release);   //B
}
 
void consumer()
{
    std::string* p2;
    while (!(p2 = ptr.load(std::memory_order_consume))) //C
        ;
    // never fires: *p2 carries dependency from ptr
    assert(*p2 == "Hello");                           //D
    // may or may not fire: data does not carry dependency from ptr
    assert(data == 42); 
}
 
int main()
{
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join(); t2.join();
}
```

让我们分析一下这个过程：
首先 A 不允许被移动到 B 的后面。
同样 D 也不允许被移动到 C 的前面。
data 与 ptr 无关，不会限制他的重排序
当 C 从 while 循环中退出了，说明 C 读取到了 B store()的那个值，此时，Thread-2 保证能够看见 Thread-1 执行 B 之前的与原子变量相关的所有写入操作（也即是 A）。
#### Sequentially-consistent ordering
Sequentially-consistent ordering 是缺省设置，在 Release-Acquire ordering 限制的基础上，保证了所有设置了 memory_order_seq_cst 标志的原子操作按照代码的先后顺序执行。
## 常见问题解答：为什么我会收到 UnsatisfiedLinkError？
在处理原生代码时，经常可以看到如下所示的失败消息：
java.lang.UnsatisfiedLinkError: Library foo not found
在某些情况下，正如字面意思所说 - 找不到库。在 出现此库但无法被 dlopen(3) 打开的其他情况，以及 您可在异常的详情消息中找到失败详情。
您可能遇到“找不到库”异常的常见原因如下：

* 库不存在或应用无法访问。使用 adb shell ls -l <path>，检查其是否存在 和权限。
* 库不是使用 NDK 构建的。这可能会导致 对设备上不存在的函数或库的依赖关系。

其他类的 UnsatisfiedLinkError 失败消息如下所示：

```
java.lang.UnsatisfiedLinkError: myfunc
        at Foo.myfunc(Native Method)
        at Foo.main(Foo.java:10)
```
在 logcat 中，您将看到以下内容：

```
W/dalvikvm(  880): No implementation found for native LFoo;.myfunc ()V
```
这意味着，运行时尝试查找匹配方法， 失败。造成此问题的一些常见原因如下：

* 库未加载。检查 logcat 输出， 库加载消息
* 名称或签名不匹配，因此找不到该方法。本次 通常由以下原因引起： 
    ◦ 对于延迟方法查找，无法声明 C++ 函数 包含 extern "C" 和适当的 可见性 (JNIEXPORT)。请注意，在投放 Ice Cream 之前， Sandwich 中，JNIEXPORT 宏不正确，因此请使用带有 旧的jni.h将无法使用。 您可以使用 arm-eabi-nm 查看符号在库中显示的符号；如果它们 损坏（诸如_Z15Java_Foo_myfuncP7_JNIEnvP7_jclass之类的内容） 而不是 Java_Foo_myfunc），或者如果符号类型是 小写“t”而不是大写的“T”，则需要 调整声明
    ◦ 对于显式注册，在输入 方法签名。请确保您传递到 与日志文件中的签名匹配。 记住“B”为“byte”和“Z”为 boolean。 签名中的类名称组成部分以“L”开头，以“;”结尾， 使用“/”分隔软件包/类名称，然后使用“$”来分隔 内部类名称（比如 Ljava/util/Map$Entry;）。

使用 javah 自动生成 JNI 标头可能会有帮助 可以避免一些问题。
## 常见问题解答：为什么 FindClass 找不到我的类？
（以下建议的大部分内容同样适用于找不到方法的问题 包含 GetMethodID、GetStaticMethodID 或字段 使用 GetFieldID 或 GetStaticFieldID。）
确保类名称字符串的格式正确无误。JNI 类 名称以软件包名称开头，并用正斜线分隔 例如 java/lang/String。如果您要查找数组类， 您需要以适当数量的方括号开头 还必须使用“L”封装类和“;”这样的一个一维数组， String 为 \[Ljava/lang/String;。 如果您要查找内部类，请使用“$”而不是“.”。一般来说， 对 .class 文件使用 javap 是查找 类的内部名称。
如果要启用代码缩减，请确保 配置要保留的代码。正在配置 适当的保留规则非常重要，因为代码压缩器可能会从别处移除类、方法 或仅通过 JNI 使用的字段。
如果类名称没有问题，则可能是因为您遇到了类加载器 问题。FindClass想要在 与代码关联的类加载器。它会检查调用堆栈， 如下所示：

```
    Foo.myfunc(Native Method)
    Foo.main(Foo.java:10)
```
最顶层的方法是 Foo.myfunc。FindClass 查找与 Foo 关联的 ClassLoader 对象 类并使用该类。
采用这种方法通常会完成您想要执行的操作。如果您 自行创建线程（可能通过调用 pthread_create） ，然后使用 AttachCurrentThread 附加该映像）。现在 不是来自应用的堆栈帧。 如果您从此线程调用 FindClass， JavaVM 将在“system”下启动类加载器，而不是 与您的应用关联，因此尝试查找特定于应用的类将失败。
您可以通过以下几种方法来解决此问题：

* FindClass JNI_OnLoad，并缓存类引用以供日后使用 。执行过程中进行的任何 FindClass 调用 JNI_OnLoad 将使用与 调用 System.loadLibrary 的函数（这是 特殊规则，以方便执行库初始化）。 如果您的应用代码要加载库，请FindClass 将使用正确的类加载器。
* 将类的实例传递到 方法是声明原生方法接受 Class 参数 然后传入 Foo.class。
* 在某处缓存对 ClassLoader 对象的引用 然后直接发出 loadClass 调用。这需要 不费吹灰之力

## 常见问题解答：如何使用原生代码共享原始数据？
您可能会发现自己需要访问大量 来自受管理代码和原生代码的原始数据的缓冲区。常见示例 包含对位图或声音样本的操纵。有两个 基本方法。
您可以将数据存储在 byte[] 中。这样可让系统 通过托管代码进行访问。而在原生广告方面 访问相应数据，而无需复制数据。在 一些实现，GetByteArrayElements 和 GetPrimitiveArrayCritical 将返回指向 托管堆中的原始数据，但在其他情况下，系统会分配缓冲区 并复制数据
另一种方法是将数据存储在直接字节缓冲区中。这些 可使用 java.nio.ByteBuffer.allocateDirect 创建，或 JNI NewDirectByteBuffer 函数。不同于普通 字节缓冲区，因此存储空间不会在托管堆上分配，并且可以 始终直接从原生代码中访问（获取 与 GetDirectBufferAddress 相关联）。取决于 已实现字节缓冲区访问，从托管代码访问数据 可能会非常慢
选择使用哪种方法取决于以下两个因素：
1. 大部分数据访问是通过使用 Java 编写的代码进行的吗 还是用 C/C++ 开发？
2. 如果数据最终传递到系统 API，以 应该在其中吗？（例如，如果数据最终传递到 函数，它接受一个 byte[] 值，直接 ByteBuffer 可能不太明智。）
如果这两种方法不分伯仲，请使用直接字节缓冲区。为他们提供的支持 直接内置在 JNI 中，性能应该会在未来版本中得到改进。
