---
layout: post
description: > 
  本文介绍了 View 框架下，自定义复杂View的一些路线
image: 
  path: /assets/img/blog/blogs_android_common_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_common_cover.png
    960w:  /assets/img/blog/blogs_android_common_cover.png
    480w:  /assets/img/blog/blogs_android_common_cover.png
accent_image: /assets/img/blog/blogs_android_common_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】Android自定义View
在 Android 开发过程中，我们经常使用官方提供的 TextView、Button 和 RecyclerView 来构建界面。但当面临复杂、个性化或高性能的 UI 需求时，这些标准组件往往力不从心。这时，自定义 View 就显得很必要了。

自定义 View 不仅仅是简单地重写 onDraw()，它更关乎 Android 系统的测量、布局、性能优化，以及触摸事件的精准处理。

## 继承自ViewGroup
这种方法主要是界面里有**多处出现的相同的公共控件**，将多个基础子View组合到一起，一起集中至一个 `ViewGroup` 中。自定义View类往往直接继承自某个特定的ViewGroup（比如LinearLayout），往里面添加可复用的子View：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:background="@drawable/bg_button"
    android:orientation="vertical"
    android:padding="10dp">

    <ImageView
        android:id="@+id/iv_count"
        android:layout_width="100dp"
        android:layout_height="200dp"
        android:layout_gravity="center"
        android:background="@drawable/testbg"
        android:textSize="24sp" />


    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:gravity="center"
        android:text="首页卡片"
        android:textColor="@color/white"
        android:textSize="20sp" />

</LinearLayout>
```

在Java或者Kotlin代码中使用这些控件，指定响应交互的方式。

```kotlin
class MainCardView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : LinearLayout(context, attrs, defStyleAttr) {

    // 加载布局文件
    private val binding: LayoutMainCardViewBinding =
        LayoutMainCardViewBinding.inflate(LayoutInflater.from(context), this, true)
    
}
```

效果截图：

![](/assets/img/blog/blogs_custom_view_card.png){:width="200" height="400" loading="lazy"}

## 直接继承自View
### 基础绘制类和方法
#### Paint
`Paint` 对象，顾名思义是画笔，定义“用什么”来画，决定了你绘制的**颜色、样式、粗细、字体**等所有外观特性。

| 核心概念 | 作用描述 | 常用方法（部分） |
| :--- | :--- | :--- |
| **颜色** | 设置绘制的颜色。 | `setColor(int color)` |
| **样式 (Style)** | 决定绘制是描边、填充还是两者都有。 | `setStyle(Paint.Style style)`<br> - `FILL` (填充)<br> - `STROKE` (描边/空心)<br> - `FILL_AND_STROKE` (填充加描边) |
| **线宽** | 设置线条或描边（`STROKE` 模式）的粗细。 | `setStrokeWidth(float width)` |
| **抗锯齿** | 让边缘更平滑，图形更美观。 | `setAntiAlias(boolean aa)` |
| **着色器 (Shader)** | 实现渐变、纹理等复杂色彩效果。 | `setShader(Shader shader)` (如 `LinearGradient`) |
| **文本相关** | 设置字体、字号、对齐方式。 | `setTextSize(float size)`, `setTypeface(Typeface typeface)` |
| **路径效果** | 设置路径的绘制效果（如虚线、点状线）。 | `setPathEffect(PathEffect effect)` (如 `DashPath`) |

常见于初始化操作：

```java
textPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
textPaint.setTextAlign(Paint.Align.CENTER);
textPaint.setColor(Color.BLACK);
```

每次调用 `Canvas` 的 `draw...` 方法时，都需要传入一个配置好的 `Paint` 对象。

例如绘制文字：

```java
 public void drawText(@NonNull String text, float x, float y, @NonNull Paint paint) {
    super.drawText(text, x, y, paint);
}
```

#### Canvas 
`Canvas` 是一个画布，定义“在哪里”画，它提供了所有**绘制图形、文本、位图**的方法。所有绘制操作都是基于 `Canvas` 上的坐标系进行的。

| 核心概念 | 作用描述 | 常用方法（部分） |
| :--- | :--- | :--- |
| **绘制方法** | 用于画出各种图形。 | `drawLine()`, `drawCircle()`, `drawRect()`, `drawPath()` 等 |
| **坐标系** | 以 View 的左上角为原点 $(0, 0)$，向右为 $x$ 轴正方向，向下为 $y$ 轴正方向。 | |
| **变换 (Matrix)** | 对画布进行平移、旋转、缩放等操作，影响后续所有绘制。 | `translate()`, `rotate()`, `scale()` |
| **图层管理** | 允许创建新的绘图层，用于复杂的叠加和混合效果。 | `save()`, `restore()` |
| **裁剪 (Clip)** | 限制绘制的区域，超出裁剪区域的内容将不会显示。 | `clipRect()`, `clipPath()` |

`Canvas` 提供的 `draw` 方法只负责“画出”这个动作和位置，至于“画成什么样子”，则完全依赖于传入的 `Paint`。
#### 常用几何图形类
这些类用于**存储**几何图形的尺寸和形状信息，作为 `Canvas` 绘制方法的参数。
##### 1\. Rect / RectF（矩形）
* **`Rect`**，存储整数坐标的矩形。适用于需要像素对齐的场景。 
* **`RectF`** ，存储浮点数坐标的矩形。在大多数绘图场景中更常用，因为浮点数精度更高。

**它们都存储四个坐标值：**
* `left`: 左边界 x 坐标
* `top`: 上边界 y 坐标
* `right`: 右边界 x 坐标
* `bottom`: 下边界 y 坐标

例如：

```java
public class CustomRectView extends View {
    private Paint mRectPaint; 
    private Rect mDrawingRect;

