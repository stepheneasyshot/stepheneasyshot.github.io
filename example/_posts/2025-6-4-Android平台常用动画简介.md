---
layout: post
description: > 
  本文介绍了Android平台上的补间动画，属性动画，帧动画，扩展的mp4，pag，lottie，kanzi，unity
image: 
  path: /assets/img/blog/blogs_android_anim_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_anim_cover.png
    960w:  /assets/img/blog/blogs_android_anim_cover.png
    480w:  /assets/img/blog/blogs_android_anim_cover.png
accent_image: /assets/img/blog/blogs_android_anim_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Android平台常用动画简介
Android 平台上公认的系统层主要的动画类型有以下三种：属性动画，补间动画，帧动画
## 系统原生
### 补间动画 (View Animation / Tween Animation)
`Android` 平台最早的动画系统之一，只能对 `View对象` 进行动画，也称为视图动画。

且只能对 View 的基本变换属性进行动画，包括：

* **透明度 (Alpha)**
* **旋转 (Rotate)**
* **缩放 (Scale)**
* **平移 (Translate)**

需要注意的是，补间动画 **改变的是绘制位置，而非View的实际位置。** 实际上并没有改变 View 的实际位置、大小等属性，它只是改变了 View 的绘制方式。这意味着即使 View 被动画移动了，它的点击事件响应区域仍在原来的位置。

对于简单的变换动画，补间动画的使用相对简单，可以通过 XML 或代码定义。适用于简单的视图变换，例如按钮点击后的放大缩小效果、图片渐入渐出效果等。对性能要求不高的简单 UI 动画。

#### **使用**
首先确认好动画的效果，是平移或是透明度等。在res文件等anim文件夹内，建立一个 `translate_anim.xml`

```xml
<!-- 平移动画 -->
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000"
    android:fromXDelta="0%"
    android:toXDelta="300%"
    android:interpolator="@android:anim/linear_interpolator"/>
```

在代码中读取这个anim对象，再调用view的 `startAnimation` 方法。

```kotlin
binding.apply {
    tvViewAnimation.setOnClickListener {
        val anim =
            AnimationUtils.loadAnimation(this@ViewAnimationActivity, R.anim.translate_anim)
        tvViewAnimation.startAnimation(anim)
    }
}
```

我们就可以看到这个 `textView` 会平移到其三倍宽度的地方， `XDelta` 属性即为这个 View 自身宽度的尺度。动画完结view的显示会回到原来的位置，如果想要停在目标位置，可以设置 `anim.fillAfter = true` 。

在运动过程中，点击View原来的位置，可以看到动画重新开始了，可以说明View的实际位置没有改变。

#### **原理**
Android 系统会不断调用 View 的 `onDraw()` 方法，在onDraw()方法中，动画会根据当前时间计算出View应该处于的变换状态，然后应用这些变换（通过Canvas的translate()、scale()、rotate()、alpha()等方法），最后绘制View的内容。不断的重绘就形成了动画效果。

而View的实际位置是在 `Layout` 布局阶段确定的，补间动画并没有改变View的上下左右边界，所以触摸事件的生效区域还在最初 布局 阶段确定的原位置。
### 属性动画 (Property Animation)
属性动画系统是 Android 3.0 (API level 11) 引入的，它允许你对 **任何对象的任何属性** 进行动画。这意味着你不仅可以动画视图的属性（如位置、大小、透明度、旋转），还可以动画自定义对象的属性，甚至那些不直接绘制到屏幕上的属性。

属性动画的核心是改变属性的值。你可以指定动画的起始值和结束值，系统会根据时间插值器 (TimeInterpolator) 计算中间值，并通过一个估值器 (TypeEvaluator) 将计算出的值应用到目标对象的属性上。

属性动画不局限于 View，可以对任何 Java 对象进行动画，只要该对象有一个 setter 方法来设置要动画的属性。

有丰富的 API 来控制动画的各个方面，包括：
* **时长 (Duration)：** 动画的总时间。
* **时间插值 (Time Interpolation)：** 控制动画的变化速度曲线（例如，加速、减速、先慢后快、回弹等）。
* **重复计数和行为 (Repeat Count and Behavior)：** 指定动画是否重复播放，以及重复的次数和方式（例如，正向播放、反向播放）。
* **动画集合 (Animator Sets)：** 将多个动画组合成一个集合，可以同时播放、按顺序播放或延迟播放。
* **帧刷新延迟 (Frame Refresh Delay)：** 指定动画帧的刷新频率。

