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
### 属性动画 (Property Animation)

* **特点：**
    * **最强大和灵活：** 属性动画系统是 Android 3.0 (API level 11) 引入的，它允许你对任何对象的任何属性进行动画。这意味着你不仅可以动画视图的属性（如位置、大小、透明度、旋转），还可以动画自定义对象的属性，甚至那些不直接绘制到屏幕上的属性。
    * **动画值：** 属性动画的核心是改变属性的值。你可以指定动画的起始值和结束值，系统会根据时间插值器 (TimeInterpolator) 计算中间值，并通过一个估值器 (TypeEvaluator) 将计算出的值应用到目标对象的属性上。
    * **独立于视图：** 属性动画不局限于 View，可以对任何 Java 对象进行动画，只要该对象有一个 setter 方法来设置要动画的属性。
    * **可控性高：** 提供丰富的 API 来控制动画的各个方面，包括：
        * **时长 (Duration)：** 动画的总时间。
        * **时间插值 (Time Interpolation)：** 控制动画的变化速度曲线（例如，加速、减速、先慢后快、回弹等）。
        * **重复计数和行为 (Repeat Count and Behavior)：** 指定动画是否重复播放，以及重复的次数和方式（例如，正向播放、反向播放）。
        * **动画集合 (Animator Sets)：** 将多个动画组合成一个集合，可以同时播放、按顺序播放或延迟播放。
        * **帧刷新延迟 (Frame Refresh Delay)：** 指定动画帧的刷新频率。
    * **主要类：** `ValueAnimator` (核心，用于计算动画值)、`ObjectAnimator` (继承自 `ValueAnimator`，可以直接对对象的属性进行动画)。

* **适用场景：**
    * 需要对视图的各种属性（如位置、大小、透明度、旋转、颜色、背景等）进行复杂、精细控制的动画。
    * 需要对非视图对象的属性进行动画。
    * 需要实现自定义的动画效果。
    * Material Design 风格的动画，例如圆形揭露动画、共享元素过渡动画等。

### 补间动画 (View Animation / Tween Animation)

* **特点：**
    * **历史悠久：** 这是 Android 最早的动画系统之一，也称为视图动画。
    * **局限于视图：** 只能对 View 对象进行动画，且只能对 View 的基本变换属性进行动画，包括：
        * **透明度 (Alpha)**
        * **旋转 (Rotate)**
        * **缩放 (Scale)**
        * **平移 (Translate)**
    * **改变的是绘制位置，而非实际位置：** 视图动画实际上并没有改变 View 的实际位置、大小等属性，它只是改变了 View 的绘制方式。这意味着即使 View 被动画移动了，它的点击事件响应区域仍在原来的位置。
    * **简单易用：** 对于简单的视图变换动画，视图动画的使用相对简单，可以通过 XML 或代码定义。

* **适用场景：**
    * 简单的视图变换，例如按钮点击后的放大缩小效果、图片渐入渐出效果等。
    * 对性能要求不高的简单 UI 动画。

### 帧动画 (Frame Animation / Drawable Animation)

* **特点：**
    * **序列图像播放：** 帧动画通过按顺序显示一系列 `Drawable` 资源来创建动画，类似于电影胶片。
    * **占用资源较多：** 由于需要加载多张图片，如果图片数量过多或分辨率过高，可能会占用较多的内存资源。
    * **主要类：** `AnimationDrawable`。
    * **定义方式：** 通常在 XML 文件中定义，指定每个帧的 `Drawable` 资源和显示时长。

* **适用场景：**
    * 播放序列帧图片，例如加载动画、角色跳跃动画、爆炸效果等。
    * 需要精确控制每一帧显示内容的场景。

### 总结比较：

| 动画类型       | 属性动画 (Property Animation)                                | 视图动画 (View Animation)                                     | 帧动画 (Frame Animation)                                     |
| :------------- | :----------------------------------------------------------- | :------------------------------------------------------------ | :----------------------------------------------------------- |
| **控制对象** | 任何对象的任何属性                                           | View 的基本变换属性 (Alpha, Rotate, Scale, Translate)           | Drawable 资源序列                                            |
| **原理** | 改变属性的实际值                                             | 改变 View 的绘制方式，不改变实际属性                         | 顺序播放一系列图片                                           |
| **灵活性** | 非常高，可实现复杂动画和自定义动画                           | 较低，只能进行预定义的变换                                    | 较低，只能实现序列帧动画                                     |
| **性能** | 较好，系统会优化属性值的计算和应用                           | 较差，频繁重绘可能导致性能问题，点击事件区域滞留              | 取决于图片数量和大小，可能占用较多内存                       |
| **API 版本** | Android 3.0 (API level 11) 及以上引入                       | Android 1.0 开始支持                                        | Android 1.0 开始支持                                        |
| **使用场景** | 复杂 UI 动画、自定义动画、Material Design 动画、对非 View 对象动画 | 简单的视图变换、快速实现过渡效果                              | 播放连续图片，如加载动画、序列帧动画                          |
| **主要类/XML** | `ValueAnimator`, `ObjectAnimator`                            | `<alpha>`, `<rotate>`, `<scale>`, `<translate>`, `<set>` (在 `res/anim` 目录下) | `AnimationDrawable` (在 `res/drawable` 目录下)，`<animation-list>` |

在现代 Android 开发中，**属性动画** 是推荐的首选动画方式，因为它提供了无与伦比的灵活性和强大的控制能力。视图动画和帧动画在某些特定场景下仍有其用武之地，但通常属性动画能够实现它们的功能，并且效果更好。

此外，对于 Jetpack Compose 这种声明式 UI 框架，也有其独特的动画 API，如 `animateFloatAsState`、`AnimatedVisibility` 等，它们在底层也是基于属性动画的原理实现的，提供了更简洁的动画开发体验。

## 三方扩展
### mp4动画
### pag动画

### lottie动画

### kanzi

### Unity 3D