    ...

    private void init() {
        mRectPaint = new Paint();
        mRectPaint.setColor(Color.BLUE); 
        mRectPaint.setStyle(Paint.Style.FILL); 
        mRectPaint.setAntiAlias(true);   
        // 初始化 Rect 对象。注意：这里的坐标通常是临时值或0，
        // 实际的边界值通常在 onSizeChanged 或 onDraw 中根据 View 尺寸设置。
        mDrawingRect = new Rect();
    }

    // 推荐在尺寸确定后设置 Rect 的边界
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        // 设置矩形的边界坐标：
        // 距离左边 50 像素，距离顶部 50 像素，宽度到 View 宽度减 50，高度到 View 高度减 50
        int left = 50;
        int top = 50;
        int right = w - 50;
        int bottom = h - 50;
        // 利用 Rect 对象的 set() 方法设置四个边界坐标
        mDrawingRect.set(left, top, right, bottom);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        // 3. 利用 Canvas 的 drawRect 方法和 Rect 对象进行绘制
        // drawRect(Rect r, Paint paint) 是 Canvas 专门用于绘制 Rect 区域的方法
        canvas.drawRect(mDrawingRect, mRectPaint);

        // 提示：也可以直接使用四个坐标值绘制，但使用 Rect 更便于管理和传递区域信息：
        // canvas.drawRect(50, 50, getWidth() - 50, getHeight() - 50, mRectPaint); 
    }
}
```

##### 2\. Path（路径）
**`Path`**用于定义**任意复杂的几何图形**，由直线、曲线等线段组成。
* `moveTo(x, y)`: 设置起点。
* `lineTo(x, y)`: 画直线到指定点。
* `quadTo()` / `cubicTo()`: 画二次/三次贝塞尔曲线
* `addCircle()` / `addRect()`: 直接添加圆形/矩形。
* `close()`: 闭合路径，将终点连接回起点。

可以绘制心形、五角星等复杂或不规则的图形，平滑的曲线或折线图。

例如利用 Path 绘制一个五角星：

```java
private void calculateStarPath(float centerX, float centerY) {
    mStarPath.reset();
    float currentAngle = (float) (Math.PI / 2.0); // 90度，从上方顶点开始
    // 五角星总共有 5 个外顶点和 5 个内陷点，共 10 个点
    for (int i = 0; i < 10; i++) {
        float radius = (i % 2 == 0) ? mOuterRadius : mInnerRadius;
        // 计算当前点的 X 坐标：centerX + radius * cos(angle)
        float x = (float) (centerX + radius * Math.cos(currentAngle));
        // 计算当前点的 Y 坐标：centerY - radius * sin(angle)。注意 Y 轴方向
        float y = (float) (centerY - radius * Math.sin(currentAngle));
        if (i == 0) {
            // 第一个点作为起始点
            mStarPath.moveTo(x, y);
        } else {
            // 连接到下一个点
            mStarPath.lineTo(x, y);
        }
        // 角度递增：每次增加 36 度（两个相邻顶点/内陷点间的角度）
        currentAngle -= ANGLE_STEP;
    }
    // 闭合路径（虽然 Path 默认会闭合，但最好显式调用）
    mStarPath.close();
}

