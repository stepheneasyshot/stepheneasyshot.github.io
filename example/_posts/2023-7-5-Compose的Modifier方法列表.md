---
layout: post
description: > 
  本文介绍了Jetpack Compose里Modifier是修饰方法列表
image: 
  path: /assets/img/blog/blogs_compose_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_compose_cover.png
    960w:  /assets/img/blog/blogs_compose_cover.png
    480w:  /assets/img/blog/blogs_compose_cover.png
accent_image: /assets/img/blog/blogs_compose_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Compose的Modifier方法列表
摘录自官方文档，用于在网络条件不好的时候查阅：


操作
作用域：任意
@ExperimentalFoundationApi
<T : Any?> Modifier.anchoredDraggable(
    state: AnchoredDraggableState<T>,
    orientation: Orientation,
    enabled: Boolean,
    interactionSource: MutableInteractionSource?,
    overscrollEffect: OverscrollEffect?,
    startDragImmediately: Boolean
)
在一组预定义值之间启用拖动手势。

作用域：任意
@ExperimentalFoundationApi
<T : Any?> Modifier.anchoredDraggable(
    state: AnchoredDraggableState<T>,
    reverseDirection: Boolean,
    orientation: Orientation,
    enabled: Boolean,
    interactionSource: MutableInteractionSource?,
    overscrollEffect: OverscrollEffect?,
    startDragImmediately: Boolean
)
在一组预定义值之间启用拖动手势。

作用域：任意
Modifier.clickable(
    enabled: Boolean,
    onClickLabel: String?,
    role: Role?,
    onClick: () -> Unit
)
将组件配置为通过输入或无障碍“点击”事件接收点击操作。

作用域：Any
Modifier.clickable(
    interactionSource: MutableInteractionSource?,
    indication: Indication?,
    enabled: Boolean,
    onClickLabel: String?,
    role: Role?,
    onClick: () -> Unit
)
将组件配置为通过输入或无障碍“点击”事件接收点击操作。

作用域：Any
@ExperimentalFoundationApi
Modifier.combinedClickable(
    enabled: Boolean,
    onClickLabel: String?,
    role: Role?,
    onLongClickLabel: String?,
    onLongClick: (() -> Unit)?,
    onDoubleClick: (() -> Unit)?,
    onClick: () -> Unit
)
将组件配置为通过输入或无障碍“点击”事件接收点击、双击和长按操作。

作用域：Any
@ExperimentalFoundationApi
Modifier.combinedClickable(
    interactionSource: MutableInteractionSource?,
    indication: Indication?,
    enabled: Boolean,
    onClickLabel: String?,
    role: Role?,
    onLongClickLabel: String?,
    onLongClick: (() -> Unit)?,
    onDoubleClick: (() -> Unit)?,
    onClick: () -> Unit
)
将组件配置为通过输入或无障碍“点击”事件接收点击、双击和长按操作。

作用域：任意
@ExperimentalFoundationApi
Modifier.mouseClickable(
    enabled: Boolean,
    onClickLabel: String?,
    role: Role?,
    onClick: MouseClickScope.() -> Unit
)
创建与 Modifier.clickable 类似的修饰符，但会提供关于按下的按钮和键盘修饰符的信息的额外上下文

作用域：任意
@ExperimentalFoundationApi
Modifier.draggable2D(
    state: Draggable2DState,
    enabled: Boolean,
    interactionSource: MutableInteractionSource?,
    startDragImmediately: Boolean,
    onDragStarted: (startedPosition: Offset) -> Unit,
    onDragStopped: (velocity: Velocity) -> Unit,
    reverseDirection: Boolean
)
为界面元素配置在两种屏幕方向下的触摸拖动。

作用域：任意
Modifier.draggable(
    state: DraggableState,
    orientation: Orientation,
    enabled: Boolean,
    interactionSource: MutableInteractionSource?,
    startDragImmediately: Boolean,
    onDragStarted: suspend CoroutineScope.(startedPosition: Offset) -> Unit,
    onDragStopped: suspend CoroutineScope.(velocity: Float) -> Unit,
    reverseDirection: Boolean
)
为界面元素配置单个 Orientation 的触摸拖动。

作用域：Any
Modifier.selectableGroup()
使用此修饰符将用于实现无障碍功能的一系列 selectable 项（例如 Tab 或 RadioButton）归到一组。

作用域：Any
Modifier.selectable(
    selected: Boolean,
    enabled: Boolean,
    role: Role?,
    onClick: () -> Unit
)
将组件配置为可选择，通常作为互斥组的一部分，在任何时间点只能选择该组中的一项。

作用域：Any
Modifier.selectable(
    selected: Boolean,
    interactionSource: MutableInteractionSource?,
    indication: Indication?,
    enabled: Boolean,
    role: Role?,
    onClick: () -> Unit
)
将组件配置为可选择，通常作为互斥组的一部分，在任何时间点只能选择该组中的一项。

作用域：任意
@ExperimentalMaterialApi
<T : Any?> Modifier. swipeable(
    state: SwipeableState<T>,
    anchors: Map<Float, T>,
    orientation: Orientation,
    enabled: Boolean,
    reverseDirection: Boolean,
    interactionSource: MutableInteractionSource?,
    thresholds: (from, to) -> ThresholdConfig,
    resistance: ResistanceConfig?,
    velocityThreshold: Dp
)
此函数已废弃。Material 的 Swipeable 已被 Foundation 的 AnchoredDraggable API 取代。

作用域：任意
@ExperimentalWearMaterialApi
<T : Any?> Modifier.swipeable(
    state: SwipeableState<T>,
    anchors: Map<Float, T>,
    orientation: Orientation,
    enabled: Boolean,
    reverseDirection: Boolean,
    interactionSource: MutableInteractionSource?,
    thresholds: (from, to) -> ThresholdConfig,
    resistance: ResistanceConfig?,
    velocityThreshold: Dp
)
在一组预定义状态之间启用滑动手势。

作用域：Any
Modifier.toggleable(
    value: Boolean,
    enabled: Boolean,
    role: Role?,
    onValueChange: (Boolean) -> Unit
)
将组件配置为可通过输入和无障碍事件切换

作用域：Any
Modifier.toggleable(
    value: Boolean,
    interactionSource: MutableInteractionSource?,
    indication: Indication?,
    enabled: Boolean,
    role: Role?,
    onValueChange: (Boolean) -> Unit
)
将组件配置为可通过输入和无障碍事件切换。

