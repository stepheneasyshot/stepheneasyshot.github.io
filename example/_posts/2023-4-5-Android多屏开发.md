---
layout: post
description: > 
  本文介绍了Android多屏开发的两种方式
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
# Android多屏开发
车载Android开发和手机端的一个重大区别之一，就是车载Android设备通常拥有多个屏幕，比如一个主屏幕和一个副屏幕。在手机端通常只需要一个屏幕，但是在车载Android开发中，我们很多时候需要同时处理多个屏幕。

例如现在相当一部分的新能源车拥有主副驾两块大屏幕，主驾显示的界面为导航和车辆状态等，副驾屏幕用来显示一些娱乐app的流媒体等。甚至很多车企，还会有吸顶屏幕给后排乘客使用。

本文将介绍Android多屏开发主流的两种实现方式：Presentation和Activity。以下两种方式默认层级场景下，无系统权限的app也可以使用。

## 多屏设备的获取

首先，我们需要获取到多屏设备的信息，包括屏幕的数量、屏幕的尺寸、屏幕的方向等。

在Android中，我们可以通过DisplayManager来获取到多屏设备的信息。

```kotlin
fun getConnectedScreenIds(context: Context): List<Int> {
    val displayManager = context.getSystemService(Context.DISPLAY_SERVICE) as DisplayManager
    val displays = displayManager.displays
    val screenIds = mutableListOf<Int>()
    for (display in displays) {
        screenIds.add(display.displayId)
        display.name
        infoLog("displayId = ${display.displayId}, name = ${display.name}")
    }
    return screenIds
}
```

其中，displayId是屏幕的唯一标识符，displayName是屏幕的名称。

其中display.getSize方法已经废弃，改为采用windowmanager中获取密度的方法来获取尺寸。

每次开关机之后，，displayId有可能不是固定的，主要看系统厂商是否对多屏的id进行了重置。

## Presentation

Presentation是Android提供的一种用于显示UI的类，它可以在一个单独的窗口中显示UI。

Presentation继承自Dialog类，因此它也可以设置窗口的属性，比如窗口的大小、窗口的位置等。

设置Presentation时，需要传入一个Context，一个Display对象，以确定Presentation的显示位置。然后在onCreate方法中设置UI即可。

```kotlin
class MyPresentation(context: Context, display: Display) : Presentation(context, display) {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.presentation)
    }
}
```

创建实例，调用show方法。同时，我们也可以精细地定义其显示层级，显示大小，以及是否可以点击外部取消。

```kotlin

val displayManager =
    this@MainActivity.getSystemService(DISPLAY_SERVICE) as DisplayManager
val display = displayManager.getDisplay(5)
val presentation = MyPresentation(this@MainActivity, display)
presentation.setCancelable(false)
presentation.show()
```

## Activity
第二种方案可以直接使用Activity来实现多屏开发。

使用到了ActivityOptions类，官方定义为，其是一个用于构建一个选项Bundle的辅助类，该Bundle可与Context.startActivity(Intent, Bundle)及相关方法一起使用。

我们可以利用它来定义转场动画等关于显示的很多参数。这里用到了其指定displayid来显示的功能。

```
fun startPassengerActivity() {
    val intent = Intent().apply {
        setComponent(ComponentName("com.stephen.redfindemo", "com.stephen.redfindemo.PassengerActivity"))
        setFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_MULTIPLE_TASK)
    }
    val displayId = 5;
    val options = ActivityOptions.makeBasic();
    options.setLaunchDisplayId(displayId);
    try {
        this.startActivity(intent, options.toBundle());
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```