@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    canvas.drawPath(mStarPath, mPaint);
}
```

#### 常用绘制方法详解
##### 1\. 绘制点和线

| 方法 | 签名 | 作用 |
| :--- | :--- | :--- |
| **`drawPoint()`** | `drawPoint(float x, float y, Paint paint)` | 在指定坐标 $(x, y)$ 绘制一个点。点的形状和大小受 `Paint` 的 **`setStrokeCap()`** (例如设置为圆形) 和 **`setStrokeWidth()`** 影响。 |
| **`drawLine()`** | `drawLine(float startX, float startY, float stopX, float stopY, Paint paint)` | 绘制一条直线，从 $(startX, startY)$ 到 $(stopX, stopY)$。线的颜色和粗细由 `Paint` 决定。 |
| **`drawLines()`** | `drawLines(float[] pts, Paint paint)` | 批量绘制多条线段。数组 `pts` 中的元素是 $[x_0, y_0, x_1, y_1, x_2, y_2, \dots]$，每四个坐标定义一条线。 |

##### 2\. 绘制圆形

| 方法 | 签名 | 作用 |
| :--- | :--- | :--- |
| **`drawCircle()`** | `drawCircle(float cx, float cy, float radius, Paint paint)` | 在中心点 $(cx, cy)$ 处，绘制一个指定半径的圆形。 |

##### 3\. 绘制矩形和椭圆

| 方法 | 签名 | 作用 |
| :--- | :--- | :--- |
| **`drawRect()`** | `drawRect(RectF rect, Paint paint)` <br>或 `drawRect(float left, float top, float right, float bottom, Paint paint)` | 绘制一个矩形。最常用的方式是传入一个 `RectF` 对象。 |
| **`drawOval()`** | `drawOval(RectF oval, Paint paint)` | 绘制一个椭圆。这个椭圆会**内切**于传入的 `RectF` 或矩形边界。如果传入的 `RectF` 是一个正方形，则会画出圆形。 |
| **`drawRoundRect()`** | `drawRoundRect(RectF rect, float rx, float ry, Paint paint)` | 绘制一个圆角矩形。`rx` 和 `ry` 分别指定圆角在 $x$ 轴和 $y$ 轴上的半径。 |

##### 4\. 绘制路径

| 方法 | 签名 | 作用 |
| :--- | :--- | :--- |
| **`drawPath()`** | `drawPath(Path path, Paint paint)` | 绘制一个由 `Path` 对象定义的复杂图形。这是绘制不规则图形和贝塞尔曲线的唯一方式。 |

### 重要回调方法
#### **`onMeasure(int widthMeasureSpec, int heightMeasureSpec)`** 
决定自身期望的尺寸，根据父容器传递的测量规格 (`MeasureSpec`) 计算并设置 `View` 最终的宽度和高度，通过调用 `setMeasuredDimension(width, height)` 来确定。必须实现，确保 `View` 能够正确地被布局，特别是当尺寸为 `wrap_content` 时。
#### **`onLayout(boolean changed, int left, int top, int right, int bottom)`** 
决定子 `View` 的位置，对于 `ViewGroup` 来说，它用于遍历所有子 `View` 并调用它们的 `layout()` 方法来定位子 `View`。对于单个 `View` 来说，这个方法通常不需要重写（除非你需要特殊的布局逻辑）。主要用于自定义 `ViewGroup`，用于定位子元素。自定义 `View` 通常不用重写。
#### **`onSizeChanged(int w, int h, int oldw, int oldh)`** 
在 `View` 的尺寸确定或发生变化时回调。它提供了 `View` 最终确定的尺寸，适合在这里进行与尺寸相关的初始化工作，如计算绘制坐标、创建 `Bitmap` 等。适合作为绘图对象的初始化场所。首次布局和尺寸变化时会回调。
#### **`onDraw(Canvas canvas)`** 
执行实际的绘制操作。onDraw 接收一个 `Canvas` 对象，可以在其上绘制图形、文本、图片等所有可见内容。所有自定义 `View` 的外观都是在这里完成的。所有自定义绘图代码都放在这里。注意要**避免**在这里执行耗时操作和对象创建。
#### **`onTouchEvent(MotionEvent event)`** 
处理用户的触摸事件，用于接收和响应用户的点击、滑动等输入。你需要在这里通过判断 `MotionEvent.getAction()` 来处理不同的手势。属于核心交互方法，返回 `true` 表示你已处理事件，并且希望继续接收后续事件。
#### **`computeScroll()`** 
配合 `Scroller` 实现平滑滚动，这个方法在 `View` 重绘时被调用，用于查询 `Scroller` 的当前位置，更新 `View` 的 `mScrollX/mScrollY`，并通过 `postInvalidate()` 驱动连续动画。实现平滑滚动（例如 `fling` 或平移）时必须重写。
#### 总结流程
`View` 的绘制流程可以简化为以下顺序（通常由系统驱动）：
1.  **构造方法**：初始化 `Paint`、属性等不依赖尺寸的对象。
2.  **`onMeasure()`**：决定 `View` 的期望尺寸。
3.  **`onSizeChanged()`**：获取最终尺寸，初始化依赖尺寸的对象。
4.  **`onDraw()`**：绘制 `View` 的外观。
5.  **`onTouchEvent()`**：处理用户交互。
6.  **`computeScroll()`**：如果在交互中触发了 `Scroller`，这个方法会驱动滚动动画。

### 静态View
这一类往往是无动态变化和交互的，一般直接在渲染绘制阶段使用特殊的手段给其加上特定的显示效果。

例如自定义一个可以渐变颜色的TextView：

```java
public class GradientTextView extends View {
    private Paint paint;
    private String text;
    private float textSize;

