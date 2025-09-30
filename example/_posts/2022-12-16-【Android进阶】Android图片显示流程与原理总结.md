---
layout: post
description: > 
  本文介绍了Android平台的图片显示的流程与原理，以及优秀第三方库是如何优化的
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
# 【Android进阶】Android图片显示流程与原理总结
最近碰到一个切换主题速度过慢的性能问题，涉及到图片编解码和显示。在解决问题的过程中了解了一些之前没有注意到的细节，干脆对图片加载与显示整体流程做一次学习总结。
### View 绘制的整体流程
在 Android 中，View 的绘制是通过 ​​View#draw(Canvas canvas)​​ 方法完成的，这个方法内部会依次执行以下几个步骤：
* ​​绘制背景（drawBackground）​​
* ​​如果需要，保存图层（saveLayer，用于透明度、阴影等）​​
* ​​绘制 View 的内容（onDraw）​​
* ​​绘制子 View（dispatchDraw）​​
* ​​绘制装饰（如滚动条、前景等）​​

其中，​​绘制背景是第一步​​，也就是 drawBackground(canvas)方法。
## 设置一个图片有哪些写法
### View.background
View.background 是所有控件共有的一个属性，它是一个 Drawable 对象，用于设置 View 的背景。

你可以通过以下方式设置背景：

```kotlin
view.background = ColorDrawable(Color.RED) // 设置颜色背景
view.background = ContextCompat.getDrawable(context, R.drawable.my_drawable) // 设置资源背景
```

### ImageView
最常见的当属使用 `ImageView` 加载 `drawable` 文件夹下的一个固定资源。

静态设置：

```html
<ImageView
    android:id="@+id/myImageView"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:src="@drawable/my_local_image"
    android:contentDescription="我的本地图片" />
```

动态替换：

```kotlin
val imageView = findViewById<ImageView>(R.id.myImageView)
imageView.setImageResource(R.drawable.my_local_image) // 直接设置资源ID
// 或者通过Drawable对象
val drawable = ContextCompat.getDrawable(this, R.drawable.my_local_image)
imageView.setImageDrawable(drawable)
```

其他的设置方法还有：

```kotlin
val imageView = findViewById<ImageView>(R.id.myImageView)

imageView.setImageBitmap(bitmap) // 设置Bitmap
imageView.setImageResource(R.drawable.my_local_image) // 设置资源ID
imageView.setImageDrawable(drawable) // 设置Drawable对象
imageView.setImageURI(uri) // 设置URI
...
```

多使用Ctrl+点击追踪几步，我们就可以看到不管是设置Background还是ImageView的各个设置方法，最终转换的都是一个Drawable对象。这个Drawable对象究竟是何方神圣？

## Drawable对象
光听名字就可以推断其作用， `Drawable` 是一个抽象基类，它代表了 **“可以被绘制”** 的东西。可以理解为所有图形的**通用容器**，它不直接包含像素数据，而是定义了如何绘制自己。

很多不同类型的图形都是它的子类，比如图片、颜色、形状、图层列表等。Drawable可以被绘制到 Canvas 上，或者作为 View 的背景。同时，因为它不直接存储像素数据，所以通常比 Bitmap 更轻量。

**常见子类：**

* **ColorDrawable：** 代表一种纯色。
* **ShapeDrawable：** 可以绘制圆形、矩形等基本形状。
* **LayerDrawable：** 可以将多个 Drawable 堆叠在一起。
* **TransitionDrawable：** 可以实现两个 Drawable 之间的渐变过渡。
* **BitmapDrawable：** 后面会介绍，它是 Drawable 的一个特殊子类，用于包装 Bitmap。

同样常见的还有一个Bitmap类。
### **Bitmap**
**Bitmap** 代表了一个**位图图像**，实际上，它就是一段**像素数据**。当你加载一张图片（比如 JPG, PNG）到内存中时，最终会解码成一个 Bitmap 对象。

**Bitmap** 直接包含图像的像素信息（颜色、透明度等），存储了完整的像素数据，因此通常是内存消耗的大户。高分辨率的图片会生成更大的 Bitmap，占用更多内存。

其内存占用的大小，取决于图像的**宽度、高度以及每个像素占用的字节数**（颜色深度，如 ARGB_8888）。

Bitmap 对象可以直接通过 `Canvas.drawBitmap()` 方法绘制到画布上。

