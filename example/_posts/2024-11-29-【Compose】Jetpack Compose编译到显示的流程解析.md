---
layout: post
description: > 
  本文介绍了Compose声明式框架从可组合项方法的编写到最终屏显的流程解析
image: 
  path: /assets/img/blog/blogs_cmp_new_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_cmp_new_cover.png
    960w:  /assets/img/blog/blogs_cmp_new_cover.png
    480w:  /assets/img/blog/blogs_cmp_new_cover.png
accent_image: /assets/img/blog/blogs_cmp_new_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Compose】Compose编译到显示的流程解析
Jetpack Compose 是 Google 推出的用于构建原生 Android UI 的现代声明式框架，它简化了 Android 应用的 UI 开发过程。

将XMl+Java的开发方式转变为Kotlin语法的Compose的开发方式，让开发者可以使用更简洁、更直观的代码来构建用户界面，开发体验上极致的统一。

那么一个 `@Composable` 方法是如何变成屏幕上的显示内容的呢？下面从Android平台为切入点，从编译阶段到运行时阶段，详细解析 `Compose` 的显示流程。再看看`Compose Multiplatform`这个跨平台框架和 `Android` 平台上的原型`Jetpack Compose`有何异同。

## 回顾View架构
我们的应用要加载一个显示界面时，会经历以下几个阶段。首先将xml的布局文件，按照内部的父控件子控件的包含关系，将它们解析成View树，然后将View树交给WindowManager进行显示。

具体的：

### 1. xml文件解析构建View树
当调用 `setContentView(R.layout.xxx)` 或 `LayoutInflater.inflate()` 时，系统会通过 `LayoutInflater` 解析 XML 文件。

使用 `XmlPullParser` 逐行解析 XML 标签，转换为内存中的视图对象（如 TextView、LinearLayout 等）。

根据标签属性（如 android:text、android:layout_width）设置视图的初始参数。

解析后的 XML 会生成一个对应的 视图树（View Hierarchy），根节点是顶层布局（如 ConstraintLayout），子节点是嵌套的视图。

每个视图的构造函数会被调用，并通过 `AttributeSet` 读取 XML 中的属性值（如 textSize、background）。

### 2. 测量(Measure)
`onMeasure(int widthMeasureSpec, int heightMeasureSpec)` ，父视图通过 MeasureSpec 向子视图传递尺寸约束（如 match_parent、wrap_content 或固定值）。

视图根据约束计算自身尺寸（可能需要多次测量，尤其是嵌套布局）。

最终通过 `setMeasuredDimension()` 保存测量结果。

### 3. 布局(Layout)
`onLayout(boolean changed, int l, int t, int r, int b)` ，父视图根据测量结果确定子视图的位置（左上右下坐标）。

例如：LinearLayout 会按垂直/水平方向依次排列子视图。

### 4. 绘制(Draw)
`onDraw(Canvas canvas)` ，视图通过 Canvas 和 Paint 绘制自身内容（如文本、背景、边框）。

绘制顺序：背景 → 主体内容（如文本/图片） → 子视图 → 前景（如滚动条）。

支持硬件加速时，绘制指令会转为 RenderNode 并交由 GPU 处理。

### 5. 合成送显
`SurfaceFlinger` 合成。各层的绘制结果（Surface）由 SurfaceFlinger 合成为最终帧。提交到屏幕时，通过 VSync 信号同步，将帧数据发送到屏幕缓冲区显示。

一般View架构的应用，在做布局相关性能优化时，有如下手段：

> 减少布局层级：避免嵌套过深，用 ConstraintLayout 替代多层 LinearLayout。

> 避免过度绘制：通过 onDraw() 优化或设置 android:background=null。

> 使用 ViewStub：延迟加载复杂但非立即显示的布局。

## Compose 的 UI Tree
### 前言一 Gap Buffer
Gap Buffer 是一种用于优化局部更新的高效数据结构。其核心思想是通过维护一个 **可移动的“间隙”** 来优化局部性操作，在数组中预留一个空白区域（Gap）来实现高效的插入和删除操作，可以减少内存移动的开销。

