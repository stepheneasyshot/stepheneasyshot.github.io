---
layout: post
description: > 
  本文介绍了在AOSP项目自定义时，没有UI设计师，开发设计自己的开机动画的一种方法流程
image: 
  path: /assets/img/blog/blogs_bootanimation.png
  srcset: 
    1920w: /assets/img/blog/blogs_bootanimation.png
    960w:  /assets/img/blog/blogs_bootanimation.png
    480w:  /assets/img/blog/blogs_bootanimation.png
accent_image: /assets/img/blog/blogs_bootanimation.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AOSP】没有UI设计师如何自制Android开机动画
## 背景
学习C++之余，想把原生AOSP的开机动画给更新替换下，换换口味。

我们都知道，AOSP的默认开机动画是一个“ANDROID”的字样，配合一个渐变的底色动画。定制一个自己的开机动画，对于手机厂商来说，有利于宣传品牌，彰显企业文化。

像国内广为人知的定制系统，比如MIUI，ColorOS，FlymeOS等，都是没有直接使用默认动画的，定制了一套他们自己厂商的开机动画。而考虑到大厂都是人力充足，设计师，动效师，应有尽有。

那像我这自己在下面玩玩源码的，没有设计师帮忙，该怎么搞一套看得过去的定制化的开机动画呢？

## 开始制作
### 从压缩包制作倒推流程
首先经过调研了解到，我们如果想要自己定制Android的开机动画，需要准备一个名为bootanimation.zip的压缩包，去替换系统默认的动画。

那压缩包里放什么文件呢？

zip包里面的文件格式一般比较固定：
* disc.txt，用来描述帧动画的播放策略和显示大小。
* 若干个文件夹，里面是按照顺序命名的帧动画文件。

像我的就是下面这个结构：

![bootanimation](/assets/img/blog/blogs_bootanimation_files.png){:width="220" height="160" }

#### disc.txt
这个文件里的内容格式也比较简单：

```
364 830 15
p 1 0 part0
```

第一排364 830 15，依次表示：开机动画显示区域heiht高度364，width宽度830，帧数15

后面可以设置多行不同表现形式的动画，这里我设置一个简单动画，只有一行```p 1 0 part0``` ，首个字母表示动画播放的时段，有三种方案可选：

```
p -- this part will play unless interrupted by the end of the boot
c -- this part will play to completion, no matter what
f -- same as p but in addition the specified number of frames is being faded out while continue playing. Only the first interrupted f part is faded out, other subsequent f parts are skipped
```

```p``` 就表示直接全程播放，直到开机完成。第二位的 1 表示播放一次，如果是 0 就是循环播放。

第三位的0表示每两帧图片之间时间间隔为 ```0``` ms，

最后的part0表示需要展示的这些帧动画在这个文件夹中。

**注意：最后需要留出一个空行，编辑时需要注意。**

文件写法明确了。难点在于，没有UI设计师帮忙，如何搞到这些帧动画呢？

### 帧动画的制作
直接先展示制作路线：

Android应用里手动写一个简单的渐亮动画——>录屏——>MP4转PNG

我准备在Android应用里手写一个动画，在想办法转成png。

设计上力求简洁，我使用“Stephen OS”作为文案，也是做一个渐亮的表现形式。

方案定下来了，接下来随便新建一个demo项目。我找到一个免费字体Cooper Black，到应用xml里添加TextView控件，设置字体fontFamily:

```xml
 <TextView
        android:id="@+id/tv_animationtext"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:fontFamily="@font/cooper_black"
        android:rotation="90"
        android:alpha="0"
        android:text="Stephen OS"
        android:textColor="@color/white"
        android:textSize="88sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```
因为我是Pixel 5手机上刷的AOSP车机系统，所以开机时的屏幕方向还是vertical方向的，而我需要让其横向展示，所以在这个控件放置时直接旋转了90度。而且初始的透明度为0.

然后在Activity里准备写逻辑，顺手在工具类里写一个顶层扩展方法，给Activity设置强制全屏：

```kotlin
fun AppCompatActivity.setFullScreenMode() {
    val layoutParams = window.attributes
    layoutParams.layoutInDisplayCutoutMode =
        WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES
    window.attributes = layoutParams
    window.setFlags(
        WindowManager.LayoutParams.FLAG_FULLSCREEN,
        WindowManager.LayoutParams.FLAG_FULLSCREEN
    )
    val uiOptions = (View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
            or View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY or View.SYSTEM_UI_FLAG_FULLSCREEN)
    window.decorView.systemUiVisibility = uiOptions
}
```

动画编码为求简洁迅速，没有用传统的ValueAnimator，而是直接协程里使用repeat加delay，这种写法相当简单粗暴。

```kotlin
val logoText = rootView.findViewById<TextView>(R.id.tv_animationtext)
MainScope().launch {
     delay(2000L)
     repeat(255) {
          delay(7)
          infoLog(it.toString())
          logoText.alpha = (it / 255.0).toFloat()
     }
}
```
然后开启录屏软件，减去首尾多余部分，得到一个纯净的MP4，就是预设的开机动画了。

最后用到这样一个网站，可以将MP4转为png：Video to PNG image sequence converter

得到png后我们下载到本地，批量重命名成顺序的格式。将其放置到part0的文件夹中。准备和disc.txt一起打包。这里有一个坑，不可以直接用7zip打包成zip，最好使用winrar，压缩方式要选择存储：

### 集成到源码
压缩完成后，我们将bootanimation.zip放置到源码的某一个文件夹中。然后在你设备product的mk文件中，随意一个位置，加一句文件复制的指令：

```
PRODUCT_COPY_FILES += \
<path-to-your-bootanimation.zip>:system/media/bootanimation.zip
```

含义为在打包时，将这个zip包复制到ROM的system/media文件夹下。

设备开机后android系统会检索这个文件夹下有没有名为```bootanimation.zip```的文件，有的话就优先播这个动画替换默认的开机动画。