    public GradientTextView(Context context) {
        super(context);
        init();
    }

    public GradientTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public GradientTextView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        paint = new Paint();
        text = "Hello, World!";
        // 使用 dp/sp 转换尺寸更好，这里暂时保持 60f
        textSize = 60f;
        paint.setTextSize(textSize);
        // 抗锯齿
        paint.setAntiAlias(true);
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        if (w > 0) {
            // 每次尺寸变化时，根据最新的宽度 w 重新创建 LinearGradient
            // 渐变从 (0, 0) 到 (w, 0)
            Shader textShader = new LinearGradient(0, 0, w, 0,
                    Color.RED, Color.BLUE, Shader.TileMode.CLAMP);
            // 将新的 Shader 设置给 Paint
            paint.setShader(textShader);
            // 确保重新绘制
            invalidate();
        }
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 1. 测量文本宽度，使用 paint.measureText(text) 获取文本的精确宽度
        float desiredWidth = paint.measureText(text);
        // 2. 测量文本高度，使用 Paint.FontMetrics 来准确计算文本高度，包括上行空间(ascent)和下行空间(descent)
        Paint.FontMetrics fontMetrics = paint.getFontMetrics();
        // 文本高度 = descent - ascent
        float desiredHeight = fontMetrics.bottom - fontMetrics.top;
        // 3. 处理 View 的尺寸规格 (MeasureSpec) 根据父布局的限制和我们计算出的期望尺寸来确定最终的尺寸
        // 确定最终宽度
        int width = resolveSize((int) desiredWidth, widthMeasureSpec);
        // 确定最终高度
        int height = resolveSize((int) desiredHeight, heightMeasureSpec);
        // 4. 设置最终测量尺寸
        setMeasuredDimension(width, height);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        // 计算文本绘制的基线位置。
        // drawText 的 y 坐标是基线位置，而不是顶部。
        // 为了让文本垂直居中（假设 View 高度 h 足够），可以使用 (getHeight() - paint.ascent() - paint.descent()) / 2
        // 但为了简单，暂时保持原始逻辑：从 y=textSize 处开始绘制 (顶部)
        canvas.drawText(text, 0, textSize, paint);
    }
}
```

效果截图：

![](/assets/img/blog/blogs_custom_view_gradient_text.png)

可以看到代码中是创建了创建了一个从红色 (Color.RED) 到蓝色 (Color.BLUE) 的水平线性渐变，然后应用到paint画笔上：

```java
textShader = new LinearGradient(0, 0, getWidth(), 0, Color.RED, Color.BLUE, Shader.TileMode.CLAMP);
```

参数解释：

* `(0, 0)`: 渐变的起始点坐标（左上角）。
* `getWidth(), 0`: 渐变的结束点坐标（右边界，与起始点 y 坐标相同，因此是水平渐变）。
* `Color.RED`: 渐变的起始颜色。
* `Color.BLUE`: 渐变的结束颜色。
* `Shader.TileMode.CLAMP`: 瓷砖模式。CLAMP 的意思是如果渐变区域不够覆盖绘制区域，则在渐变的起止颜色之外的区域，使用渐变的起始色和结束色来延伸填充（在这个例子中，超出部分会被最接近的红色或蓝色填充）。

注意我们是依靠宽度来设置线性简便效果的，还需要重写 `onMeasure()` 方法，实现自适应的效果，以正确显示渐变。

### onMeasure 实现自适应尺寸
#### 包裹内容
上一个例子中，根据文字内容的宽高来重写了onMeasure方法，以达到包裹文字的效果。

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 1. 测量文本宽度，使用 paint.measureText(text) 获取文本的精确宽度
    float desiredWidth = paint.measureText(text);
    // 2. 测量文本高度，使用 Paint.FontMetrics 来准确计算文本高度，包括上行空间(ascent)和下行空间(descent)
    Paint.FontMetrics fontMetrics = paint.getFontMetrics();
    // 文本高度 = descent - ascent
    float desiredHeight = fontMetrics.bottom - fontMetrics.top;
    // 3. 处理 View 的尺寸规格 (MeasureSpec) 根据父布局的限制和我们计算出的期望尺寸来确定最终的尺寸
    // 确定最终宽度
    int width = resolveSize((int) desiredWidth, widthMeasureSpec);
    // 确定最终高度
    int height = resolveSize((int) desiredHeight, heightMeasureSpec);
    // 4. 设置最终测量尺寸
    setMeasuredDimension(width, height);
}
```