#### **最常见的应用——文本编辑器**
​缓冲区结构​​，将内存分为三部分：左文本区、间隙（Gap）和右文本区。
初始时，间隙通常位于缓冲区末尾（如 `[文本][间隙]` ），但随着输入的光标移动，这个间隙会动态调整位置。

* ​当在光标处插入字符时，直接将数据填入间隙。若间隙不足，则扩展间隙（如重新分配更大的内存）。
* 删除字符时，通过调整间隙边界“吸收”被删除的字符，避免立即移动数据。
* ​​光标移动​​：移动光标时，间隙会同步移动到新位置，此时需要将原间隙两侧的文本交换位置（例如，光标右移时，将右文本区的左端字符移到左文本区末尾）。

关键操作示例​​:
```
​​插入字符​​：
假设缓冲区状态：[Hello][ ][World]（[]表示间隙）。
在Hello后插入! → [Hello!][ ][World]，间隙缩小。

​​移动光标​​：
光标从Hello!后移动到W前：
原状态：[Hello!][ ][World]
移动后：[Hello! ][W][orld]（间隙移动到W前，W从右文本区移到左文本区）。

​​删除字符​​：
删除W → [Hello! ][][orld]，间隙扩展“吸收”W。
```

其优势主要为高效的局部操作，劣势为大范围的操作时，需要移动大量数据调整间隙的位置，最坏的情况下可能需要O(n)的时间复杂度。同时间隙填满后，扩展的成本也较高。

### 前言二 Slot Table
#### **数据结构描述**
SlotTable 是 Compose 里的内部数据结构，用于跟踪 **组合层次结构** 中的视图数据，包括 **节点、组、键和记忆值** 。这个数据结构上的各个组的结构及其值由编译器决定，并在运行时随着层次结构和应用状态的建立和更新而变化。

SlotTable 是一个 **树形结构** ，其中每个节点都是一个组，且内部可能有子项。其每个元素被称作“插槽（Slot）”。每个插槽能存储特定类型的数据，像组件的类型、属性、状态等信息。它以扁平化的方式存储 UI 树的信息，取代了传统的树形结构，从而简化了 UI 的管理与更新操作。

组（Groups）包含以下信息：
1. **键**： 用于区分组的识别符，通过快速识别组的变化来帮助重新组合。它不需要是唯一的。
2. **标志**： 有关分组的元数据，包括分组所含节点数的计数器。
3. **槽**： 为组存储的值的有序列表，可以修改或删除。槽支持引用类型和基元，可独立跟踪以避免自动排序惩罚。实用槽由槽表管理，其他槽则由金豪编译器生成，跟踪记忆值和可编译函数参数。

还有以下的可选属性：
1. **节点**：与组相关联的节点，由 Applier 使用。Composer 通过 SlotTable 在内部维护这些节点。
2. **对象键**：补充标准整数键的可选键。
3. **辅助值***：与节点相关联的辅助数据值，设置与组的其他槽无关。它用于记录 CompositionLocal 地图。

SlotTable 的实现是一个基于页面的链接表，它将组信息编码成整数，并将其打包成数组，以避免额外的分配。组内部维护了 **几个指针** 指向其父组、第一个子组和下一个同级组的指针，编码为指向页面的地址和页面内的索引。

该数据结构返回和使用的所有 GroupAddresses 都是稳定的。一旦分配，地址将不会改变，除非将组删除并重新添加到表中。

一个 SlotTable 可以与另一个 SlotTable 共享地址空间，这样就可以通过指针重新分配而不是内存复制，在表之间有效地移动组。

### 编译器对 Composable 函数的转换
我们编写界面UI时，使用的可组合项都会添加一个 `@Composable` 注解，被 @Composable 所注解的函数称为 `可组合函数` 。

添加该注解的函数会被真实地改变类型，改变方式与 suspend 类似，在编译期进行处理，只不过 Compose 并非语言特性，无法采用语言关键字的形式进行实现。