#### **场景**
* 对视图的各种属性（如位置、大小、透明度、旋转、颜色、背景等）进行复杂、精细控制的动画。
** 对非视图对象的属性进行动画。
* 实现自定义的动画效果。
* Material Design 风格的动画，例如圆形揭露动画、共享元素过渡动画等。

#### **使用**
属性动画的使用相对复杂，需要创建一个 `Animator` 对象，并指定要动画的属性和目标值。然后，将这个 `Animator` 对象应用到目标对象上。

**针对View的属性动画**

```kotlin
tvPropAnimation.apply {
    setOnClickListener {
        val animator =
            ObjectAnimator.ofFloat(this, "translationX", 0f, this.width * 3f)
        animator.duration = 2000
        animator.interpolator = LinearInterpolator()
        animator.start()
    }
}
```

这里我们可以看到，我们创建了一个属性动画，这个动画会让这个 `textView` 从自身的X轴方向平移到自身宽度三倍的位置。和上面补间动画一样的效果。

**针对变量的属性动画**

使用ValueAnimator来创建属性动画，ValueAnimator会根据时间插值器计算出中间值，并通过一个估值器将计算出的值应用到目标对象的属性上。

```kotlin
tvVarPropAnimation.apply {
    setOnClickListener {
        val valueAnimator = ValueAnimator.ofInt(this.width, this.width * 3)
        valueAnimator.duration = 2000
        valueAnimator.interpolator = LinearInterpolator()
        valueAnimator.addUpdateListener { animation ->
            this.width = animation.animatedValue as Int
        }
        valueAnimator.start()
    }
}
```

这个动画会把TextView的宽度在2s内扩大3倍。
### 帧动画 (Frame Animation / Drawable Animation)
帧动画是指通过按顺序显示一系列 `Drawable` 资源来创建的动画，类似于电影胶片的原理。

由于需要加载多张图片，如果图片数量过多或分辨率过高，可能会占用较多的内存资源。通常在 XML 文件中定义，指定每个帧的 `Drawable` 资源和显示时长。

播放序列帧图片，例如加载动画、游戏中的角色跳跃动画、爆炸效果等。精确控制每一帧显示内容的场景。

#### **使用**
在 `drawable` 文件夹内，建立一个 `frame_anim.xml` 的文件。

```xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true">
    <item
        android:drawable="@drawable/fragnance_open_00"
        android:duration="50" />
    <item
        android:drawable="@drawable/fragnance_open_01"
        android:duration="50" />
    <item
        android:drawable="@drawable/fragnance_open_02"
        android:duration="50" />
    <item
        android:drawable="@drawable/fragnance_open_03"
        android:duration="50" />
</animation-list>
```

在代码中读取这个anim对象，一般会使用一个ImageView来承载，触发后，调用 ImageView 的 `setBackgroundResource` 方法。

```kotlin
binding.apply {
    ivFrameAnimation.setOnClickListener {
        ivFrameAnimation.setBackgroundResource(R.drawable.frame_anim)
        (ivFrameAnimation.background as AnimationDrawable).start()
    }
}   
```

#### **原理**
帧动画的原理是通过不断地切换 `Drawable` 资源来实现的。系统会根据每个 `Drawable` 的显示时长，依次显示每个 `Drawable`，形成动画效果。

### 小结
在现代 Android 开发中，**属性动画** 是推荐的首选动画方式，因为它提供了无与伦比的灵活性和强大的控制能力。视图动画和帧动画在某些特定场景下仍有其用武之地，但通常属性动画能够实现它们的功能，并且效果更好。

此外，对于 Jetpack Compose 这种声明式 UI 框架，也有其独特的动画 API，如 `animateFloatAsState`、`AnimatedVisibility` 等，它们在底层也是基于属性动画的原理实现的，提供了更简洁的动画开发体验。

## 三方扩展
### Lottie动画
Lottie 是一个由 Airbnb 开发的开源动画库，它的核心理念是将 AE 中创建的动画导出为轻量级的 JSON 文件，然后 Lottie 库可以在各种平台上（包括 Android, iOS, Web, React Native, Windows, HarmonyOS 等）原生渲染这些动画。

