---
layout: post
description: > 
  本文介绍了Android平台的图片显示库Coil，其优化点和基本使用方式
image: 
  path: /assets/img/blog/blogs_android_image_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_image_cover.png
    960w:  /assets/img/blog/blogs_android_image_cover.png
    480w:  /assets/img/blog/blogs_android_image_cover.png
accent_image: /assets/img/blog/blogs_android_image_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】Coil图片加载库介绍
## Android图片加载体系
在 Android 中，加载并显示一张图片文件（如 JPEG、PNG）到屏幕上，核心机制是 **Bitmap -> Drawable -> ImageView** 的配合使用。

Bitmap 存储图片在内存中的实际像素数据（如 RGB 值）。Drawable 属于抽象层，代表可绘制对象，是所有可绘制内容的抽象基类，作为 Bitmap 与 View 之间的桥梁，管理 Bitmap 的绘制状态和尺寸信息。ImageView负责将 `Drawable` 对象的内容绘制到屏幕上，并处理用户交互。
### 配合加载一张图片文件的完整流程
当一张图片文件（例如 `image.jpg`）从磁盘或网络被加载，直到最终显示在 `ImageView` 中，主要分为以下三个步骤：
#### 步骤 1: 图片数据解码成 Bitmap（数据准备）
图片文件本身是经过压缩的（如 JPEG），不能直接显示。这一步的任务是**将压缩数据解压并解码成原始像素数据**。
1.  **解码：** 使用 `BitmapFactory` 或 `ImageDecoder` 等 API，将 `image.jpg` 文件读取为字节流。
2.  **生成 Bitmap：** 解码器根据字节流，在内存（RAM）中开辟一块空间，将图片的像素数据填充进去，创建出 `Bitmap` 对象。
3.  **内存占用：** 此时 `Bitmap` 占用的内存大小 = 图片像素宽 × 图片像素高 × 每个像素占用的字节数（如 ARGB_8888 模式下占 4 字节）。

> **关键代码：** `BitmapFactory.decodeFile(path)` 或 `ImageDecoder.decodeBitmap(source)`

#### 步骤 2: Bitmap 封装成 Drawable（状态管理）
`Bitmap` 纯粹是像素数据，而 Android 的 View 系统需要一个**可绘制对象** (`Drawable`) 来进行绘制和状态管理。
1.  **封装：** `Bitmap` 对象会被封装到一个具体的 `Drawable` 子类中，最常见的是 `BitmapDrawable`。
2.  **提供信息：** `BitmapDrawable` 获得了 `Bitmap` 像素信息后，还添加了绘制所需的额外信息，比如：
    * **固有的宽高 (`getIntrinsicWidth`/`getIntrinsicHeight`)：** 来源于 `Bitmap` 的像素尺寸。
    * **不透明度、颜色过滤、状态（选中/按下等）：** 允许在绘制时对 `Bitmap` 进行调整和控制。

> **关键代码（底层）：** `BitmapDrawable drawable = new BitmapDrawable(resources, bitmap);`

#### 步骤 3: ImageView 绘制 Drawable（视图呈现）
`ImageView` 是最终的显示容器，它负责将 `Drawable` 的内容呈现在屏幕上。
1.  **设置 Drawable：** 通过 `imageView.setImageDrawable(drawable)` 或 `imageView.setImageBitmap(bitmap)`（内部会自动封装成 `BitmapDrawable`）将 Drawable 对象交给 `ImageView`。
2.  **计算尺寸：** `ImageView` 会根据自身的布局参数（如 `layout_width`、`layout_height`）和 `scaleType`（如 `centerCrop`、`fitCenter`）来决定如何缩放和裁剪内部的 `Drawable`。
3.  **触发绘制：** 在 `ImageView` 的 `onDraw()` 方法中，它会调用 `Drawable.draw(canvas)` 方法，让 `Drawable` 将其内部的 `Bitmap` 绘制到 `ImageView` 的画布（Canvas）上，最终呈现在屏幕上。

> **关键代码（上层）：** `imageView.setImageBitmap(bitmap)` 或在 XML 中使用 `android:src="@drawable/..."`。

