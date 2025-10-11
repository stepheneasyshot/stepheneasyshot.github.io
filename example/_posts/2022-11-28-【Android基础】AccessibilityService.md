---
layout: post
description: > 
  本文介绍了AccessibilityService的使用方法
image: 
  path: /assets/img/blog/blogs_android_access_service_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_access_service_cover.png
    960w:  /assets/img/blog/blogs_android_access_service_cover.png
    480w:  /assets/img/blog/blogs_android_access_service_cover.png
accent_image: /assets/img/blog/blogs_android_access_service_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android基础】AccessibilityService 无障碍服务使用
**AccessibilityService** 是 Android 系统提供的一种特殊类型的服务，它允许应用程序监听系统和应用中的各种事件，并与用户界面（UI）进行交互。

它最初设计目的是为了帮助有视觉、听觉或运动障碍的用户更有效地使用 Android 设备。例如，它可以朗读屏幕上的内容、响应特定的手势或提供自定义的导航。

但由于其强大的能力，它也被广泛用于实现**自动化任务**、**屏幕内容监控**和**手势模拟**等高级功能。

## 核心能力和工作原理
继承自 `AccessibilityService` ，实现的服务中使用最多的有以下核心功能：
### 1. 监听事件
它可以监听系统和应用发出的各种可访问性事件（AccessibilityEvent），这些事件包括：
* **窗口状态变化：** 例如，新窗口打开、关闭或聚焦变化。
* **视图内容变化：** 例如，文本框中的文字被修改、列表中的项目被添加。
* **焦点变化：** 当用户或程序将焦点移动到不同的 UI 元素时。
* **通知栏变化：** 接收和处理通知栏的发布、更新和移除事件。

### 2. 访问屏幕内容
服务可以获取到屏幕上当前活动窗口的 **视图层次结构**（AccessibilityNodeInfo）。通过这个节点信息，服务可以获取任何可见 TextView、EditText 或其他可访问元素上的文本内容。还能通过资源 ID（例如 `com.example.app:id/button_ok`）定位特定的 UI 元素。

### 3\. 模拟用户操作

这是 `AccessibilityService` 最强大的功能之一，它允许服务代替用户执行操作，实现自动化：
* **点击和长按：** 模拟点击任何可点击的 UI 元素。
* **输入文本：** 填充 EditText 字段。
* **滚动：** 向上、向下、向左或向右滚动可滚动的视图。
* **执行全局操作：** 例如，返回（BACK）、主页（HOME）、打开最近任务列表或显示通知栏。
* **模拟手势：** 在屏幕的任意坐标上模拟复杂的触摸手势，如滑动（Swipe）。

## 开发实践
以一个自动跳广告的Demo为例，展示如何使用 `AccessibilityService` 实现自动化任务。
### 创建服务类
首先定义服务类 `AutoSkipAdsService` ，继承自 `AccessibilityService` ：

```kotlin
class AutoSkipAdsService : AccessibilityService()
```

这时候会自动要求实现以下方法：

```kotlin
override fun onAccessibilityEvent(event: AccessibilityEvent) {
    // 处理事件，例如查找广告并点击跳过按钮
}

override fun onInterrupt() {
    // 服务被中断时回调，例如用户关闭了服务
}
```

一般来说，还需要在 `onCreate()` 中设置为前台服务，以提示用户服务正在运行。

```kotlin
override fun onCreate() {
    super.onCreate()
    // 创建前台通知，提示用户服务正在运行
    val notification = NotificationCompat.Builder(this, CHANNEL_ID)
        .setContentTitle(getString(R.string.accessibility_service_title))
        .setContentText(getString(R.string.accessibility_service_description))
        .setSmallIcon(R.drawable.ic_accessibility)
        .build()
    // 启动前台服务，显示通知
    startForeground(NOTIFICATION_ID, notification)
}
```

### 配置 Manifest 声明

在应用的 `AndroidManifest.xml` 文件中声明 `AccessibilityService`：

```xml
<service
    android:description="@string/description_in_manifest"
    android:exported="true"
    android:foregroundServiceType="mediaPlayback"
    android:label="自动跳过广告"
    android:name=".service.AutoSkipAdsService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    tools:ignore="ForegroundServicePermission">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>

    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_config" />
</service>
```

其中最重要的配置就是 `accessibility_config.xml` ，它定义了服务的行为和关注的事件类型。关键的配置项：

| 属性 | 描述 |
| :--- | :--- |
| `accessibilityEventTypes` | 服务的关注事件类型（例如：`typeAll`、`typeViewClicked`）。 |
| `accessibilityFeedbackType` | 服务提供的反馈类型（例如：`feedbackGeneric` 用于自动化）。 |
| `canRetrieveWindowContent` | **设置为 `true`** 才能访问窗口内容（读取屏幕信息）。 |
| `packageNames` | 服务仅关注的应用包名列表。如果不设置，则监听所有应用。 |
| `canRequestTouchExploration` | 是否请求触摸探索模式。 |

我的配置如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:canPerformGestures="true"
    android:canRetrieveWindowContent="true"
    android:description="@string/description_in_xml"
    android:notificationTimeout="100" />