Lottie 动画的优势
* 文件体积小巧，Lottie 动画文件通常比 GIF 或 APNG 等传统位图动画格式小得多。这意味着更快的加载速度和更少的带宽消耗，尤其是在移动端体验上优势明显。
* 高保真还原，Lottie 能够几乎完美地还原 AE 中设计的动画效果，包括路径动画、形状变换、颜色渐变、蒙版、预合成等，解决了传统方式下设计师和开发者之间协作的难题（即“所见即所得”）。
* 矢量性，大部分 Lottie 动画是基于矢量图形的，这意味着它们在不同分辨率的屏幕上都能保持清晰锐利，不会出现像素化的问题。
* 跨平台兼容性，同一个 JSON 文件可以在多个平台上使用，大大提高了开发效率和动画的一致性。
* 易于集成，Lottie 提供了简洁易用的 API，开发者可以轻松地将 Lottie 动画集成到他们的应用中。

#### **原理**
**JSON 文件解析：**

* 在应用程序运行时，Lottie 库会读取这个 JSON 文件。
* 它会解析 JSON 数据，将其映射为 Lottie 内部的数据结构，代表动画中的各种元素和它们的动画属性。

**原生渲染**

* Lottie 库根据解析后的数据，利用各个平台原生的绘图 API 进行渲染。
* 在 Android 上，Lottie 主要通过 `Canvas` 进行 2D 渲染，并结合原生的 `ValueAnimator` 等动画组件来实现动画效果。
* 它会遍历动画的每一帧，计算当前帧每个图层和元素的变换（平移、缩放、旋转、透明度、路径变化等），然后使用 `Canvas` 的绘图方法（如 `drawPath`, `drawCircle`, `drawRect` 等）将其绘制出来。

#### **使用**
**添加依赖：** 

在 `build.gradle` (Module: app) 中添加 Lottie 库的依赖。

```gradle
dependencies {
    implementation 'com.airbnb.android:lottie:6.0.0' // 替换为最新版本
}
```
**放置 JSON 文件** 

将导出的 `.json` 动画文件放置在 `src/main/assets` 目录下。

**在布局中添加 `LottieAnimationView`：**
```xml
<com.airbnb.lottie.LottieAnimationView
    android:id="@+id/animation_view"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:lottie_fileName="your_animation.json"
    app:lottie_autoPlay="true"
    app:lottie_loop="true" />
```

> `app:lottie_fileName`：指定位于 `assets` 目录下的 JSON 文件名。
`app:lottie_autoPlay`：是否自动播放。
`app:lottie_loop`：是否循环播放。

**在代码中控制** 
你也可以通过代码来加载和控制动画。

```java
LottieAnimationView animationView = findViewById(R.id.animation_view);
// 从 assets 加载
animationView.setAnimation("your_animation.json");
// 从 raw 资源加载
// animationView.setAnimation(R.raw.your_animation);
// 从 URL 加载
// animationView.setAnimationFromUrl("https://example.com/your_animation.json");
animationView.playAnimation(); // 播放
animationView.pauseAnimation(); // 暂停
animationView.cancelAnimation(); // 取消
animationView.loop(true); // 设置循环
animationView.setSpeed(1.5f); // 设置播放速度
animationView.setProgress(0.5f); // 设置播放进度
// ... 更多控制 API
```

Lottie 是现代移动和 Web 应用中实现高质量动画的强大工具，极大地简化了设计师和开发者之间的协作流程，并提供了出色的性能和灵活性。
### PAG动画
**PAG动画** 是由腾讯开发的一种高效的动画渲染解决方案，全称为 **Portable Animated Graphics**。它主要用于在移动端、Web 和桌面应用中实现高性能的矢量动画播放，广泛应用于社交、广告、游戏和短视频等领域。

PAG动画的核心特点：

* 高性能渲染，采用 **硬件加速**（如OpenGL/Vulkan/Metal），确保动画流畅播放，即使在低端设备上也能保持高帧率。支持 **多线程解码**，减少主线程压力，提升响应速度。
* 全平台支持，覆盖 **iOS、Android、Web、Windows、macOS** 等平台，提供统一的动画体验。支持 **Flutter、React Native** 等跨平台框架。
* 矢量动画兼容性，支持从 **Adobe After Effects (AE)** 直接导出动画（通过PAG插件），保留图层、遮罩、特效等属性。兼容 **Lottie** 的部分特性，但性能更优，文件更小。
* 动态编辑能力，运行时可以修改文本、图片、颜色等元素（如替换广告文案或贴图），无需重新导出动画。支持 **视频模板**，方便替换视频片段。
* 文件体积小，采用二进制格式（`.pag` 文件），比JSON格式的Lottie（`.json`）更紧凑，加载更快。

