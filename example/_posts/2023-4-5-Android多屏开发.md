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


获取可用的屏幕列表：

```kotlin
fun getConnectedScreenIds(context: Context): List<Int> {
    val displayManager = context.getSystemService(Context.DISPLAY_SERVICE) as DisplayManager
    val displays = displayManager.displays
    val screenIds = mutableListOf<Int>()
    for (display in displays) {
        screenIds.add(display.displayId)
    }
    infoLog("screenIds = $screenIds")
    return screenIds
}
```

## Presentation

```kotlin
class MyPresentation(context: Context, display: Display) : Presentation(context, display) {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.presentation)
    }
}
```

```kotlin

val displayManager =
    this@MainActivity.getSystemService(DISPLAY_SERVICE) as DisplayManager
val display = displayManager.getDisplay(5)
val presentation = MyPresentation(this@MainActivity, display)
presentation.setCancelable(false)
presentation.show()
```

## Activity

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
