---
layout: post
description: > 
  本文介绍了Activity组件的进阶知识，包括Activity的生命周期、Activity的启动模式、Activity的通信方式等。
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
# Activity组件进阶
作为一名 Android 开发者，**Activity** 绝对是你最常用、也是最重要的组件。它是用户界面的单一入口点，承载着应用与用户交互的各种操作。你可以把它想象成应用中的一个“屏幕”或“页面”。

-----

### Activity 的核心作用

1.  **用户界面载体：** Activity 提供一个窗口，你可以在其中绘制 UI 界面（如按钮、文本框、图片等），并与用户进行交互。
2.  **生命周期管理：** Activity 拥有一套定义好的生命周期回调方法，允许你在 Activity 的不同状态（如创建、启动、暂停、停止、销毁等）下执行相应的逻辑。
3.  **多任务处理：** 每个 Activity 实例都与一个任务（task）相关联。当用户启动应用时，系统会为它创建一个任务，并在这个任务中管理 Activity 的堆栈。
4.  **通信桥梁：** Activity 可以启动其他 Activity（包括自己应用内或第三方应用的 Activity），并通过 `Intent` 传递数据。

-----

### Activity 的生命周期

理解 Activity 的生命周期是 Android 开发的基石。当你用户在应用中导航、接电话、切换应用等操作时，Activity 的状态会发生变化，系统会调用相应的回调方法。

以下是 Activity 生命周期中的几个核心方法：

  * **`onCreate()`**:
      * **何时调用：** Activity 首次创建时调用。
      * **作用：** 进行 Activity 的初始化工作，如设置布局（`setContentView()`）、初始化视图组件、绑定数据、恢复 `savedInstanceState` 中的数据等。这是你放置大部分一次性设置代码的地方。
  * **`onStart()`**:
      * **何时调用：** Activity 可见时调用，无论是因为首次创建还是从后台回到前台。
      * **作用：** Activity 即将对用户可见，但尚未获得用户焦点。
  * **`onResume()`**:
      * **何时调用：** Activity 获得用户焦点并可与用户交互时调用。
      * **作用：** 在这里启动动画、访问设备相机或传感器等独占性资源。任何需要 Activity 处于前台才能进行的轻量级操作都应放在这里。
  * **`onPause()`**:
      * **何时调用：** Activity 失去焦点时调用，例如用户点击返回键，或启动了另一个 Activity 但当前 Activity 仍然部分可见（如弹出一个对话框），或屏幕关闭。
      * **作用：** 在这里暂停动画、释放独占性资源（如相机预览），并保存任何需要持久化的小量数据（但不要在这里执行耗时操作，因为下一个 Activity 必须等待当前 Activity 的 `onPause()` 执行完毕才能 `onResume()`）。
  * **`onStop()`**:
      * **何时调用：** Activity 不再可见时调用，例如用户切换到另一个应用、按下 Home 键、或启动了一个完全覆盖当前 Activity 的新 Activity。
      * **作用：** 释放那些在 Activity 不可见时不再需要的资源。重量级 CPU 操作，例如向数据库写入数据，应该在这里完成。
  * **`onDestroy()`**:
      * **何时调用：** Activity 被系统销毁时调用。这可能是以下原因之一：
          * 用户通过按下返回键完全退出 Activity。
          * 系统为了回收资源而销毁 Activity（例如，当内存不足时）。
          * 配置变更（如屏幕旋转、主题切换）导致 Activity 重新创建。
      * **作用：** 释放所有在 `onCreate()` 中创建的资源，如解绑广播接收器、关闭数据库游标、停止后台线程等。

-----

### Activity 状态和数据保存

当 Activity 被销毁后又重建时（例如屏幕旋转），你可能需要恢复之前的用户界面状态或数据。

  * **`onSaveInstanceState(Bundle outState)`**:
      * **何时调用：** Activity 即将被销毁，但未来可能会被重新创建时调用（例如屏幕旋转、内存不足导致系统回收）。
      * **作用：** 你可以将少量瞬态数据（如 UI 状态、滚动位置等）保存到 `Bundle` 中。这个 `Bundle` 会在 Activity 重新创建时通过 `onCreate()` 方法传递回来。
  * **`onRestoreInstanceState(Bundle savedInstanceState)`**:
      * **何时调用：** 在 `onStart()` 之后，并且仅当 Activity 之前因系统原因被销毁并重新创建时调用。
      * **作用：** 在这里恢复 `onSaveInstanceState()` 中保存的数据。通常，你也可以在 `onCreate()` 中通过 `savedInstanceState` 参数来恢复这些数据。