作用域：Any
Modifier.triStateToggleable(
    state: ToggleableState,
    enabled: Boolean,
    role: Role?,
    onClick: () -> Unit
)
将组件配置为可通过输入和无障碍事件在三种状态之间切换：启用、停用和不确定。

作用域：Any
Modifier.triStateToggleable(
    state: ToggleableState,
    interactionSource: MutableInteractionSource?,
    indication: Indication?,
    enabled: Boolean,
    role: Role?,
    onClick: () -> Unit
)
将组件配置为可通过输入和无障碍事件在三种状态之间切换：启用、停用和不确定。

对齐方式
作用域： RowScope
Modifier.align(alignment: Alignment.Vertical)
在 Row 中垂直对齐元素。

作用域： RowScope
Modifier.alignBy(alignmentLineBlock: (Measured) -> Int)
垂直放置元素，使由 alignmentLineBlock 确定的内容的对齐线与同样配置为 alignBy 的同级元素对齐。

作用域： RowScope
Modifier.alignBy(alignmentLine: HorizontalAlignmentLine)
垂直放置元素，使其 alignmentLine 与同样配置为 alignBy 的同级元素对齐。

作用域： RowScope
Modifier.alignByBaseline()
垂直放置元素，使其第一条基线与同样配置为 alignByBaseline 或 alignBy 的同级元素对齐。

作用域： ColumnScope
Modifier.align(alignment: Alignment.Horizontal)
在 Column 中水平对齐元素。

作用域： ColumnScope
Modifier.alignBy(alignmentLineBlock: (Measured) -> Int)
水平放置元素，使由 alignmentLineBlock 确定的内容的对齐线与同样配置为 alignBy 的同级元素对齐。

作用域： ColumnScope
Modifier.alignBy(alignmentLine: VerticalAlignmentLine)
水平放置元素，使其 alignmentLine 与同样配置为 alignBy 的同级元素对齐。

作用域： BoxScope
Modifier.align(alignment: Alignment)
将内容元素拉取到 Box 中的特定 Alignment。

动画
作用域： AnimatedVisibilityScope
Modifier.animateEnterExit(
    enter: EnterTransition,
    exit: ExitTransition,
    label: String
)
animateEnterExit 修饰符可用于 AnimatedVisibility 的任何直接或间接子项，以创建与 AnimatedVisibility 中指定的内容不同的进入/退出动画。

作用域： LazyItemScope
Modifier.animateItem(
    fadeInSpec: FiniteAnimationSpec<Float>?,
    placementSpec: FiniteAnimationSpec<IntOffset>?,
    fadeOutSpec: FiniteAnimationSpec<Float>?
)
此修饰符可为项的显示（淡入）、消失（淡出）和位置更改（例如项重新排序）添加动画效果。

作用域： LazyItemScope
@ExperimentalFoundationApi
Modifier. animateItemPlacement(
    animationSpec: FiniteAnimationSpec<IntOffset>
)
此函数已废弃。请改用 Modifier.animateItem()

作用域： LazyStaggeredGridItemScope
Modifier.animateItem(
    fadeInSpec: FiniteAnimationSpec<Float>?,
    placementSpec: FiniteAnimationSpec<IntOffset>?,
    fadeOutSpec: FiniteAnimationSpec<Float>?
)
此修饰符可为项的显示（淡入）、消失（淡出）和位置更改（例如项重新排序）添加动画效果。

作用域： LazyStaggeredGridItemScope
@ExperimentalFoundationApi
Modifier. animateItemPlacement(
    animationSpec: FiniteAnimationSpec<IntOffset>
)
此函数已废弃。请改用 Modifier.animateItem()

边框
作用域：Any
Modifier.border(border: BorderStroke, shape: Shape)
修改元素，以添加使用 border 和 shape 指定了外观的边框，并进行裁剪。

作用域：Any
Modifier.border(width: Dp, brush: Brush, shape: Shape)
修改元素，以添加使用 width、brush 和 shape 指定了外观的边框，并进行裁剪。

作用域：Any
Modifier.border(width: Dp, color: Color, shape: Shape)
修改元素，以添加使用 width、color 和 shape 指定了外观的边框，并进行裁剪。

绘图
作用域：Any
Modifier.alpha(alpha: Float)
使用可能小于 1 的修饰的 alpha 绘制内容。

作用域：Any
Modifier.background(color: Color, shape: Shape)
使用纯 color 在内容后方绘制 shape。

作用域：Any
Modifier.background(
    brush: Brush,
    shape: Shape,
    alpha: @FloatRange(from = 0.0, to = 1.0) Float
)
使用 brush 在内容后方绘制 shape。

作用域：Any
Modifier.clip(shape: Shape)
将内容裁剪到 shape。

作用域：Any
Modifier.clipToBounds()
将内容裁剪到此修饰符定义的图层的边界。

作用域：Any
Modifier.drawBehind(onDraw: DrawScope.() -> Unit)
绘制到修饰的内容后方的 Canvas 中。

作用域：Any
Modifier.drawWithCache(onBuildDrawCache: CacheDrawScope.() -> DrawResult)
绘制到 DrawScope 中，只要绘制区域的大小不变或读取的任何状态对象未发生变化，便使内容在各绘制调用中始终保留。

作用域：Any
Modifier.drawWithContent(onDraw: ContentDrawScope.() -> Unit)
创建 DrawModifier，允许开发者在布局内容前后进行绘制。

作用域：Any
Modifier.indication(
    interactionSource: InteractionSource,
    indication: Indication?
)
在发生互动时为此组件绘制视觉效果。

作用域：Any
Modifier.paint(
    painter: Painter,
    sizeToIntrinsics: Boolean,
    alignment: Alignment,
    contentScale: ContentScale,
    alpha: Float,
    colorFilter: ColorFilter?
)
使用 painter 绘制内容。

作用域：Any
Modifier.shadow(
    elevation: Dp,
    shape: Shape,
    clip: Boolean,
    ambientColor: Color,
    spotColor: Color
)
创建一个用于绘制阴影的 graphicsLayer。

作用域：Any
Modifier.safeDrawingPadding()
添加内边距，以适应 safe drawing 边衬区。

作用域：Any
Modifier.zIndex(zIndex: Float)
创建一个修饰符，用于控制同一布局父项的子项的绘制顺序。

焦点
作用域：Any
Modifier.onFocusChanged(onFocusChanged: (FocusState) -> Unit)
将此修饰符添加到组件，以观察焦点状态事件。

作用域：Any
Modifier.onFocusEvent(onFocusEvent: (FocusState) -> Unit)
将此修饰符添加到组件，以观察焦点状态事件。