```

### 实现逻辑
既然要跳广告，就需要在页面内容变化时，扫描页面上的广告元素。如果发现广告元素，就模拟点击 **跳过** 按钮。

在 `AccessibilityService` 整个作用域中，我们都可以获取到当前活动窗口的根节点 `rootInActiveWindow` ，它是一个 `AccessibilityNodeInfo` 对象，代表了当前活动窗口的视图层次结构。这个结构里包含了所有可见的 UI 元素，我们可以通过遍历这个结构，来查找广告元素和跳过按钮。

构建一个扩展方法 `scanAndClickByText` ，用于扫描页面上的文字元素，并点击指定的元素。在 `onAccessibilityEvent()` 回调方法中，我们可以调用这个扫描方法，来查找并点击广告跳过按钮。

```kotlin
/**
 * 扫描文字，点击扫描到的第index个，默认第一个
 */
fun AccessibilityService.scanAndClickByText(scanText: String, index: Int = 0) = try {
    infoLog("scanText:$scanText")
    rootInActiveWindow?.findAccessibilityNodeInfosByText(scanText)
        ?.get(index)?.apply {
            infoLog(this.text.toString())
            val rect = Rect()
            this.getBoundsInScreen(rect)
            val x = rect.centerX()
            val y = rect.centerY()
            infoLog("x:$x, y: $y")
            performClickByCoordinate(x.toFloat(), y.toFloat())
        }
} catch (e: Exception) {
    e.message?.let { errorLog(it) }
}
```

有时候软件提供商会规避自己的文字被辅助服务扫描到，这时候也可以根据控件id来识别元素。先找到广告元素的id，然后根据id来点击跳过按钮。

```kotlin
/**
 * 扫描控件id，点击扫描到的第index个，默认第一个
 */
fun AccessibilityService.scanAndClickById(viewId: String, index: Int = 0) = try {
    infoLog("scanViewId:$viewId")
    rootInActiveWindow?.findAccessibilityNodeInfosByViewId(viewId)
        ?.get(index)?.apply {
            infoLog(this.text.toString())
            val rect = Rect()
            this.getBoundsInScreen(rect)
            val x = rect.centerX()
            val y = rect.centerY()
            infoLog("x:$x, y: $y")
            performClickByCoordinate(x.toFloat(), y.toFloat())
        }
} catch (e: Exception) {
    e.message?.let { errorLog(it) }
}
```

`performClickByCoordinate` 这个方法又是怎么实现点击的呢？这里要用到 `GestureDescription` 类。

`GestureDescription` 是 Android 7.0 (API 24) 及以上版本中，AccessibilityService 用来创建和执行复杂触摸手势的核心类。它取代了之前通过 `sendMotionEvent` 模拟点击的旧方法，提供了更强大、更灵活的途径来自动化手势操作。

不管是滑动还是点击，都可以用 `GestureDescription` 来实现。

```kotlin
/**
 * 创建滑动手势
 * 使用：第二第三参数均可为空
 * dispatchGesture(@NonNull GestureDescription gesture,
 *             @Nullable GestureResultCallback callback,
 *             @Nullable Handler handler)
 *
 */
fun AccessibilityService.startSwipeGesture(
    startX: Float,
    startY: Float,
    endX: Float,
    endY: Float,
    duration: Long = 500L,
    callback: GestureResultCallback? = null,
    handler: Handler? = null
) {
    val path = Path()
    path.moveTo(startX, startY)
    path.lineTo(endX, endY)
    val builder = GestureDescription.Builder()
    // 立即开始
    val startTime = 0L
    // 滑动持续时间（单位：毫秒）
    val duration = duration
    val stroke = GestureDescription.StrokeDescription(path, startTime, duration)
    builder.addStroke(stroke)
    // 分发滑动手势
    dispatchGesture(builder.build(), callback, handler)
}
```

要模拟点击的话，将 `duration` 持续时间这个参数设置比较短即可。
### 用户授权
在项目代码中按照普通服务的启动方式，不管是 `startService()` 还是 `bindService()` 都是无法启动辅助服务的。

由于 `AccessibilityService` 拥有极高的权限，他可以做的事情和用户手动操作的权限是相同的。用户必须在 **设置 - 辅助功能** 中找到定义的服务祝福，并手动确认启用你的服务。在代码中，可以引导用户跳转到相应的设置页面进行授权。

![](/assets/img/blog/blogs_access_service_skip_ads_demo.png)

## 其他典型应用场景
Android辅助服务主要用于以下场景：
* **辅助工具：** 屏幕阅读器、盲人导航应用、放大镜等。
* **自动化和效率工具：**
    * 自动跳过开屏广告。
    * 模拟用户操作完成重复的签到、点赞等任务。
    * 在特定条件下自动执行点击操作。
* **家长控制与安全监控：** 监控孩子使用的应用、限制特定操作。
* **跨应用功能：** 实现全局快捷操作，例如截屏或启动特定功能。

**开发注意事项（重要！）**

由于其高权限特性， `Google` 对 `AccessibilityService` 的使用有严格的政策要求：

1.  **目的透明：** 你的应用必须有一个清晰且可访问的核心功能，**直接需要** `AccessibilityService` 的权限才能工作。例如，如果你的应用是一个自动化工具，这是合理的。
2.  **明确告知：** 必须在应用内明显的位置（如 $\text{Google Play}$ 描述、首次使用提示）**明确告知**用户你的应用使用此服务的原因，以及它将访问哪些数据。
3.  **不得滥用：** 严禁用于窃取用户隐私信息（如密码、银行卡信息）或在用户不知情的情况下进行欺诈性点击。

总结来说，**AccessibilityService** 是 Android 开发者实现跨应用自动化和高级交互功能的强大工具。

**使用时，务必遵守法律法规，确保用户的知情权和数据安全。**