## Coil
Coil (全称 **Co**routine **I**mage **L**oader) 是一个专为 Android 打造的现代化图片加载库，它之所以被认为是高效的，主要得益于其现代化的架构和多项针对性的优化。

以下是 Coil 优化的主要方面：
### 1. 核心架构优化 (基于 Kotlin 协程)
这是 Coil 最核心的优化点。Coil 的名字就来源于此。它完全基于 Kotlin 协程 (Coroutines) 来执行所有的异步操作（如网络请求、磁盘I/O、图片解码）。

相比于传统的线程池或 `AsyncTask`，协程更加轻量级。它们可以挂起 (suspend) 而不阻塞线程，从而用更少的线程处理大量的并发请求。这减少了线程切换的开销，提高了吞吐量，并且能非常简单地实现请求的取消和管理。
### 2. 内存优化
Coil 在内存管理上做了大量工作，以避免 `OutOfMemoryError` 并保持应用流畅：
* **图像降采样 (Downsampling)：** Coil的第一次解码只读取图片的原始宽高（不分配内存）。根据原始宽高和目标 ImageView 的宽高，计算出最佳的inSampleSize采样率。再带着计算出的inSampleSize进行第二次解码，将缩小的 Bitmap 加载到内存。
* **Bitmap 池化 (Bitmap Pooling)：** Coil 会复用 Bitmap 对象。当一个 Bitmap 不再需要显示时，它会被放回一个“池”中，而不是立即被垃圾回收 (GC)。当需要加载新图片时，Coil 会尝试从池中获取一个可复用的 Bitmap，而不是重新分配内存。这大大减少了 GC 的频率，从而减少了 UI 卡顿。
* **内存缓存 (Memory Cache)：** 使用 `LruCache`（最近最少使用算法）在内存中快速缓存已经加载的图片。如果同一张图片被再次请求，Coil 会直接从内存中读取，实现即时加载。
* **自动大小调整 (Automatic Sizing for Compose)：** 在 Jetpack Compose 中使用时，`AsyncImage` 能够自动检测 Composable 的约束（大小），并请求一个最优尺寸的图片。

### 3. 网络与磁盘 I/O 优化
Coil 将网络请求和磁盘缓存完全委托给了 OkHttp，将文件I/O委托给了 Okio。

OkHttp 是目前 Android 上最高效、最主流的网络库，它内置了连接池、gzip压缩、HTTP/2 支持和强大的磁盘缓存系统。Okio 则提供了非常高效的缓冲I/O操作。Coil 无需“重复造轮子”，直接站在了巨人的肩膀上。

利用 OkHttp 的磁盘缓存，实现网络图片的持久化存储，下次请求时（即使应用重启）也能快速加载。新版本的 Coil 支持遵循服务器的 `Cache-Control` 响应头，实现更智能的网络缓存策略。
### 4. 请求与生命周期管理
Coil 自动与 `androidx.lifecycle` 库集成。它会观察 Activity、Fragment 或 Composable 的生命周期。当组件进入 `onStop` 或 `onDestroy` 状态时，Coil 会自动取消相关的图片加载请求，这避免了无效的计算、内存占用和潜在的内存泄漏。

如果短时间内有多个地方请求同一张图片（例如在 `RecyclerView` 中），Coil 只会发起一次加载任务，并将结果分发给所有请求方。

支持设置图片加载的优先级，确保关键图片（如屏幕内的图片）优先于非关键图片（如屏幕外的图片）加载。
### 5. 轻量级与现代 API
相比于 Glide 和 Fresco，Coil 的库体积和方法数都更小，有助于减小 APK 大小。其API 设计简洁易用，充分利用了 Kotlin 的语言特性（如扩展函数、lambda 等）。与 Glide 不同，Coil 不使用注解处理器，这可以轻微加快应用的编译速度。

总结来说，Coil 的最大优化在于它**全面拥抱了 Kotlin 协程和 OkHttp/Okio 这一现代化技术栈**，并在此基础上实现了一套高效的内存管理（降采样、Bitmap池化）和智能的请求生命周期控制。