#### 固定尺寸
如果是一个图标或者图片类型的自定义View，期望在设置 `wrap_content` 自适应时，直接用一个预设的固定值，这时候就需要动态判断 `measureSpec` 这两个参数了。

```kotlin
companion object {
    const val FIXED_WIDTH_DP: Float = 200f
    const val FIXED_HEIGHT_DP: Float = 200f
}

override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    // 1. 获取固定的像素值
    val fixedWidthPx: Int = (FIXED_WIDTH_DP).dp2px()
    val fixedHeightPx: Int = (FIXED_HEIGHT_DP).dp2px()
    // 2. 处理宽度
    val widthMode = MeasureSpec.getMode(widthMeasureSpec)
    val widthSize = MeasureSpec.getSize(widthMeasureSpec)
    val finalWidth: Int = when (widthMode) {
        MeasureSpec.EXACTLY -> {
            // 布局指定了确切尺寸 (match_parent 或固定值)
            widthSize
        }
        MeasureSpec.AT_MOST -> {
            // 布局是 wrap_content，我们在这里返回固定尺寸
            // 但不能超过父布局的限制
            min(fixedWidthPx, widthSize)
        }
        else -> {
            // UNSPCIFIED，通常用于 Adapter View 等，直接返回固定尺寸
            fixedWidthPx
        }
    }
    // 3. 处理高度 (逻辑与宽度相同)
    val heightMode = MeasureSpec.getMode(heightMeasureSpec)
    val heightSize = MeasureSpec.getSize(heightMeasureSpec)
    val finalHeight: Int = when (heightMode) {
        MeasureSpec.EXACTLY -> {
            heightSize
        }
        MeasureSpec.AT_MOST -> {
            // 布局是 wrap_content，返回固定尺寸
            min(fixedHeightPx, heightSize)
        }
        else -> {
            fixedHeightPx
        }
    }
    // 4. 设置最终测量尺寸
    setMeasuredDimension(finalWidth, finalHeight)
}
```