作用域：Any
Modifier. focusModifier()
此函数已废弃。取而代之的是 focusTarget

作用域：Any
Modifier.focusTarget()
将此修饰符添加到组件，以使其可聚焦。

作用域：Any
Modifier. focusOrder(focusOrderReceiver: FocusOrder.() -> Unit)
此函数已废弃。请改用 focusProperties()

作用域：Any
Modifier. focusOrder(focusRequester: FocusRequester)
此函数已废弃。请改用 focusRequester()

作用域：Any
Modifier. focusOrder(
    focusRequester: FocusRequester,
    focusOrderReceiver: FocusOrder.() -> Unit
)
此函数已废弃。请改用 focusProperties() 和 focusRequester()

作用域：Any
Modifier.focusProperties(scope: FocusProperties.() -> Unit)
此修饰符可让您指定可供修饰符链中更底层或子布局节点上的 focusTarget 访问的属性。

作用域：Any
Modifier.focusRequester(focusRequester: FocusRequester)
将此修饰符添加到组件中，以请求更改焦点。

作用域：任意
@ExperimentalComposeUiApi
Modifier.focusRestorer(onRestoreFailed: (() -> FocusRequester)?)
此修饰符可用于保存焦点并将其恢复至焦点小组。

作用域：任意
Modifier.focusGroup()
创建焦点群组或将此组件标记为焦点群组。

作用域：Any
Modifier.focusable(
    enabled: Boolean,
    interactionSource: MutableInteractionSource?
)
将组件配置为可通过焦点系统或无障碍“焦点”事件聚焦。

作用域：Any
@ExperimentalFoundationApi
Modifier.onFocusedBoundsChanged(
    onPositioned: (LayoutCoordinates?) -> Unit
)
每当当前聚焦区域的边界发生变化时，就会调用 onPositioned。

Graphics
作用域：Any
Modifier.graphicsLayer(block: GraphicsLayerScope.() -> Unit)
此 Modifier.Node 可使内容绘制到绘图层。

作用域：Any
Modifier.graphicsLayer(
    scaleX: Float,
    scaleY: Float,
    alpha: Float,
    translationX: Float,
    translationY: Float,
    shadowElevation: Float,
    rotationX: Float,
    rotationY: Float,
    rotationZ: Float,
    cameraDistance: Float,
    transformOrigin: TransformOrigin,
    shape: Shape,
    clip: Boolean,
    renderEffect: RenderEffect?,
    ambientShadowColor: Color,
    spotShadowColor: Color,
    compositingStrategy: CompositingStrategy
)
此 Modifier.Element 可使内容绘制到绘图层。

作用域：Any
Modifier.toolingGraphicsLayer()
此 Modifier.Element 可添加绘制图层，以便工具可以识别所绘制图片中的元素。

键盘
作用域：Any
Modifier.onKeyEvent(onKeyEvent: (KeyEvent) -> Boolean)
将此 modifier 添加到组件的 modifier 参数中后，它可以在自己（或其某个子级）获得焦点时截获硬件按键事件。

作用域：Any
Modifier.onPreviewKeyEvent(onPreviewKeyEvent: (KeyEvent) -> Boolean)
将此 modifier 添加到组件的 modifier 参数中后，它可以在自己（或其某个子级）获得焦点时截获硬件按键事件。

Layout
作用域：任意
Modifier.layoutId(layoutId: String, tag: String?)
支持使用 tag 的 androidx.compose.ui.layout.layoutId 的替代方法。

作用域：任意
Modifier.layoutId(layoutId: Any)
使用 layoutId 标记元素，以在其父项中识别它。

作用域：Any
Modifier.layout(measure: MeasureScope.(Measurable, Constraints) -> MeasureResult)
创建 LayoutModifier，以允许更改已封装元素的测量和布局方式。

作用域：Any
Modifier.onGloballyPositioned(
    onGloballyPositioned: (LayoutCoordinates) -> Unit
)
当内容的全局位置可能发生变化时，使用元素的 LayoutCoordinates 调用 onGloballyPositioned。

内边距
作用域：Any
Modifier.paddingFrom(alignmentLine: AlignmentLine, before: Dp, after: Dp)
此 Modifier 可添加内边距，以根据从内容边界到 alignment line 的指定距离放置内容。

作用域：Any
Modifier.paddingFrom(
    alignmentLine: AlignmentLine,
    before: TextUnit,
    after: TextUnit
)
此 Modifier 可添加内边距，以根据从内容边界到 alignment line 的指定距离放置内容。

作用域：Any
Modifier.paddingFromBaseline(top: Dp, bottom: Dp)
此 Modifier 用于在布局中放置内容，使从布局顶部到 baseline of the first line of text in the content 的距离为 top，且从 baseline of the last line of text in the content 到布局底部的距离为 bottom。

作用域：Any
Modifier.paddingFromBaseline(top: TextUnit, bottom: TextUnit)
此 Modifier 用于在布局中放置内容，使从布局顶部到 baseline of the first line of text in the content 的距离为 top，且从 baseline of the last line of text in the content 到布局底部的距离为 bottom。

作用域：Any
Modifier.absolutePadding(left: Dp, top: Dp, right: Dp, bottom: Dp)
沿着内容的每个边缘应用 Dp 的额外空间：left、top、right 和 bottom。

作用域：任意
Modifier.padding(all: Dp)
沿着内容的每个边缘（左侧、顶部、右侧和底部）应用 all dp 的额外空间。

作用域：Any
Modifier.padding(paddingValues: PaddingValues)
向组件应用 PaddingValues，将其作为沿内容左侧、顶部、右侧和底部的额外空间。

作用域：任意
Modifier.padding(horizontal: Dp, vertical: Dp)
沿着内容的左侧和右侧边缘应用 horizontal dp 空间，并沿着顶部和底部边缘应用 vertical dp 空间。

作用域：任意
Modifier.padding(start: Dp, top: Dp, end: Dp, bottom: Dp)
沿着内容的每个边缘应用 Dp 的额外空间：start、top、end 和 bottom。

作用域：任意
Modifier.captionBarPadding()
添加内边距，以适应 caption bar 边衬区。

作用域：Any
Modifier.displayCutoutPadding()
添加内边距，以适应 display cutout。

作用域：Any
Modifier.imePadding()
添加内边距，以适应 ime 边衬区。

作用域：Any
Modifier.mandatorySystemGesturesPadding()
添加内边距，以适应 mandatory system gestures 边衬区。

作用域：Any
Modifier.navigationBarsPadding()
添加内边距，以适应 navigation bars 边衬区。