示例：
```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello $name")
}

// 编译器生成的近似结构（概念性表示）
fun Greeting(name: String, parentComposer: Composer, changed: Int) {
    val composer = parentComposer.startRestartGroup(GROUP_HASH)

    val dirty = calculateState(changed)
    
    if (stateHasChanged(dirty) || composer.skipping) {
        Text("Hello $name", composer = composer, changed = ...)
    } else {
        composer.skipToGroupEnd()
    }

    composer.endRestartGroup()?.updateScope {
        Greeting(name, changed)
    }
}
```

可见被 `@Composable` 注解后，函数增添了额外的参数，其中的 `Composer` 类型参数 **作为运行环境** 贯穿在整个可组合函数调用链中，所以可组合函数无法在普通函数中调用，因为 **不包含相应的环境** 。

可组合函数实现的起始与结尾通过 `Composer.startRestartGroup()` 与 `Composer.endRestartGroup()` 在 Slot Table 中创建 Group，而可组合函数内部所调用的可组合函数在两个调用之间创建新的 Group，从而 **在 Slot Table 内部完成视图树的构建** 。

Composer 根据当前是否正在修改视图树而确定这些调用的实现类型。

在视图树构建完成后，若数据更新导致部分视图需要刷新，此时非刷新部分对应可组合函数的调用就不再是进行视图树的构建，而是视图树的访问，正如代码中的 `Composer.skipToGroupEnd()` 调用，表示在访问过程中直接跳到当前 Group 的末端。

Composer 对 Slot Table 的操作是读写分离的，只有写操作完成后才将所有写入内容更新到 Slot Table 中。

除此之外，可组合函数还将通过 **传入标记参数的位运算** 判断内部的可组合函数执行或跳过，这可以 **避免访问无需更新的节点** ，提升执行效率。

#### **Gap Buffer在 Compose 中的应用**
Gap Buffer 是 Compose 内部用于管理 Slot Table 的核心数据结构。

Compose 编译器会将 `@Composable` 函数编译成上面的形式。当Compose 运行时执行这些函数，并将组合项的信息添加到一个名为 Slot Table 的数据结构中。`Slot Table` 的每个槽可以存储关于 Composable 的信息，如其参数、内部状态（如 `remember` 持有的值）以及其他组合细节。它本质上是记录组合过程的。

而 `Gap Buffer` 是 `Slot Table` 的底层实现。在 Slot Table 的上下文中，当 UI 由于状态变化而需要更新时（重组），Compose 需要更新 Slot Table 中的特定部分。由于移动 gap 本身是一个 `O(n)` 操作，这在典型的 UI 更改中并不频繁。大多数 UI 更新涉及的为小的、局部化的修改，Gap Buffer 在这些场景中非常高效。

Compose 不直接构建传统的 "view tree" 像 Android Views 那样，但它确实在 **Composition 阶段** 构建了一个 **UI tree** (node tree) 。这个 UI tree 代表了 UI 元素的层次结构。Slot Table会使用 Gap Buffer 来存储与这个 UI tree 相关的元数据和状态信息。当状态改变时，Compose 会确定哪些 `@Composable` 函数需要重组。即基于 Slot Table 中的信息来确定哪些部分的 UI tree 需要更新。由于 gap buffer 的高效性，Compose 可以在 Slot Table 中高效地插入、删除或移动 "组" 中的 composables 可组合项，而不需要重建整个 UI tree。这就是 Compose 实现 "smart recomposition" 的关键所在，即只更新受状态变化影响的部分，从而显著提高性能。

整体结构的工作流程如下： 

**1. 组合阶段**
在首次运行或状态改变时，Composable 函数会被执行，生成 UI 描述树。此时，Composer 会遍历这个 UI 描述树，把相关信息写入 Slot Table。例如，可组合函数实现的起始与结尾通过 Composer.startRestartGroup() 与 Composer.endRestartGroup() 在 Slot Table 中创建 Group，以此来表示 UI 树的层次结构。