一般当我们需要对图片进行**像素级别的操作**时（比如修改像素、裁剪、缩放），或者需要将图片加载到内存中以供显示时，就会用到 Bitmap。
### **BitmapDrawable**
**BitmapDrawable** 听名字有点混乱，它是 **Drawable** 的一个子类，它的作用是**封装一个 Bitmap 对象，并使其具备 Drawable 的特性**。

它继承了 Drawable 的所有特性，因此可以像其他 Drawable 一样使用（例如作为 View 的背景）。核心就是持有并管理一个 Bitmap 对象。

BitmapDrawable 提供了额外的绘制控制，比如可以设置平铺模式 (tile mode)、抗锯齿 (anti-aliasing) 等。Android 系统在处理图片资源时，常常会将图片解码成 Bitmap，然后封装成 BitmapDrawable 来使用。

当你有一张图片 (Bitmap) 并希望将其作为 UI 元素（如 View 的背景、ImageView 的 `src`）来显示，并且可能需要一些 Drawable 提供的额外功能（如拉伸、平铺）时，BitmapDrawable 就非常有用。

**三者的关系**可以总结为：
* **Bitmap** 是原始的图片数据；
* **Drawable** 是一个接口或者说一个规范，定义了“可绘制”的通用行为；
* **BitmapDrawable** 是一个具体的实现，它把 **Bitmap** 包装起来，让这个原始数据也能符合 **Drawable** 的规范，从而可以在 Android 的 UI 框架中方便地使用和绘制。

当我们从资源文件 (`res/drawable`) 中加载图片时，系统通常会返回一个 **BitmapDrawable** 对象。如果需要对图片进行复杂的处理（比如滤镜、合成），你可能会先获取到 **Bitmap**，处理完后再将其包装成 **BitmapDrawable** 或直接绘制。在布局文件中使用 `android:src` 或 `android:background` 引用图片时，背后使用的其实就是 **BitmapDrawable**。

## Drawable对象定义了哪些行为
`Drawable` 对象定义了“可绘制”的抽象概念，它提供了许多基础能力，让各种视觉元素都能以统一的方式进行处理和显示。主要定义了以下核心行为：

> 绘制自身 (`draw(Canvas canvas)`)，这是 `Drawable` 最核心的行为。所有 `Drawable` 子类都必须实现这个方法，在这个方法里定义了如何将自身绘制到给定的 `Canvas`（画布）上。`Canvas` 提供了各种绘制方法，比如画线、画矩形、画位图等。

> 设置边界 (`setBounds(int left, int top, int right, int bottom)` / `setBounds(Rect bounds)`)，`Drawable` 可以通过设置边界来定义它应该被绘制在 `Canvas` 上的哪个区域。这个边界决定了 `Drawable` 的大小和位置。很多 `Drawable`（例如 `BitmapDrawable`）会根据这些边界来缩放或平铺自己的内容。

> 获取固有尺寸 (`getIntrinsicWidth()` / `getIntrinsicHeight()`)，有些 `Drawable` 拥有固有的尺寸，比如一个 `BitmapDrawable` 包装的位图图片。通过这两个方法，可以获取到 `Drawable` 建议的宽度和高度。这些尺寸常常被 `View` 用来计算其自身的默认大小。

> 设置透明度 (`setAlpha(int alpha)`)，你可以设置 `Drawable` 的整体透明度，范围从 0（完全透明）到 255（完全不透明）。这个方法会影响 `Drawable` 绘制时的不透明度。

> 管理回调 (`setCallback(Callback cb)`)，`Drawable` 可以注册一个回调接口 (`Drawable.Callback`)，当 `Drawable` 自身的状态发生改变时（比如需要重新绘制、边界改变），可以通过这个回调通知宿主 `View` 进行相应的操作。

## 如何获取或者创建一个Drawable对象
了解了大概的 Drawable 定义的行为，可以来看看如何获取或者创建一个Drawable对象。
### 1\. 从资源文件中获取 Drawable
这是最常用也是推荐的方法，特别是当你需要将 Drawable 作为应用资源管理时。

例如 **`Context.getDrawable(int id)` 或 `Resources.getDrawable(int id)`**，这是最直接的方式。`Context` 类（例如 `Activity` 或 `Application` 对象）提供了 `getDrawable()` 方法，它会使用与该 `Context` 关联的 `Resources` 对象来加载 Drawable。