作用域：Any
Modifier.safeContentPadding()
添加内边距，以适应 safe content 边衬区。

作用域：Any
Modifier.safeGesturesPadding()
添加内边距，以适应 safe gestures 边衬区。

作用域：Any
Modifier.statusBarsPadding()
添加内边距，以适应 status bars 边衬区。

作用域：Any
Modifier.systemBarsPadding()
添加内边距，以适应 system bars 边衬区。

作用域：Any
Modifier.systemGesturesPadding()
添加内边距，以适应 system gestures 边衬区。

作用域：Any
Modifier.waterfallPadding()
添加内边距，以适应 waterfall 边衬区。

作用域：Any
Modifier.windowInsetsPadding(insets: WindowInsets)
添加内边距，使内容不会进入 insets 空间。

Pointer
作用域：任意
Modifier.pointerHoverIcon(
    icon: PointerIcon,
    overrideDescendants: Boolean
)
可让开发者定义光标悬停在元素上时要显示的指针图标的修饰符。

作用域：任意
@ExperimentalComposeUiApi
Modifier.pointerInteropFilter(
    requestDisallowInterceptTouchEvent: RequestDisallowInterceptTouchEvent?,
    onTouchEvent: (MotionEvent) -> Boolean
)
一个特殊的 PointerInputModifier，可提供对最初分派到 Compose 的底层 MotionEvent 的访问权限。

作用域：任意
@ExperimentalComposeUiApi
Modifier.pointerMoveFilter(
    onMove: (position: Offset) -> Boolean,
    onExit: () -> Boolean,
    onEnter: () -> Boolean
)
允许跟踪指针（即鼠标或触控板）移动事件的修饰符。

作用域：任意
Modifier. pointerInput(block: suspend PointerInputScope.() -> Unit)
此函数已废弃。Modifier.pointerInput 必须提供一个或多个“key”参数来定义此修饰符的身份，以及确定应在何时取消它的上一个输入处理协程并为新键启动新效果。

作用域：Any
Modifier.pointerInput(key1: Any?, block: suspend PointerInputScope.() -> Unit)
创建一个用于在修饰的元素区域内处理指针输入的修饰符。

作用域：Any
Modifier.pointerInput(vararg keys: Any?, block: suspend PointerInputScope.() -> Unit)
创建一个用于在修饰的元素区域内处理指针输入的修饰符。

作用域：Any
Modifier.pointerInput(key1: Any?, key2: Any?, block: suspend PointerInputScope.() -> Unit)
创建一个用于在修饰的元素区域内处理指针输入的修饰符。

位置
作用域：任意
Modifier.absoluteOffset(offset: Density.() -> IntOffset)
将内容偏移 offset 像素。

作用域：Any
Modifier.absoluteOffset(x: Dp, y: Dp)
将内容偏移（x dp，y dp）。

作用域：Any
Modifier.offset(offset: Density.() -> IntOffset)
将内容偏移 offset 像素。

作用域：Any
Modifier.offset(x: Dp, y: Dp)
将内容偏移（x dp，y dp）。

作用域： TabRowDefaults
Modifier.tabIndicatorOffset(currentTabPosition: TabPosition)
此 Modifier 会占用 TabRow 内的所有可用宽度，然后以动画方式呈现它应用到指示器的偏移量（具体取决于 currentTabPosition）。

作用域： TabRowDefaults
Modifier.tabIndicatorOffset(currentTabPosition: TabPosition)
此 Modifier 会占用 TabRow 内的所有可用宽度，然后以动画方式呈现它应用到指示器的偏移量（具体取决于 currentTabPosition）。

语义
作用域：任意
Modifier.progressSemantics()
包含不确定性进度指示器所需的 semantics，它表示操作正在进行中。

作用域：任意
Modifier.progressSemantics(
    value: Float,
    valueRange: ClosedFloatingPointRange<Float>,
    steps: @IntRange(from = 0) Int
)
包含确定性进度指示器或滑块的进度部分所需的 semantics，它表示 valueRange 内的进度。

作用域：任意
Modifier.rangeSemantics(
    value: Float,
    enabled: Boolean,
    onValueChange: (Float) -> Unit,
    valueRange: ClosedFloatingPointRange<Float>,
    steps: Int
)
用于添加表示步进器/滑块进度的语义的修饰符。

作用域：任意
Modifier.clearAndSetSemantics(properties: SemanticsPropertyReceiver.() -> Unit)
清除所有后代节点的语义并设置新语义。

作用域：Any
Modifier.semantics(mergeDescendants: Boolean, properties: SemanticsPropertyReceiver.() -> Unit)
将语义键值对添加到布局节点，以便用于测试、无障碍功能等。

滚动
作用域：Any
Modifier.clipScrollableContainer(orientation: Orientation)
裁剪可滚动容器在主轴上的边界，同时在交叉轴上为背景效果（例如阴影）留出空间。

作用域：任意
Modifier. mouseScrollFilter(
    onMouseScroll: (event: MouseScrollEvent, bounds: IntSize) -> Boolean
)
此函数已废弃。使用 Modifier.pointerInput + PointerEventType.Scroll

作用域：任意
Modifier.nestedScroll(
    connection: NestedScrollConnection,
    dispatcher: NestedScrollDispatcher?
)
修改元素，以使其参与嵌套滚动层次结构。

作用域：任意
@ExperimentalFoundationApi
Modifier.overscroll(overscrollEffect: OverscrollEffect)
通过提供的 overscrollEffect 渲染滚动回弹。

作用域：任意
Modifier.onPreRotaryScrollEvent(
    onPreRotaryScrollEvent: (RotaryScrollEvent) -> Boolean
)
将此 modifier 添加到组件的 modifier 参数中后，它可以在自身（或其子项之一）获得焦点时拦截 RotaryScrollEvent。

作用域：Any
Modifier.onRotaryScrollEvent(
    onRotaryScrollEvent: (RotaryScrollEvent) -> Boolean
)
将此 modifier 添加到组件的 modifier 参数中后，它可以在自己（或其某个子级）获得焦点时截获 RotaryScrollEvent。

作用域：任意
Modifier.rotaryScrollable(
    behavior: RotaryScrollableBehavior,
    focusRequester: FocusRequester,
    reverseDirection: Boolean
)
用于将旋转事件与可滚动容器（例如 Column、LazyList 等）相关联的修饰符。

作用域：任意
Modifier.scrollAway(scrollState: ScrollState, offset: Dp)
根据 ScrollState 使项目垂直滚入/滚出视图。