**2. 差异比较阶段**
当可组合项所观测的 `mutableStateOf` 值发生变化，导致部分组合无效。
Compose 重新执行受到影响的 `@Composable` 函数。即触发了重组，Composer 会将新生成的 UI 描述树与 Slot Table 里存储的上一次组合结果进行比较，找出需要更新的部分，生成一个变更列表（Change List）。

**3. 更新阶段**
依据变更列表，Composer 对 Slot Table 进行更新，仅修改那些发生变化的插槽，而不改变未变化的部分。这可能涉及插入新的槽来表示新的 composables，删除槽来表示移除的 composables，或者更新现有槽的数据以反映参数/状态的变化。这种局部更新的方式提升了 UI 更新的效率。

## Compose的测量过程
`Android View` 系统，内部可能会进行多次测量，这样测量次数随嵌套层数加深会成几何式增长。而 `Compose` 采用了 **单遍测量（Single-pass Measurement）** 的机制，这大大提升了性能和可预测性。

Compose 的测量流程是自上而下、递归进行的，遵循以下三步算法：

1.  **测量子项 (Measure children)**：
    * 父节点（Composable）会向其所有子节点发出测量请求。
    * 这个请求会向下传递**约束条件 (Constraints)**。约束条件定义了子项可用的最小和最大宽度、高度。
    * 子项在这些约束条件下决定自己的尺寸。

2.  **确定自身尺寸 (Decide own size)**：
    * 在子项完成测量并报告其尺寸后，父节点会根据其所有子项的测量结果和自身的逻辑（例如 `Row` 会累加子项的宽度，`Column` 会累加子项的高度）来决定自己的最终尺寸。

### 约束条件 (Constraints) 的传递
在测量过程中，约束条件从父节点向下传递给子节点：

* **父节点决定了子节点的最大可用空间**。它会根据自身被赋予的约束条件和其布局逻辑，生成并传递新的约束条件给子节点。
* 子节点在这些约束范围内，根据自身内容（如文本长度、图片大小等）和修饰符（Modifiers）的设置，决定自己的尺寸。
* 叶子节点（没有子节点的 Composable，如 `Text`、`Image`）直接根据收到的约束条件和自身内容来决定尺寸并报告给其父节点。

### 测量示例 (以 `Row` 包含 `Image` 和 `Column` 为例)
假设我们有一个 UI 结构：

```kotlin
Row {
    Image(...)
    Column {
        Text("Hello")
        Text("Compose")
    }
}
```

测量流程如下：

1.  `Row` 节点被要求测量自身。
2.  `Row` 首先会要求其子项 `Image` 和 `Column` 进行测量。
3.  **测量 `Image`：**
    * `Image` 是一个叶子节点，它没有子节点。
    * 它根据收到的约束条件和图片自身的尺寸来决定其最终尺寸，并报告给 `Row`。
4.  **测量 `Column`：**
    * `Column` 会先要求其子项（两个 `Text` Composable）进行测量。
    * **测量第一个 `Text`：** 它是叶子节点，根据自身文本内容和约束条件决定尺寸，并报告给 `Column`。
    * **测量第二个 `Text`：** 同上，决定尺寸并报告给 `Column`。
    * `Column` 收到两个 `Text` 的尺寸后，根据其布局逻辑（通常是最大子项宽度和子项高度之和）来决定自己的尺寸，并报告给 `Row`。
5.  **`Row` 确定自身尺寸和放置子项：**
    * `Row` 收到 `Image` 和 `Column` 的尺寸后，根据其布局逻辑（通常是子项宽度之和和最大子项高度）来决定自己的尺寸。
    * 最后，`Row` 会相对于自身的位置来放置 `Image` 和 `Column`。

### 固有特性测量 (Intrinsic Measurements)
虽然 Compose 强制单遍测量，但在某些情况下，父节点可能需要在 **实际测量子节点之前** ，了解子节点的一些 **“固有”尺寸信息** （例如，一个 `Text` 在无限宽度下能达到的最小高度）。这时就用到了**固有特性测量 (Intrinsic Measurements)**。