#### **使用**
PAG动画的使用相对简单，主要分为以下几个步骤：

**导包**

```toml
tencent-libpag = { module = "com.tencent.tav:libpag", version.ref = "tencentLibpag" }
```

**放置源文件**

将设计师所提供的 `.pag动画` 文件导入到项目中，一般放置于assets文件夹。

**xml布局添加承载的pagView**

```xml
<com.tencent.libpag.PAGView
    android:id="@+id/pagView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

**代码中加载动画**

```kotlin
// 调用
playPAGView(pagView, "anim_file.pag")

//开始播放动效
fun playPAGView(pagView: PAGView?, fileName: String) {
    LogUtils.i(TAG, "playPAGView: fileName:$fileName")
    if (pagView != null) {
        if (pagView.isPlaying) {
            pagView.stop()
        }
        pagView.visibility = View.VISIBLE
        val pf = PAGFile.Load(appContext.assets, fileName)
        pagView.composition = pf
        pagView.setRepeatCount(-1)
        pagView.play()
    }
}
```

PAG动画凭借其高性能和灵活性，逐渐成为替代Lottie的主流方案，特别适合需要复杂动效和实时编辑的场景。

#### **原理**
整体架构如下：

```
┌───────────────────────┐
│      Android App      │
├───────────────────────┤
│  PAG SDK (Java/Kotlin)│
├───────────┬───────────┤
│ 文件解析层 │ 渲染引擎层 │
├───────────┴───────────┤
│      OpenGL ES/Skia   │
└───────────────────────┘
```

**(1) 文件解析与数据解码**

.pag 文件格式：采用二进制编码（而非 Lottie 的 JSON），体积更小，解析更快。解析时仅解码当前帧所需的数据，减少内存占用。采用了多线程解码，在后台线程解析动画数据，避免阻塞 UI 线程。

**(2) 渲染引擎（Skia + OpenGL ES）**

PAG 的渲染核心依赖：
Skia：处理矢量路径（Path）、颜色渐变、遮罩等 2D 图形操作。
OpenGL ES：实现硬件加速，通过 GPU 高效合成图层（类似游戏渲染）。

渲染流程：
> 图层树构建：解析 .pag 文件后生成图层树（类似 AE 的图层结构）。
帧数据计算：根据当前时间戳计算各图层的变换（位移、旋转、透明度等）。

GPU 绘制：
> 矢量路径 → 由 Skia 转换为 GPU 可识别的网格（Mesh）。
位图/图片 → 使用 OpenGL 纹理（Texture）渲染。
特效（模糊、渐变）→ 通过 GLSL 着色器实现。

**(3) 动画控制与动态修改**

时间轴驱动：基于 ValueAnimator 或 Choreographer 同步帧率（通常 60FPS）。

动态替换：
> 文本替换：运行时修改 PAGTextLayer 的内容。
图片替换：通过 PAGImageLayer 绑定 Bitmap 对象。
播放控制：支持播放、暂停、循环、进度跳转等。

### PAG和Lottie对比
简单来说，Lottie动画的文件简单小巧，文件主要基于JSON打包，跨平台特性比较好。

而PAG动画可实现的效果更多，同时文件采用了二进制压缩，二进制格式通常具有更高的压缩率和解析速度，但可读性较差，不易直接修改。

选择 Lottie 还是 PAG 取决于你的具体需求：

* 如果你主要制作轻量级、矢量化、UI 相关的动画，且对 After Effects 复杂特效没有特别要求，那么 Lottie 通常是更好的选择，它更简洁、轻便。
* 如果你需要实现After Effects 中更复杂、更炫酷的动画效果，可能包含视频、图片、大量文本动态替换，或者需要制作短视频模板等，那么 PAG 能提供更高的还原度和更强大的运行时编辑能力。

### kanzi && Unity 3D
这两个动画比较详细的介绍，此前已经有一篇记录。

[【Android集成Unity的两种方式】](./2024-12-31-Android集成Unity的两种方式.md)