作用域：Any
Modifier.scrollAway(
    scrollState: LazyListState,
    itemIndex: Int,
    offset: Dp
)
根据 LazyListState 使项目垂直滚入/滚出视图。

作用域：Any
Modifier.scrollAway(
    scrollState: ScalingLazyListState,
    itemIndex: Int,
    offset: Dp
)
根据 ScalingLazyListState 使项目垂直滚入/滚出视图。

作用域：任意
Modifier. scrollAway(
    scrollState: ScalingLazyListState,
    itemIndex: Int,
    offset: Dp
)
此函数已废弃。此过载提供是为了向后兼容 Compose for Wear OS 1.1。我们提供了一个较新的过载，它使用 wear.compose.foundation.lazy 软件包中的 ScalingLazyListState

作用域：任意
Modifier.horizontalScroll(
    state: ScrollState,
    enabled: Boolean,
    flingBehavior: FlingBehavior?,
    reverseScrolling: Boolean
)
修改元素，以支持在内容的宽度大于允许的最大约束时水平滚动。

作用域：Any
Modifier.verticalScroll(
    state: ScrollState,
    enabled: Boolean,
    flingBehavior: FlingBehavior?,
    reverseScrolling: Boolean
)
修改元素，以支持在内容的高度大于允许的最大约束值时垂直滚动。

作用域：Any
Modifier.scrollable(
    state: ScrollableState,
    orientation: Orientation,
    enabled: Boolean,
    reverseDirection: Boolean,
    flingBehavior: FlingBehavior?,
    interactionSource: MutableInteractionSource?
)
在单个 Orientation 为界面元素配置触摸滚动和快速滑动。

作用域：Any
@ExperimentalFoundationApi
Modifier.scrollable(
    state: ScrollableState,
    orientation: Orientation,
    overscrollEffect: OverscrollEffect?,
    enabled: Boolean,
    reverseDirection: Boolean,
    flingBehavior: FlingBehavior?,
    interactionSource: MutableInteractionSource?,
    bringIntoViewSpec: BringIntoViewSpec?
)
在单个 Orientation 为界面元素配置触摸滚动和快速滑动。

作用域：任意
@ExperimentalTvFoundationApi
Modifier. scrollableWithPivot(
    state: ScrollableState,
    orientation: Orientation,
    pivotOffsets: PivotOffsets,
    enabled: Boolean,
    reverseDirection: Boolean
)
此函数已废弃。scrollableWithPivot 已废弃。

作用域：任意
@ExperimentalLayoutApi
Modifier.imeNestedScroll()
在 Android R 及更高版本上将软键盘控制为嵌套滚动。

大小
作用域：Any
Modifier.animateContentSize(
    animationSpec: FiniteAnimationSpec<IntSize>,
    finishedListener: ((initialValue: IntSize, targetValue: IntSize) -> Unit)?
)
此修饰符会在其子修饰符（或子可组合项，如果它已位于链尾）更改尺寸时为其自己的尺寸添加动画效果。

作用域：任意
Modifier.animateContentSize(
    animationSpec: FiniteAnimationSpec<IntSize>,
    alignment: Alignment,
    finishedListener: ((initialValue: IntSize, targetValue: IntSize) -> Unit)?
)
此修饰符会在其子修饰符（或子可组合项，如果它已位于链尾）更改尺寸时为其自己的尺寸添加动画效果。

作用域：Any
Modifier.aspectRatio(
    ratio: @FloatRange(from = 0.0, fromInclusive = false) Float,
    matchHeightConstraintsFirst: Boolean
)
尝试按以下顺序匹配传入约束条件之一，尝试通过尝试匹配指定的宽高比来调整内容的大小：Constraints.maxWidth、Constraints.maxHeight、Constraints.minWidth、Constraints.minHeight（如果 matchHeightConstraintsFirst 为 false（默认值））；或 Constraints.maxHeight、Constraints.maxWidth、Constraints.minHeight、Constraints.minWidth（如果 matchHeightConstraintsFirst 为 true）。

作用域：任意
Modifier.minimumInteractiveComponentSize()
预留至少 48dp 的大小，以便在元素尺寸较小时消除触摸交互的歧义。

作用域：任意
Modifier.minimumInteractiveComponentSize()
预留至少 48dp 的大小，以便在元素尺寸较小时消除触摸交互的歧义。

作用域：任意
Modifier.minimumInteractiveComponentSize()
预留至少 48dp 的大小，以便在元素尺寸较小时消除触摸交互的歧义。

作用域：任意
Modifier.minimumInteractiveComponentSize()
预留至少 48dp 的大小，以便在元素尺寸较小时消除触摸交互的歧义。

作用域：任意
Modifier.height(intrinsicSize: IntrinsicSize)
将内容的首选高度声明为与内容的最小或最大固有高度相同。

作用域：Any
Modifier.requiredHeight(intrinsicSize: IntrinsicSize)
将内容的高度声明为与内容的最小或最大固有高度完全相同。

作用域：Any
Modifier.requiredWidth(intrinsicSize: IntrinsicSize)
将内容的宽度声明为与内容的最小或最大固有宽度完全相同。

作用域：Any
Modifier.width(intrinsicSize: IntrinsicSize)
将内容的首选宽度声明为与内容的最小或最大固有宽度相同。

作用域：Any
Modifier.onSizeChanged(onSizeChanged: (IntSize) -> Unit)
首次测量元素时或元素的大小发生变化时，通过修改后的 Compose 界面元素的大小进行调用。

作用域：Any
Modifier.defaultMinSize(minWidth: Dp, minHeight: Dp)
仅在封装的布局不受约束时约束其尺寸：minWidth 和 minHeight 约束仅在传入的相应约束为 0 时应用。

作用域：Any
Modifier.fillMaxHeight(fraction: @FloatRange(from = 0.0, to = 1.0) Float)
让内容填充（可能仅部分填充）传入的测量约束的 Constraints.maxHeight，方法是将 minimum height 和 maximum height 设置为等于 maximum height 乘以 fraction 得出的值。

作用域：Any
Modifier.fillMaxSize(fraction: @FloatRange(from = 0.0, to = 1.0) Float)
让内容填充（可能仅部分填充）传入的测量约束的 Constraints.maxWidth 和 Constraints.maxHeight，方法是将 minimum width 和 maximum width 设置为等于 maximum width 乘以 fraction 得出的值，并将 minimum height 和 maximum height 设置为等于 maximum height 乘以 fraction 得出的值。