固有特性测量允许父节点“查询”子节点，获取其在给定约束条件下的最小或最大固有尺寸（如 `minIntrinsicWidth`、`maxIntrinsicWidth`、`minIntrinsicHeight`、`maxIntrinsicHeight`）。这些查询并**不**是真正的测量，它们不会导致子节点被实际测量两次。它们只是让父节点能够根据这些预估信息，来更好地计算在实际测量时应该传递给子节点的约束条件。

例如，当你使用 `Modifier.height(IntrinsicSize.Min)` 时，它会要求父级布局根据其子项的最小固有高度来确定自身高度。

## Android平台的Compose显示
在 Android 平台，Compose UI 实际上运行在一个 `ComposeView` 内部，
ComposeView 是一个特殊的 View 类（它继承自 `androidx.compose.ui.platform.AbstractComposeView`，而 `AbstractComposeView` 又是一个 ViewGroup），它充当了 Compose UI 内容的容器。它本身是一个标准的 Android View，可以像其他任何 TextView 或 LinearLayout 一样被添加到 View 层次结构中。

### 渲染
Compose 在 Android 上的实现最终依赖于 AndroidComposeView，且这是一个 ViewGroup，那么按原生视图渲染的角度，看一下 AndroidComposeView 对 onDraw() 与 dispatchDraw() 的实现，即可看到 Compose 渲染的原理。

```kotlin
@SuppressLint("ViewConstructor", "VisibleForTests")
@OptIn(ExperimentalComposeUiApi::class)
@RequiresApi(Build.VERSION_CODES.LOLLIPOP)
internal class AndroidComposeView(context: Context) :
    ViewGroup(context), Owner, ViewRootForTest, PositionCalculator {
    
    ...
    
    override fun onDraw(canvas: android.graphics.Canvas) {
    }
      
    ...
    
    override fun dispatchDraw(canvas: android.graphics.Canvas) {
        ...
        measureAndLayout()

        // we don't have to observe here because the root has a layer modifier
        // that will observe all children. The AndroidComposeView has only the
        // root, so it doesn't have to invalidate itself based on model changes.
        canvasHolder.drawInto(canvas) { root.draw(this) }

        ...
    }
    
    ...
}
```

可以看到，在 `dispatchDraw()` 中，调用了 `measureAndLayout()` 方法，这个方法会执行 Compose 中的测量和布局过程，最终会调用 `root.draw()` 方法，这个方法会执行 Compose 中的绘制过程，最终将绘制结果绘制到 Canvas 上。

### Compose和View兼容使用
#### **ComposeView的使用**
在原来的Android项目中使用Compose，可以使用 `ComposeView` 来嵌入Compose UI。

ComposeView 的工作原理：
* 当 ComposeView 被添加到 View 层次结构并依附到窗口时，它会启动一个 Compose 组合（Composition）。这个组合会运行你传递给 setContent 的 `@Composable` 函数。
* Compose UI 的整个生命周期（组合、布局、绘制）都发生在 ComposeView 的边界内。
* ComposeView 负责将其内部 Compose UI 的绘制结果，最终通过标准的 Android 渲染管道（HWUI 和 Skia）呈现在屏幕上，就像其他 View 一样。
* ComposeView 也负责管理 Compose UI 的生命周期，例如在 ComposeView 被从窗口分离或宿主 Lifecycle 销毁时，正确地处置 Compose 组合。

#### **AndroidView**
在Compose中，使用传统的View，需要使用 `AndroidView` 来包裹，这样才能正常显示。

使用示例：

```kotlin
@Composable
fun MyViewInCompose() {
    var counter by remember { mutableStateOf(0) }

    Column {
        Text("Compose counter: $counter")
        AndroidView(
            factory = { context ->
                // 第一次组合时创建并返回一个传统的 Button
                android.widget.Button(context).apply {
                    text = "Click me (View)"
                    setOnClickListener {
                        counter++
                    }
                }
            },
            update = { button ->
                // 每次重组时更新 Button 的文本
                button.text = "Click me (View) - Count: $counter"
            }
        )
    }
}
```