### 动态View
这种类型的实现方式是可以动态更新的View，比如具体某一个时间点，或者一个其他的什么条件，达成之后，动态地改变内部绘制的参数。

这里举一个时钟表盘的例子：

```kotlin
class ClockView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val calendar = Calendar.getInstance()

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        // 获取当前时间
        calendar.timeInMillis = System.currentTimeMillis()
        val hour = calendar.get(Calendar.HOUR)
        val minute = calendar.get(Calendar.MINUTE)
        val second = calendar.get(Calendar.SECOND)

        // 绘制表盘
        drawClockFace(canvas)

        // 绘制时针
        drawHand(canvas, hour.toFloat() * 5 + minute / 12f, 0.5f, Color.BLACK)

        // 绘制分针
        drawHand(canvas, minute.toFloat(), 0.7f, Color.BLACK)

        // 绘制秒针
        drawHand(canvas, second.toFloat(), 0.9f, Color.RED)

        // 每秒更新一次
        postInvalidateDelayed(1000)
    }

    private fun drawClockFace(canvas: Canvas) {
        val centerX = width / 2f
        val centerY = height / 2f
        val radius = width.coerceAtMost(height) / 2f * 0.8f

        // 绘制表盘背景
        paint.color = Color.WHITE
        paint.style = Paint.Style.FILL
        canvas.drawCircle(centerX, centerY, radius, paint)

        // 绘制表盘边框
        paint.color = Color.BLACK
        paint.style = Paint.Style.STROKE
        paint.strokeWidth = 5f
        canvas.drawCircle(centerX, centerY, radius, paint)

        // 绘制刻度
        paint.strokeWidth = 3f
        for (i in 0..59) {
            val angle = Math.PI * i / 30 - Math.PI / 2
            val startX = centerX + (radius - 20) * cos(angle)
            val startY = centerY + (radius - 20) * sin(angle)
            val stopX = centerX + radius * cos(angle)
            val stopY = centerY + radius * sin(angle)
            canvas.drawLine(
                startX.toFloat(),
                startY.toFloat(),
                stopX.toFloat(),
                stopY.toFloat(),
                paint
            )
        }

        // 绘制数字
        paint.textSize = 40f
        paint.textAlign = Paint.Align.CENTER
        for (i in 1..12) {
            val angle = Math.PI * i / 6 - Math.PI / 2
            val x = centerX + (radius - 60) * cos(angle)
            val y = centerY + (radius - 60) * sin(angle) + paint.textSize / 3
            canvas.drawText(i.toString(), x.toFloat(), y.toFloat(), paint)
        }
    }

    private fun drawHand(canvas: Canvas, position: Float, length: Float, color: Int) {
        val centerX = width / 2f
        val centerY = height / 2f
        val radius = width.coerceAtMost(height) / 2f * 0.8f

        paint.color = color
        paint.strokeWidth = 10f
        paint.strokeCap = Paint.Cap.ROUND

        val angle = Math.PI * position / 30 - Math.PI / 2
        val stopX = centerX + radius * length * cos(angle)
        val stopY = centerY + radius * length * sin(angle)
        canvas.drawLine(centerX, centerY, stopX.toFloat(), stopY.toFloat(), paint)
    }
}
```

> 这里为了简单实现，直接在onDraw里进行的重绘消息post，实际应该在外部调用，或者使用一个Handler来循环刷新。

运行截图：

![](/assets/img/blog/blogs_custom_view_clock.png)