作用域：Any
Modifier.fillMaxWidth(fraction: @FloatRange(from = 0.0, to = 1.0) Float)
让内容填充（可能仅部分填充）传入的测量约束的 Constraints.maxWidth，方法是将 minimum width 和 maximum width 设置为等于 maximum width 乘以 fraction 得出的值。

作用域：Any
Modifier.height(height: Dp)
将内容的首选高度声明为正好 heightdp。

作用域：Any
Modifier.heightIn(min: Dp, max: Dp)
如果传入的测量 Constraints 允许，将内容的高度限制在 mindp 与 maxdp 之间。

作用域：Any
Modifier.requiredHeight(height: Dp)
将内容的高度声明为正好 heightdp。

作用域：Any
Modifier.requiredHeightIn(min: Dp, max: Dp)
将内容的高度限制在 mindp 与 maxdp 之间。

作用域：Any
Modifier.requiredSize(size: Dp)
将内容的尺寸声明为宽度和高度正好是 sizedp。

作用域：任意
Modifier.requiredSize(size: DpSize)
将内容的尺寸声明为正好 size。

作用域：任意
Modifier.requiredSize(width: Dp, height: Dp)
将内容的尺寸声明为正好是 widthdp 和 heightdp。

作用域：任意
Modifier.requiredSizeIn(
    minWidth: Dp,
    minHeight: Dp,
    maxWidth: Dp,
    maxHeight: Dp
)
将内容的宽度限制在 minWidthdp 与 maxWidthdp 之间，并将内容的高度限制在 minHeightdp 与 maxHeightdp 之间。

作用域：Any
Modifier.requiredWidth(width: Dp)
将内容的宽度声明为正好 widthdp。

作用域：Any
Modifier.requiredWidthIn(min: Dp, max: Dp)
将内容的宽度限制在 mindp 与 maxdp 之间。

作用域：Any
Modifier.size(size: Dp)
将内容的首选尺寸声明为正好是 sizedp 的方形。

作用域：任意
Modifier.size(size: DpSize)
将内容的首选尺寸声明为正好 size。

作用域：任意
Modifier.size(width: Dp, height: Dp)
将内容的首选尺寸声明为正好是 widthdp x heightdp。

作用域：任意
Modifier.sizeIn(minWidth: Dp, minHeight: Dp, maxWidth: Dp, maxHeight: Dp)
如果传入的测量 Constraints 允许，将内容的宽度限制在 minWidthdp 与 maxWidthdp 之间，并将内容的高度限制在 minHeightdp 与 maxHeightdp 之间。

作用域：Any
Modifier.width(width: Dp)
将内容的首选宽度声明为正好 widthdp。

作用域：Any
Modifier.widthIn(min: Dp, max: Dp)
如果传入的测量 Constraints 允许，将内容的宽度限制在 mindp 与 maxdp 之间。

作用域：Any
Modifier.wrapContentHeight(
    align: Alignment.Vertical,
    unbounded: Boolean
)
允许在不考虑传入的测量 minimum height constraint 的情况下将内容测量为所需高度；如果 unbounded 为 true，则也可不考虑传入的测量 maximum height constraint。

作用域：Any
Modifier.wrapContentSize(align: Alignment, unbounded: Boolean)
允许在不考虑传入的测量 minimum width 或 minimum height 约束的情况下将内容测量为所需大小；如果 unbounded 为 true，则也可不考虑传入的最大约束。

作用域：Any
Modifier.wrapContentWidth(
    align: Alignment.Horizontal,
    unbounded: Boolean
)
允许在不考虑传入的测量 minimum width constraint 的情况下以所需宽度测量内容；如果 unbounded 为 true，则也可不考虑传入的测量 maximum width constraint。

作用域：任意
Modifier.touchTargetAwareSize(size: Dp)
用于同时为 IconButton 和 TextButton 设置大小和建议的触摸目标的修饰符。

作用域：任意
Modifier.windowInsetsBottomHeight(insets: WindowInsets)
将高度设置为屏幕 bottom 处 insets 的高度。

作用域：Any
Modifier.windowInsetsEndWidth(insets: WindowInsets)
根据 LayoutDirection，使用 left 或 right 将宽度设置为屏幕 end 处 insets 的宽度。

作用域：Any
Modifier.windowInsetsStartWidth(insets: WindowInsets)
根据 LayoutDirection，使用 left 或 right 将宽度设置为屏幕 start 处 insets 的宽度。

作用域：Any
Modifier.windowInsetsTopHeight(insets: WindowInsets)
将高度设置为屏幕 top 处 insets 的高度。

作用域： RowScope
Modifier.weight(
    weight: @FloatRange(from = 0.0, fromInclusive = false) Float,
    fill: Boolean
)
根据元素相对于 Row 中其他加权同级元素的 weight，按比例调整元素的宽度。

作用域： ColumnScope
Modifier.weight(
    weight: @FloatRange(from = 0.0, fromInclusive = false) Float,
    fill: Boolean
)
根据元素相对于 Column 中其他加权同级元素的 weight，按比例调整元素的高度。

作用域： BoxScope
Modifier.matchParentSize()
在测量完所有其他内容元素后，调整元素的尺寸，使其与 Box 的尺寸一致。

作用域： LazyItemScope
Modifier.fillParentMaxHeight(
    fraction: @FloatRange(from = 0.0, to = 1.0) Float
)
让内容填充传入的测量约束的 Constraints.maxHeight，方法是将 minimum height 设置为等于 maximum height 乘以 fraction 得出的值。

作用域： LazyItemScope
Modifier.fillParentMaxSize(
    fraction: @FloatRange(from = 0.0, to = 1.0) Float
)
让内容填充父级测量约束的 Constraints.maxWidth 和 Constraints.maxHeight，方法是将 minimum width 设置为等于 maximum width 乘以 fraction 得出的值，并将 minimum height 设置为等于 maximum height 乘以 fraction 得出的值。

作用域： LazyItemScope
Modifier.fillParentMaxWidth(
    fraction: @FloatRange(from = 0.0, to = 1.0) Float
)
让内容填充父级测量约束的 Constraints.maxWidth，方法是将 minimum width 设置为等于 maximum width 乘以 fraction 得出的值。

作用域： ExposedDropdownMenuBoxScope
abstract	
Modifier.exposedDropdownSize(matchTextFieldWidth: Boolean)
应应用于放置在作用域内的 ExposedDropdownMenu 的修饰符。

作用域： ExposedDropdownMenuBoxScope
abstract	
Modifier.exposedDropdownSize(matchTextFieldWidth: Boolean)
应应用于放置在 ExposedDropdownMenuBoxScope 内的菜单的修饰符。