可以看到 `AndroidView` 接受一个 `factory lambda`，这个 lambda 会返回你想要嵌入的 View 实例。这个 factory 只会在首次组合时执行一次。

它还接受一个 update lambda，这个 lambda 会在 Compose 重组时（当 AndroidView 的参数发生变化时）被调用，允许你更新嵌入 View 的属性，以响应 Compose 状态的变化。

#### **AndroidViewBinding**
另一个在Compose中使用传统View的方案是 `AndroidViewBinding` ，使用示例：

```kotlin
// my_layout.xml
// <LinearLayout ...>
//     <TextView android:id="@+id/my_text_view" ... />
// </LinearLayout>

@Composable
fun MyXmlLayoutInCompose() {
    var message by remember { mutableStateOf("Hello from XML!") }

    Column {
        Text("Compose Message: $message")
        AndroidViewBinding(MyLayoutBinding::inflate) {
            // 在这里可以访问 my_text_view
            myTextView.text = message
            myTextView.setOnClickListener {
                message = "XML updated!"
            }
        }
    }
}
```

这个组件用于嵌入整个 XML 布局文件，AndroidViewBinding 接受一个 View Binding 类的 inflate 方法引用作为参数。也提供了一个 update lambda，你可以在其中访问绑定对象并更新 XML 布局中各个 View 的属性。

**AndroidView / AndroidViewBinding 的工作原理**

当这些 Composable 被组合时，它们会创建一个传统的 Android View 实例（或一个 View 层次结构）。
Compose 会将这个 View 实例插入到其内部的 LayoutNode 树中，确保它能够参与 Compose 的布局和绘制过程。
尽管这些 View 是传统的 Android View，它们仍然被 Compose 的单遍测量和布局规则所管理。Compose 会向它们传递约束，并根据它们报告的尺寸进行布局。
绘制时，Compose 会在适当的时机调用嵌入 View 的 draw() 方法，并将结果集成到 Compose 自身的渲染中。

## Compose和View指标对比
以下来自Google官方文档，有删减。