### 交互式
这类也是需要处理动态刷新的自定义View，更进一步还结合了触摸事件的处理，即需要根据 `ACTION_MOVE` 事件或者封装的 `GestureDetector` 手势回调里，判断方向和距离，对内部的组件进行特定的移动，缩放等处理逻辑。

比如一个空调温度选择滑动列表：

```java
public class TemperaturePickerView extends View {

    private static final String TAG = "TemperaturePickerView";

    // 温度范围
    private static final int MIN_TEMPERATURE = 16;
    private static final int MAX_TEMPERATURE = 30;

    // 绘制相关
    private Paint textPaint;
    private Paint selectedLinePaint;

    // 尺寸相关
    private float itemHeight;
    private float centerY;
    private float textBaseline;

    // 滚动相关
    private Scroller scroller;
    private GestureDetector gestureDetector;
    private float currentScrollY = 0;
    private float maxScrollY;

    // 选中监听
    private OnTemperatureChangeListener listener;
    private int currentTemperature = 20;

    // 字体大小配置
    private float maxTextSize;
    private float minTextSize;

    public interface OnTemperatureChangeListener {
        void onTemperatureChanged(int temperature);
    }

    public TemperaturePickerView(Context context) {
        super(context);
        init(context);
    }

    public TemperaturePickerView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public TemperaturePickerView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
        // 初始化文字画笔
        textPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        textPaint.setTextAlign(Paint.Align.CENTER);
        textPaint.setColor(Color.BLACK);

        // 选中线画笔
        selectedLinePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        selectedLinePaint.setStyle(Paint.Style.STROKE);
        selectedLinePaint.setStrokeWidth(3);
        selectedLinePaint.setColor(0xFF2196F3);

        // 滚动控制器
        scroller = new Scroller(context, new DecelerateInterpolator());

        // 手势检测
        gestureDetector = new GestureDetector(context, new GestureDetector.SimpleOnGestureListener() {
            @Override
            public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
                // 处理滚动
                scrollByInternal(0, (int) distanceY);
                return true;
            }

            @Override
            public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
                // 处理快速滑动
                scroller.fling(0, (int) currentScrollY, 0, (int) -velocityY,
                        0, 0, 0, (int) maxScrollY);
                Log.i(TAG, "onFling: velocityY = " + velocityY);
                adjustPosition();
                invalidate();
                return true;
            }
        });

        // 尺寸转换
        maxTextSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 24, getResources().getDisplayMetrics());
        minTextSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 14, getResources().getDisplayMetrics());
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        Log.i(TAG, "onSizeChanged: w = " + w + ", h = " + h);
        centerY = h / 2f;
        itemHeight = h / 8f;

        // 计算文字基线位置
        Paint.FontMetrics fontMetrics = textPaint.getFontMetrics();
        textBaseline = centerY - (fontMetrics.descent + fontMetrics.ascent) / 2;

        // 计算最大滚动范围
        int totalItems = MAX_TEMPERATURE - MIN_TEMPERATURE + 1;
        maxScrollY = (totalItems - 1) * itemHeight;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        drawTemperatures(canvas);
        drawSelectedLine(canvas);
    }

    private void drawSelectedLine(Canvas canvas) {
        float centerX = getWidth() / 2f;
        // 绘制选中线
        canvas.drawLine(centerX - 80, centerY, centerX + 80, centerY, selectedLinePaint);

        // 绘制选中指示器
        selectedLinePaint.setStyle(Paint.Style.FILL);
        canvas.drawCircle(centerX - 80, centerY, 8, selectedLinePaint);
        canvas.drawCircle(centerX + 80, centerY, 8, selectedLinePaint);
        selectedLinePaint.setStyle(Paint.Style.STROKE);
    }

    private void drawTemperatures(Canvas canvas) {
        float centerX = getWidth() / 2f;

        // 计算当前可见的温度范围
        int centerIndex = Math.round(currentScrollY / itemHeight);
        int startIndex = centerIndex - 2;
        int endIndex = centerIndex + 2;

        // 确保在有效范围内
        startIndex = Math.max(startIndex, 0);
        endIndex = Math.min(endIndex, MAX_TEMPERATURE - MIN_TEMPERATURE);

        for (int i = startIndex; i <= endIndex; i++) {
            int temperature = MIN_TEMPERATURE + i;
            float y = i * itemHeight - currentScrollY + centerY;

            // 计算距离中心的偏移量
            float offset = Math.abs(y - centerY);
            float scale = Math.max(0, 1 - offset / (2 * itemHeight));

            // 根据距离中心的位置调整字体大小
            float textSize = minTextSize + (maxTextSize - minTextSize) * scale;
            textPaint.setTextSize(textSize);

            // 根据距离调整透明度
            int alpha = (int) (255 * scale);
            textPaint.setAlpha(alpha);

            // 绘制温度文字
            String text = temperature + "°C";
            canvas.drawText(text, centerX, y + textBaseline - centerY, textPaint);

            // 如果是选中项，更新当前温度
            if (Math.abs(y - centerY) < itemHeight / 4) {
                if (currentTemperature != temperature) {
                    currentTemperature = temperature;
                    if (listener != null) {
                        listener.onTemperatureChanged(temperature);
                    }
                }
            }
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // 先让手势检测器处理
        if (gestureDetector.onTouchEvent(event)) {
            return true;
        }

        // 处理抬起事件，进行位置修正
        if (event.getAction() == MotionEvent.ACTION_UP) {
            adjustPosition();
            return true;
        }

        return true;
    }

    private void scrollByInternal(float x, float y) {
        float newScrollY = currentScrollY + y;

        // 限制滚动范围
        if (newScrollY < 0) {
            newScrollY = 0;
        } else if (newScrollY > maxScrollY) {
            newScrollY = maxScrollY;
        }

        currentScrollY = newScrollY;
        invalidate();
    }

    private void adjustPosition() {
        // 计算最近的刻度位置
        int targetIndex = Math.round(currentScrollY / itemHeight);
        float targetScrollY = targetIndex * itemHeight;
        Log.i(TAG, "adjustPosition: targetIndex = " + targetIndex + ", targetScrollY = " + targetScrollY);
        // 使用动画平滑滚动到目标位置
        ValueAnimator animator = ValueAnimator.ofFloat(currentScrollY, targetScrollY);
        animator.setDuration(300);
        animator.setInterpolator(new DecelerateInterpolator());
        animator.addUpdateListener(animation -> {
            currentScrollY = (float) animation.getAnimatedValue();
            invalidate();
        });
        animator.start();
    }

    @Override
    public void computeScroll() {
        if (scroller.computeScrollOffset()) {
            currentScrollY = scroller.getCurrY();

            // 限制滚动范围
            if (currentScrollY < 0) {
                currentScrollY = 0;
                scroller.forceFinished(true);
            } else if (currentScrollY > maxScrollY) {
                currentScrollY = maxScrollY;
                scroller.forceFinished(true);
            }

            invalidate();
        }
    }

    // 设置温度变化监听器
    public void setOnTemperatureChangeListener(OnTemperatureChangeListener listener) {
        this.listener = listener;
    }

    // 获取当前温度
    public int getCurrentTemperature() {
        return currentTemperature;
    }

    // 设置当前温度
    public void setTemperature(int temperature) {
        if (temperature < MIN_TEMPERATURE || temperature > MAX_TEMPERATURE) {
            return;
        }

        int index = temperature - MIN_TEMPERATURE;
        currentScrollY = index * itemHeight;
        currentTemperature = temperature;
        invalidate();
    }
}
```

运行截图：

![](/assets/img/blog/blogs_custom_view_temp_list.png){width="200" height="400" loading="lazy"}

在这个例子中，使用的 `gestureDetector` 来处理DOWN和MOVE事件，onTouchevent的事件先交由 `gestureDetector` 来处理，其内部计算具体的手势，是普通的滑动，还是快速的fling，由 `computeScroll` 回调来驱动滑动 `Scroller` 计算滑动距离。

onDraw回调里，在距离改变时，第一步计算这个滑动距离下，有哪几个item的文字应该显示到屏幕上，在对每一个温度文字在屏幕上距离显示中心的offset偏移量，根据这个偏移量来计算出文字应该显示多少透明度和字号，最后整体绘制出正确的温度列表。