测试
作用域：Any
Modifier.testTag(tag: String)
应用标记以允许在测试中找到修饰的元素。

变换
作用域：Any
Modifier.rotate(degrees: Float)
设置视图围绕可组合项中心旋转的角度。

作用域：任意
Modifier.scale(scale: Float)
按相同的缩放比例沿水平轴和垂直轴均匀缩放内容。

作用域：任意
Modifier.scale(scaleX: Float, scaleY: Float)
分别按以下缩放比例沿水平轴和垂直轴缩放可组合项的内容。

作用域：任意
Modifier.transformable(
    state: TransformableState,
    lockRotationOnZoomPan: Boolean,
    enabled: Boolean
)
启用修饰的界面元素的变换手势。

作用域：任意
@ExperimentalFoundationApi
Modifier.transformable(
    state: TransformableState,
    canPan: (Offset) -> Boolean,
    lockRotationOnZoomPan: Boolean,
    enabled: Boolean
)
启用修饰的界面元素的变换手势。

其他
作用域：任意
@ExperimentalFoundationApi
Modifier.dragAndDropSource(block: suspend DragAndDropSourceScope.() -> Unit)
一种修饰符，可将应用于它的元素视为拖放操作的来源。

作用域：任意
Modifier.basicMarquee(
    iterations: Int,
    animationMode: MarqueeAnimationMode,
    repeatDelayMillis: Int,
    initialDelayMillis: Int,
    spacing: MarqueeSpacing,
    velocity: Dp
)
如果修改后的内容宽度超出可用空间，则会对其应用动画 Marquee 效果。

作用域：任意
Modifier.edgeSwipeToDismiss(
    swipeToDismissBoxState: SwipeToDismissBoxState,
    edgeWidth: Dp
)
将滑动关闭操作限制为只能从视口边缘开始操作。

作用域：任意
Modifier.blur(radius: Dp, edgeTreatment: BlurredEdgeTreatment)
绘制内容，并使用指定的半径模糊处理内容。

作用域：Any
Modifier.blur(
    radiusX: Dp,
    radiusY: Dp,
    edgeTreatment: BlurredEdgeTreatment
)
绘制内容，并使用指定的半径模糊处理内容。

作用域：任意
@ExperimentalFoundationApi
Modifier.bringIntoViewRequester(
    bringIntoViewRequester: BringIntoViewRequester
)
可用于发送 scrollIntoView 请求的修饰符。

作用域：任意
@ExperimentalFoundationApi
Modifier.bringIntoViewResponder(responder: BringIntoViewResponder)
一个父级，可以响应其子级的 BringIntoViewRequester 请求，并可以滚动以在屏幕上显示项目。

作用域：Any
Modifier.composed(
    inspectorInfo: InspectorInfo.() -> Unit,
    factory: @Composable Modifier.() -> Modifier
)
声明将针对所修饰的每个元素进行组合的 Modifier 的即时组合。

作用域：Any
@ExperimentalComposeUiApi
Modifier.composed(
    fullyQualifiedName: String,
    key1: Any?,
    inspectorInfo: InspectorInfo.() -> Unit,
    factory: @Composable Modifier.() -> Modifier
)
声明将针对所修饰的每个元素进行组合的 Modifier 的即时组合。

作用域：Any
@ExperimentalComposeUiApi
Modifier.composed(
    fullyQualifiedName: String,
    vararg keys: Any?,
    inspectorInfo: InspectorInfo.() -> Unit,
    factory: @Composable Modifier.() -> Modifier
)
声明将针对所修饰的每个元素进行组合的 Modifier 的即时组合。

作用域：Any
@ExperimentalComposeUiApi
Modifier.composed(
    fullyQualifiedName: String,
    key1: Any?,
    key2: Any?,
    inspectorInfo: InspectorInfo.() -> Unit,
    factory: @Composable Modifier.() -> Modifier
)
声明将针对所修饰的每个元素进行组合的 Modifier 的即时组合。

作用域：Any
@ExperimentalComposeUiApi
Modifier.composed(
    fullyQualifiedName: String,
    key1: Any?,
    key2: Any?,
    key3: Any?,
    inspectorInfo: InspectorInfo.() -> Unit,
    factory: @Composable Modifier.() -> Modifier
)
声明将针对所修饰的每个元素进行组合的 Modifier 的即时组合。

作用域：任意
@ExperimentalFoundationApi
Modifier.dragAndDropSource(
    drawDragDecoration: DrawScope.() -> Unit,
    block: suspend DragAndDropSourceScope.() -> Unit
)
一种修饰符，可将应用于它的元素视为拖放操作的来源。

作用域：任意
@ExperimentalFoundationApi
Modifier.dragAndDropTarget(
    shouldStartDragAndDrop: (startEvent: DragAndDropEvent) -> Boolean,
    target: DragAndDropTarget
)
一种修饰符，用于接收拖放手势。

作用域：任意
Modifier. excludeFromSystemGesture()
此函数已废弃。请使用 systemGestureExclusion

作用域：Any
Modifier. excludeFromSystemGesture(
    exclusion: (LayoutCoordinates) -> Rect
)
此函数已废弃。请使用 systemGestureExclusion

作用域：任意
Modifier.handwritingDetector(callback: () -> Unit)
将某个元素配置为手写检测器，用于检测触控笔手写内容并将识别出文本的处理委托给另一个元素。

作用域：任意
Modifier.handwritingHandler()
将元素配置为触控笔手写处理程序，该处理程序可以处理由手写检测器上的触控笔手写触发的手写会话中的文本输入。

作用域：任意
Modifier.hoverable(
    interactionSource: MutableInteractionSource,
    enabled: Boolean
)
将组件配置为可通过指针进入/退出事件悬停。

作用域：任意
inline	
Modifier. inspectable(
    noinline inspectorInfo: InspectorInfo.() -> Unit,
    factory: Modifier.() -> Modifier
)
此函数已废弃。此 API 会导致过多地使修饰符失效，因此不建议使用。

作用域：任意
Modifier.approachLayout(
    isMeasurementApproachInProgress: (lookaheadSize: IntSize) -> Boolean,
    isPlacementApproachInProgress: Placeable.PlacementScope.(lookaheadCoordinates: LayoutCoordinates) -> Boolean,
    approachMeasure: ApproachMeasureScope.(measurable: Measurable, constraints: Constraints) -> MeasureResult
)
创建一个过渡布局，以帮助逐步接近在预测性传递中计算的目标布局。