[比较 Compose 指标和 View 指标](https://developer.android.com/develop/ui/compose/migrate/compare-metrics?hl=zh-cn)

### APK 大小
将库添加到项目中会增加其 APK 大小。

首次将 Compose 添加到 Sunflower 后，APK 大小从 2,252 KB 增加到 3,034 KB，增加了 782 KB。生成的 APK 包含混合了 View 和 Compose 的界面 build。由于向 Sunflower 添加了其他依赖项，因此出现这种增加是意料之中的。

相反，将 Sunflower 迁移为 **仅使用 Compose 的应用** 后，APK 大小从 3,034 KB 减少到 2,966 KB，减少了 68 KB。

之所以减少，是因为移除了未使用的 View 依赖项，例如 AppCompat 和 ConstraintLayout。

### 构建时间
添加 Compose 会增加应用的构建时间，因为 Compose 编译器会处理应用中的可组合项。以下结果是使用独立的 gradle-profiler 工具获得的，该工具会多次执行构建，以便为 Sunflower 的调试 build 时长获取平均构建时间。

首次将 Compose 添加到 Sunflower 时，平均构建时间从 299 毫秒增加到 399 毫秒，增加了 100 毫秒。这是因为 Compose 编译器会执行其他任务来转换项目中定义的 Compose 代码。

相反，在完成 Sunflower 向 Compose 的迁移后，平均构建时间缩短至 342 毫秒，减少了 57 毫秒。构建时间缩短可以归因于多种因素，这些因素共同缩短了构建时间，例如移除数据绑定、将使用 kapt 的依赖项迁移到 KSP，以及将多个依赖项更新到最新版本。

### 使用基准配置文件帮助Compose实现AOT编译
又可能仅为Google Play实现，国内商店尚未调研。

由于 Jetpack Compose 是未捆绑库，因此它无法受益于 Zygote，后者会 **预加载 View 系统的界面工具包类和可绘制对象** 。Jetpack Compose 1.0 利用了 release build 的配置文件安装。ProfileInstaller 可让应用指定要在安装时进行预编译 (AOT) 的关键代码。Compose 随附配置文件安装规则，可减少 Compose 应用的启动时间和卡顿。

基准配置文件是加快常见用户体验历程的绝佳方式。在应用中添加基准配置文件可以避免对包含的代码路径执行解译和即时 (JIT) 编译步骤，从而使应用首次启动时的代码执行速度即可提高约 30%。

Jetpack Compose 库包含自己的基准配置文件，当您在应用中使用 Compose 时，系统会自动获取这些优化。不过，这些优化仅会影响 Compose 库内的代码路径，因此我们建议您向应用添加基准配置文件，以涵盖 Compose 之外的代码路径。

## 跨平台框架Compose Multiplatform的显示渲染
最后一个章节是由Jetbrains维护的Compose跨平台版本，在各个平台上渲染方式的总结。

Compose Multiplatform (CMP) 的核心理念是共享 UI 代码，并尽可能在不同平台上提供原生级别的性能和外观。它实现这一目标的关键在于其底层的渲染机制，尤其是对 [Skia](https://skia.org/) 图形库的依赖。

**核心原理：Skia 作为跨平台图形引擎**

Compose Multiplatform 利用 **Skiko** (Skia for Kotlin) 这个库，将强大的 Skia 图形库引入到 Kotlin/JVM 和 Kotlin/Native 环境中。Skia 是 Google 开发的一个开源 2D 图形库，用于绘制文本、几何图形和图像。Chrome 浏览器、Android 操作系统、Flutter 等都使用 Skia 进行图形渲染。

这意味着，在大部分支持的平台上，Compose Multiplatform 的 UI 并不是直接映射到平台的原生 UI 组件（例如 Android 上的 `View` 或 iOS 上的 `UIView`），而是**将 UI 描述转化为 Skia 绘制指令，然后由 Skia 在平台的画布上进行硬件加速渲染。**

让我们逐一看看不同平台的渲染方式：

### 1. Android

* **渲染方式：** Compose Multiplatform 在 Android 平台上直接使用 **Jetpack Compose** 的渲染机制。Jetpack Compose 内部也会将 Composable 的 UI 描述转换为 `RenderNode`，然后通过 Android 的 **HWUI (Hardware Accelerated UI)** 渲染管道，利用 GPU (通常是 OpenGL ES) 进行绘制。
* **与传统 View 的集成：** 如前所述，Compose UI 运行在一个特殊的 `ComposeView` 中，这个 `ComposeView` 本身是一个传统的 Android `View`，它负责承载 Compose 内容并将其绘制到屏幕上。因此，最终 Compose 的绘制内容会通过 `ComposeView` 的 `Canvas` 传递给底层的 Android 渲染系统。
* **性能：** 高性能，得益于 Jetpack Compose 的单遍测量、智能重组以及 Android 硬件加速的渲染管道。

### 2. iOS

* **渲染方式：** 这是 Compose Multiplatform 最引人注目的突破之一。在 iOS 上，Compose Multiplatform **不使用 UIKit/SwiftUI 原生组件**。它利用 **Kotlin/Native** 技术将 Kotlin 代码编译为原生二进制代码，并通过 **Skiko** 库将 Compose UI 的绘制指令转化为 Skia 调用。
* **底层：** 最终，这些 Skia 绘制指令会在一个特殊的 **`UIView` (通常是一个 `UIViewController` 的内容 View)** 上进行渲染。这个 `UIView` 充当一个画布，Compose 通过 Skia 直接在上面绘制所有 UI 元素。
* **优势：** 实现了高度一致的 UI 表现，因为绘制逻辑是共享的。同时，它也提供了与原生 iOS 行为（如滚动物理、文本编辑、辅助功能）的紧密集成，以确保应用感觉原生。
* **互操作性：** 尽管 Compose UI 自身是基于 Skia 绘制的，但它仍然提供了与 UIKit 和 SwiftUI 的互操作性，允许你在 Compose 屏幕中嵌入原生 iOS View，或将 Compose UI 嵌入到现有的 iOS 应用中。
* **性能：** 通过直接利用 GPU 和高效的 Skia 渲染，Compose Multiplatform 在 iOS 上也能实现接近原生的性能。

### 3. Desktop (macOS, Windows, Linux)

* **渲染方式：** 在桌面平台上，Compose Multiplatform 运行在 **JVM** 上。它也通过 **Skiko** 库，将 Compose UI 的绘制指令传递给 Skia。
* **底层：** Skia 会利用底层的图形 API（如 OpenGL、DirectX 或 Vulkan，具体取决于平台和驱动）在桌面窗口中进行硬件加速渲染。
* **平台特性：** Compose Multiplatform for Desktop 提供了桌面平台特有的功能，如窗口管理、菜单栏、系统托盘、文件选择器等，这些功能通常通过平台特定的 API 实现，并与 Compose UI 集成。
* **性能：** 高性能，充分利用桌面硬件加速，提供流畅的 UI 体验。

### 4. Web (Wasm / JavaScript)

* **渲染方式：** Compose Multiplatform for Web 目标是 **Kotlin/Wasm**（或之前的 Kotlin/JS），它将 Compose UI 编译为 WebAssembly 或 JavaScript。
* **底层：** 渲染机制主要是将整个 Compose UI 绘制到一个 HTML 的 **`<canvas>` 元素**中。同样，这得益于 Skiko 库，Skia 会被编译为 WebAssembly，并在浏览器中进行渲染。
* **特点：**
    * **全屏画布：** 你的整个 Compose Multiplatform Web 应用通常被渲染为一个大的画布元素。
    * **非原生 DOM：** 这意味着你的 UI 元素不是标准的 HTML DOM 元素（如 `<div>`, `<p>`, `<button>`)。因此，一些传统的 Web 特性（如文本选择、右键上下文菜单、SEO 优化）可能需要额外的处理或适配。
    * **性能：** 依赖于浏览器的 `<canvas>` 性能和 Skia/Wasm 的渲染效率。对于复杂的、动态的 UI，它通常表现良好。
* **Compose HTML (补充)：** 值得注意的是，JetBrains 还提供了一个独立的库 **Compose HTML**。Compose HTML **不是** Compose Multiplatform 的一部分，它仅用于 Kotlin/JS，允许你使用 Compose 的声明式 API 来直接构建和操作 HTML DOM 元素。这意味着它能更好地与 Web 的原生特性集成，但不能共享 UI 渲染代码到移动/桌面平台。Compose Multiplatform Web 主要关注基于 Skia 的画布渲染。

### 总结

Compose Multiplatform 的核心渲染策略是使用 **Skia 作为跨平台图形引擎**。它将你的声明式 UI 描述转化为 Skia 的绘制指令，然后在每个平台的画布上进行硬件加速渲染。这使得 UI 能够保持高度的一致性，同时通过平台特定的集成层，尽可能地提供原生性能和用户体验。

| 平台    | 渲染引擎/方式                                      | 底层技术/API                                 |
| :------ | :------------------------------------------------- | :------------------------------------------- |
| **Android** | Jetpack Compose (HWUI)                             | `RenderNode`, OpenGL ES                      |
| **iOS** | Skia (通过 Skiko 库直接绘制到 `UIView` 画布上) | Kotlin/Native, Skia, Metal/OpenGL ES         |
| **Desktop** | Skia (通过 Skiko 库直接绘制到桌面窗口)           | JVM, Skia, OpenGL/DirectX/Vulkan (取决于平台) |
| **Web** | Skia (通过 Skiko 库绘制到 HTML `<canvas>` 元素)  | Kotlin/Wasm (或 JS), Skia, WebGL             |