**注意：** 对于大量数据或需要长期保存的数据，不应依赖 `onSaveInstanceState()`。而应该使用 Room 数据库、SharedPreferences 或 ViewModel 来持久化数据。

-----

### 启动 Activity 和数据传递

你可以使用 `Intent` 来启动其他 Activity。

```kotlin
// 1. 启动一个显式 Activity (明确指定要启动的 Activity 类)
val intent = Intent(this, SecondActivity::class.java)
startActivity(intent)

// 2. 启动一个隐式 Activity (系统根据 IntentFilter 找到合适的 Activity)
val webIntent = Intent(Intent.ACTION_VIEW, Uri.parse("http://www.google.com"))
startActivity(webIntent)

// 3. 启动 Activity 并传递数据
val dataIntent = Intent(this, DetailActivity::class.java).apply {
    putExtra("item_id", 123)
    putExtra("item_name", "Awesome Product")
}
startActivity(dataIntent)

// 在 DetailActivity 中获取数据
// override fun onCreate(savedInstanceState: Bundle?) {
//     super.onCreate(savedInstanceState)
//     val itemId = intent.getIntExtra("item_id", -1)
//     val itemName = intent.getStringExtra("item_name")
// }

// 4. 启动 Activity 并获取返回结果 (旧方法，现在推荐 Activity Result API)
// override fun onCreate(...) {
//     val startForResult = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
//         if (result.resultCode == Activity.RESULT_OK) {
//             val data: Intent? = result.data
//             val message = data?.getStringExtra("return_message")
//             // 处理返回的数据
//         }
//     }
//
//     // 在某个点击事件中启动
//     myButton.setOnClickListener {
//         val intent = Intent(this, ResultActivity::class.java)
//         startForResult.launch(intent)
//     }
// }
```

-----

### Activity 任务栈 (Task Stack)

Android 系统通过\*\*任务（Task）\*\*来管理 Activity 的组织结构。一个任务是用户执行某项工作时与之交互的 Activity 的集合。这些 Activity 被组织在一个“后退栈”（Back Stack）中，以堆栈（LIFO，后进先出）的形式排列。

  * 当用户启动一个新 Activity 时，它会被推送到当前任务栈的顶部，并成为焦点。
  * 当用户按下返回键时，栈顶的 Activity 会被弹出并销毁，前一个 Activity 恢复到顶部。
  * 当栈中最后一个 Activity 被弹出时，任务就不再存在。

你可以在 `AndroidManifest.xml` 中使用 `android:launchMode` 来调整 Activity 的启动模式，从而影响其在任务栈中的行为：

  * **`standard` (默认)：** 每次启动都会创建新的实例。
  * **`singleTop`：** 如果目标 Activity 已经在栈顶，则不会创建新实例，而是调用其 `onNewIntent()` 方法。
  * **`singleTask`：** 确保一个任务中只有一个该 Activity 的实例。如果实例已存在于任何位置，则将其移动到栈顶并清理其上方的所有 Activity。
  * **`singleInstance`：** 类似于 `singleTask`，但它会创建一个全新的任务来包含这个 Activity，并且这个任务中只能有这一个 Activity。

-----

### Activity 和 Fragment 的关系

作为 Android 开发者，你经常会看到 Activity 和 Fragment 一起工作。理解它们的区别和联系至关重要：

  * **Activity 是骨架：** Activity 提供应用窗口和基本框架，管理整个屏幕的生命周期。
  * **Fragment 是模块：** Fragment 是 Activity 的一部分，有自己的生命周期和布局，但必须依附于 Activity。它用于构建模块化、可复用的 UI 片段。
  * **分工合作：** Activity 负责协调不同 Fragment 之间的交互、处理系统事件，而 Fragment 负责管理其内部的 UI 逻辑和数据。

-----

### Manifest 文件中的 Activity 配置

每个 Activity 都必须在应用的 `AndroidManifest.xml` 文件中声明。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.yourapp">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.YourApp">

        <activity
            android:name=".MainActivity"  android:exported="true"      android:launchMode="singleTask" android:screenOrientation="portrait" android:configChanges="orientation|screenSize|keyboardHidden" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" /> <category android:name="android.intent.category.LAUNCHER" /> </intent-filter>
        </activity>

        <activity android:name=".SecondActivity" />
        </application>
</manifest>
```

-----

希望这个详细的介绍能帮助你更全面地理解 Android Activity！如果你在开发过程中遇到关于 Activity 的具体问题，或者想进一步了解某个方面，都可以随时提问。