作用域：任意
Modifier.magnifier(
    sourceCenter: Density.() -> Offset,
    magnifierCenter: (Density.() -> Offset)?,
    onSizeChanged: ((DpSize) -> Unit)?,
    zoom: Float,
    size: DpSize,
    cornerRadius: Dp,
    elevation: Dp,
    clip: Boolean
)
显示 Magnifier widget，以显示相对于当前布局节点而言在 sourceCenter 处内容的放大版。

作用域：Any
@ExperimentalComposeUiApi
Modifier.modifierLocalConsumer(consumer: ModifierLocalReadScope.() -> Unit)
该修饰符可用于使用由布局树中的其他修饰符（位于此修饰符的左侧或上方）提供的 ModifierLocal。

作用域：Any
@ExperimentalComposeUiApi
<T : Any?> Modifier.modifierLocalProvider(
    key: ProvidableModifierLocal<T>,
    value: () -> T
)
该修饰符可用于提供可被其他修饰符（位于该修饰符的右侧或者是该修饰符附加到的布局节点的子项）读取的 ModifierLocal。

作用域：Any
Modifier.onPlaced(onPlaced: (LayoutCoordinates) -> Unit)
在放置父级 LayoutModifier 和父布局之后以及放置子级 LayoutModifier 之前，调用 onPlaced。

作用域：任意
@ExperimentalWearMaterialApi
@Composable
Modifier.placeholder(
    placeholderState: PlaceholderState,
    shape: Shape,
    color: Color
)
在可组合项的顶部绘制一个占位符形状，并为擦除效果添加动画效果以移除占位符。

作用域：任意
@ExperimentalWearMaterialApi
@Composable
Modifier.placeholderShimmer(
    placeholderState: PlaceholderState,
    shape: Shape,
    color: Color
)
用于在组件上绘制占位闪烁效果的修饰符。

作用域：任意
@ExperimentalComposeUiApi
Modifier.motionEventSpy(watcher: (motionEvent: MotionEvent) -> Unit)
使用布局区域或任何子级 pointerInput 接收的每个 MotionEvent 调用 watcher。

作用域：任意
Modifier.preferKeepClear()
将布局矩形标记为首选避开悬浮窗口。

作用域：任意
Modifier.preferKeepClear(rectProvider: (LayoutCoordinates) -> Rect)
在局部布局坐标中标记一个矩形，以避免浮动窗口。

作用域：任意
@ExperimentalMaterialApi
Modifier.pullRefreshIndicatorTransform(
    state: PullRefreshState,
    scale: Boolean
)
此修饰符用于转换位置，并根据给定的 PullRefreshState 缩放下拉刷新指示器的大小。

作用域：任意
@ExperimentalMaterialApi
Modifier.pullRefresh(state: PullRefreshState, enabled: Boolean)
一种嵌套滚动修饰符，用于向 state 提供滚动事件。

作用域：任意
@ExperimentalMaterialApi
Modifier.pullRefresh(
    onPull: (pullDelta: Float) -> Float,
    onRelease: suspend (flingVelocity: Float) -> Float,
    enabled: Boolean
)
嵌套滚动修饰符，可提供 onPull 和 onRelease 回调，以帮助构建自定义拉取刷新组件。

作用域：任意
@ExperimentalMaterial3Api
Modifier.pullToRefresh(
    isRefreshing: Boolean,
    state: PullToRefreshState,
    enabled: Boolean,
    threshold: Dp,
    onRefresh: () -> Unit
)
一种修饰符，用于向容器添加嵌套滚动，以支持下拉刷新手势。

作用域：任意
@ExperimentalMaterial3Api
Modifier.pullToRefreshIndicator(
    state: PullToRefreshState,
    isRefreshing: Boolean,
    threshold: Dp,
    shape: Shape,
    containerColor: Color,
    elevation: Dp
)
用于处理下拉刷新指示器的大小、偏移、剪裁、阴影和背景绘制的修饰符，在实现自定义指示器时非常有用。

作用域：任意
@ExperimentalFoundationApi
Modifier.contentReceiver(
    receiveContentListener: ReceiveContentListener
)
将当前节点和所有子节点配置为内容接收器。

作用域：任意
@ExperimentalComposeUiApi
Modifier.onInterceptKeyBeforeSoftKeyboard(
    onInterceptKeyBeforeSoftKeyboard: (KeyEvent) -> Boolean
)
将此 modifier 添加到组件的 modifier 参数中后，它可以在硬件按键事件发送到软件键盘之前截获这些事件。

作用域：任意
@ExperimentalComposeUiApi
Modifier.onPreInterceptKeyBeforeSoftKeyboard(
    onPreInterceptKeyBeforeSoftKeyboard: (KeyEvent) -> Boolean
)
将此 modifier 添加到组件的 modifier 参数中后，它可以在硬件按键事件发送到软件键盘之前截获这些事件。

作用域：任意
Modifier. edgeSwipeToDismiss(
    swipeToDismissBoxState: SwipeToDismissBoxState,
    edgeWidth: Dp
)
此函数已废弃。SwipeToDismiss 已迁移到 androidx.wear.compose.foundation。

作用域：任意
Modifier.systemGestureExclusion()
从系统手势中排除布局矩形。

作用域：Any
Modifier.systemGestureExclusion(exclusion: (LayoutCoordinates) -> Rect)
从系统手势中排除局部布局坐标中的矩形。

作用域：Any
Modifier.consumeWindowInsets(insets: WindowInsets)
使用尚未被类似于 windowInsetsPadding 的其他边衬区修饰符使用的边衬区，而无需添加任何内边距。

作用域：Any
Modifier.consumeWindowInsets(paddingValues: PaddingValues)
将 paddingValues 用作边衬区，就像添加了内边距而不考虑边衬区。

作用域：Any
Modifier.onConsumedWindowInsetsChanged(
    block: (consumedWindowInsets: WindowInsets) -> Unit
)
使用由 consumeWindowInsets 或其中一个内边距修饰符（如 imePadding）使用的 WindowInsets 调用 block。

作用域： TooltipBoxScope
Modifier.tooltipTrigger()
在长按锚点可组合项后显示提示时，应应用于锚点可组合项的 Modifier。

作用域： ExposedDropdownMenuBoxScope
Modifier. menuAnchor()
此函数已废弃。使用接受 MenuAnchorType 和 enabled 参数的重载

作用域： ExposedDropdownMenuBoxScope
abstract	
Modifier.menuAnchor(type: MenuAnchorType, enabled: Boolean)
应应用于 ExposedDropdownMenuBoxScope 内的元素（通常是文本字段或文本字段中的图标）的修饰符。