使用这种方式来获取，资源集中管理，方便多语言、多密度适配，自动处理主题和缓存。
### 2\. 通过代码创建 Drawable
当你的 Drawable 形状或颜色需要动态生成，或者它们不适合作为静态资源时，你可以通过代码动态地来创建。

  * **`ShapeDrawable`**
    用于绘制简单的几何形状，如矩形、圆形、椭圆等。你可以动态设置其颜色、边框、圆角等。

    ```java
    ShapeDrawable shapeDrawable = new ShapeDrawable(new OvalShape());
    shapeDrawable.getPaint().setColor(Color.BLUE);
    shapeDrawable.setBounds(0, 0, 100, 100); // 设置 Drawable 的边界
    ```

  * **`GradientDrawable`**
    比 `ShapeDrawable` 更强大，可以创建渐变效果、实心颜色、边框、圆角等，通常在 XML 中定义。但也可以在代码中创建和配置。

    ```java
    GradientDrawable gradientDrawable = new GradientDrawable();
    gradientDrawable.setShape(GradientDrawable.RECTANGLE); // 矩形
    gradientDrawable.setCornerRadius(8f); // 圆角
    gradientDrawable.setColor(Color.RED); // 实心颜色
    gradientDrawable.setStroke(2, Color.BLACK); // 边框
    // 设置渐变
    // gradientDrawable.setGradientType(GradientDrawable.LINEAR_GRADIENT);
    // gradientDrawable.setColors(new int[]{Color.YELLOW, Color.GREEN});
    ```

  * **`ColorDrawable`**
    最简单的 Drawable，只显示一种纯色。

    ```java
    ColorDrawable colorDrawable = new ColorDrawable(Color.parseColor("#FF4081"));
    ```

  * **`BitmapDrawable`**
    用于将 `Bitmap` 对象包装成 `Drawable`。当你通过网络或其他方式获取到图片数据（通常是 `Bitmap` 格式）并希望将其用作 Drawable 时很有用。

    ```java
    // 假设你已经有一个 Bitmap 对象
    Bitmap myBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.my_image);
    BitmapDrawable bitmapDrawable = new BitmapDrawable(getResources(), myBitmap);
    ```

  * **`LayerDrawable`**
    将多个 Drawable 堆叠在一起，形成一个 Drawable 层。

    ```java
    Drawable[] layers = new Drawable[2];
    layers[0] = new ColorDrawable(Color.GRAY);
    layers[1] = ContextCompat.getDrawable(this, R.drawable.my_icon); // 假设有一个图标
    LayerDrawable layerDrawable = new LayerDrawable(layers);
    // 可以设置每个层的偏移量等
    // layerDrawable.setLayerInset(1, 10, 10, 10, 10);
    ```

  * **`StateListDrawable`**
    根据 View 的不同状态（如按下、选中、聚焦等）显示不同的 Drawable。通常在 XML 中定义。

    ```java
    // 这是一个简化示例，通常在代码中创建会比较复杂，更推荐XML
    StateListDrawable stateListDrawable = new StateListDrawable();
    // 添加普通状态
    stateListDrawable.addState(new int[]{}, new ColorDrawable(Color.BLUE));
    // 添加按下状态
    stateListDrawable.addState(new int[]{android.R.attr.state_pressed}, new ColorDrawable(Color.RED));
    ```

  * **自定义 `Drawable`**
    如果上述内置 Drawable 无法满足你的需求，你可以继承 `Drawable` 类并重写 `draw()` 方法来绘制你想要的任何内容。这是最灵活但也是最复杂的方法。

    ```java
    public class CustomDrawable extends Drawable {
        // ... 实现 draw(), setAlpha(), setColorFilter(), getOpacity() 等方法
        @Override
        public void draw(@NonNull Canvas canvas) {
            // 在这里进行自定义绘制
        }
        // ...
    }
    ```

### 3\. 从 URI 或文件路径获取 Drawable
当我们需要加载来自文件系统、网络或内容提供者的图片时。

可以使用 **`Drawable.createFromPath(String pathName)`** ，从给定的文件路径创建 Drawable。

```java
String imagePath = "/sdcard/my_pictures/some_image.png";
Drawable drawableFromFile = Drawable.createFromPath(imagePath);
```

另外还可以使用 **`Drawable.createFromStream(InputStream is, String srcName)`**，从 `InputStream` 创建 Drawable，常用于从网络下载图片后。

## 图像是如何加载到界面上的
以 `ImageView.setImageResource(@DrawableRes int resId)` 为例，看看这里走了哪些过程。

