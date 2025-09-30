---
layout: post
description: > 
  本文介绍了Android平台上的一些常用的硬件设备相关的api总结。
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
# 【Android进阶】Android平台硬件状态获取总结
纲要转载自掘金老哥的一篇文章，实机测试及扩展而来。
[【掘金原文】](https://juejin.cn/post/7501159920600760320#comment)

## 检测是手机还是平板
Android中没有提供特定的方法来判断设备是手机还是平板，只能通过别的方式来间接判断，比如通过判断屏幕尺寸。

```kotlin
infoText.text = checkIsTablet()

private fun checkIsTablet(): String {
    val metrics = resources.displayMetrics
    val widthInches = metrics.widthPixels / metrics.xdpi
    val heightInches = metrics.heightPixels / metrics.ydpi
    val diagonalInches = sqrt(widthInches.pow(2.0f) + heightInches.pow(2.0f))
    return if (diagonalInches >= 7.0) {
        "手机还是平板：平板"
    } else {
        "手机还是平板：手机"
    }
}
```

## 判断是否为折叠屏
其实在折叠屏没出现的时候，判断手机或者是平板使用上述方法还是够用的，但是在折叠屏面前就显得信心不足了，折叠屏一展开，那就是一个长着平板脸的手机，为了识别折叠屏，Android10出来了一个新的感应器类型TYPE_HINGE_ANGLE，可以通过是否存在这种感应器来识别折叠屏。

```kotlin
private fun checkIsFoldScreen(): String {
    val sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
    val hingeAngleSensor = sensorManager.getDefaultSensor(Sensor.TYPE_HINGE_ANGLE)
    return if (hingeAngleSensor == null) {
        "是否折叠屏：否"
    } else {
        "是否折叠屏: 是"
    }
}
```

如果说想要具体拿到折叠屏的状态，比如是全展开还是半展开，或者收起状态，就要使用Jetpack WindowManager这个库了，分别可以通过以下api拿到不同的状态。

![](/assets/img/blog/blogs_jetpack_windowmanager.png)

## 屏幕密度与密度比例
这两个值相信基本每个项目都会用到，屏幕密度一般用来判断屏幕适配，加载不同的图片资源，密度比例一般用来单位换算，这俩值都可以通过DisplayMetrics来获得

```kotlin
infoText.text = checkScreenDpiAndDensity()

private fun checkScreenDpiAndDensity(): String {
    val displayMetric = resources.displayMetrics
    val dpi = displayMetric.densityDpi
    val density = displayMetric.density
    return "屏幕密度：${dpi}  密度比例：${density}"
}
```

## 屏幕像素(宽高)
```kotlin
private fun checkScreenPixel(): String {
    val displayMetrics = DisplayMetrics()
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        display?.getRealMetrics(displayMetrics)
    }else{
        windowManager.defaultDisplay.getRealMetrics(displayMetrics);
    }
    val wPixel = displayMetrics.widthPixels
    val hPixel = displayMetrics.heightPixels
    return "像素(宽)：${wPixel} 像素(高)：${hPixel}"
}
```

## 物理尺寸
物理尺寸在安卓上单位是英寸，它表示一个屏幕对角线的长度，至于如何计算对角线，就要用到上学时候用到的勾股定理，x,y分别是屏幕的宽高，注意的是由于单位是英寸，所以也要把上面计算出来的像素转换成英寸，具体代码如下

```kotlin
private fun checkPhysicalSize(): String {
    val displayMetrics = DisplayMetrics()
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        display?.getRealMetrics(displayMetrics)
    }else{
        windowManager.defaultDisplay.getRealMetrics(displayMetrics);
    }
    val widthInches = displayMetrics.widthPixels / displayMetrics.xdpi
    val heightInches = displayMetrics.heightPixels / displayMetrics.ydpi
    val diagonalInches = sqrt(
        widthInches.pow(2.0f) + heightInches.pow(2.0f)
    )
    return "物理尺寸 $diagonalInches"
}
```

## 刷新率
刷新率一般就是指Android屏幕上每秒更新画面的频率，单位是赫兹，正常来讲，普通设备的刷新率都为60赫兹，获取刷新率的代码如下

```kotlin
private fun checkRefreshRate(): String {
    val mDisplay = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        display
    }else{
        windowManager.defaultDisplay
    }
    return "刷新率 ${mDisplay?.refreshRate}"
}
```

## 广色域
有的设备支持广色域，有的设备仅仅支持标准色域，广色域的意思是屏幕可以显示比标准色域(sRGB)更加丰富的颜色范围，判断一个设备是否支持广色域的方式如下

```kotlin
private fun checkColorGamut(): String {
    val config: Configuration = resources.configuration
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val isWideColorGamut: Boolean = config.isScreenWideColorGamut
        val support = if(isWideColorGamut) "支持" else "不支持"
        return "是否支持广色域模式:${support}"
    }
    return "不支持广色域"
}
```

## 获取内存(Runtime)
一般来讲应用想获取内存信息，用的最多的就是通过Runtime来获取，可以通过它获取应用最大可用内存，当前分配的内存，当前空闲内存，已使用内存

```kotlin
infoText.text = checkMemoryRuntime()

private fun checkMemoryRuntime(): String {
    val runtime = Runtime.getRuntime()
    val maxMemory = runtime.maxMemory() // 应用最大可用内存
    val totalMemory = runtime.totalMemory() // 当前分配的内存
    val freeMemory = runtime.freeMemory() // 当前空闲内存
    val usedMemory = totalMemory - freeMemory // 已使用内存
    return "最大可用内存:${maxMemory} 当前分配的内存:${totalMemory} 当前空闲内存:${freeMemory} 已使用内存${usedMemory}"
}
```

## 获取内存(MemoryInfo)
还有一种方式就是通过获取MemoryInfo来拿到内存信息，比如总内存，当前空闲内存以及判断内存是否过低.

```kotlin
infoText.text = checkMemoryMemoInfo()

private fun checkMemoryMemoInfo(): String {
    val activityManager = getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
    val memoryInfo = ActivityManager.MemoryInfo()
    activityManager.getMemoryInfo(memoryInfo)
    val totalMem = memoryInfo.totalMem
    val availMem = memoryInfo.availMem
    val lowMemory = memoryInfo.lowMemory
    return "总内存:${totalMem} 当前空闲内存:${availMem} 内存是否过低${lowMemory}"
}
```

## 磁盘空间(外部存储与内部存储)
可以通过StatFs来获取外部存储以及内存存储容量

```kotlin
//外部存储
infoText.text = checkExternalStorageInfo()

private fun checkExternalStorageInfo(): String {
    if (Environment.getExternalStorageState() == Environment.MEDIA_MOUNTED) {
        val path = Environment.getExternalStorageDirectory() // 外部存储根目录
        val stat = StatFs(path.path)
        val blockSize = stat.blockSizeLong
        val totalBlocks = stat.blockCountLong
        val availableBlocks = stat.availableBlocksLong
        val totalSize = blockSize * totalBlocks
        val availableSize = blockSize * availableBlocks
        val usedSize = totalSize - availableSize
        return "外部存储总容量:${totalSize} 外部存储可用容量:${availableSize} 外部存储已用容量:${usedSize}"
    }
    return ""
}
```

```kotlin
//内部存储
infoText.text = checkInternalStorageInfo()

private fun checkInternalStorageInfo(): String {
    val path = Environment.getDataDirectory() // 内部存储根目录
    val stat = StatFs(path.path)
    val blockSize = stat.blockSizeLong // 每个block的大小
    val totalBlocks = stat.blockCountLong // 总block数
    val availableBlocks = stat.availableBlocksLong // 可用block数

    val totalSize = blockSize * totalBlocks // 总容量
    val availableSize = blockSize * availableBlocks // 可用容量
    val usedSize = totalSize - availableSize // 已用容量
    return "内部存储总容量:${totalSize} 内部存储可用容量:${availableSize} 内部存储已用容量:${usedSize}"
}
```

## CPU内核数量
获取CPU的内核数量很简单，Runtime类中有现成的方法

```kotlin
infoText.text = checkCPUcoreNumber()

private fun checkCPUcoreNumber() = 
     "cpu核心数 ： ${Runtime.getRuntime().availableProcessors()}"
```

## CPU架构

```kotlin
infoText.text = checkCPUArchitecture()

private fun checkCPUArchitecture() =
     "cpu架构：${Build.SUPPORTED_ABIS[0]}"
```

## CPU最大频率

```kotlin
fun getCpuFreq(): String? {
    try {
        val reader =
            BufferedReader(FileReader("/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq"))
        val freq = reader.readLine()
        reader.close()
        // 将频率从 kHz 转换为 MHz
        return "${(freq.toLong() / 1000)} MHz"
    } catch (e: IOException) {
        e.printStackTrace()
    }
    return null
}
```

## CPU硬件信息
```kotlin
infoText.text = checkCPUHardware()
private fun checkCPUHardware() = 
        "硬件信息：${Build.HARDWARE}"
```

## 检测设备是否root
同样的没有任何api可以直接去判断设备是否有root权限，我们只能从以下几个方式去判断:
**判断检查是否存在相关root文件**

```kotlin
var fileRooted = false
val paths = arrayOf(
    "/system/app/Superuser.apk",
    "/sbin/su",
    "/system/bin/su",
    "/system/xbin/su",
    "/data/local/xbin/su",
    "/data/local/bin/su",
    "/system/sd/xbin/su",
    "/system/bin/failsafe/su",
    "/data/local/su",
    "/su/bin/su"
)
for (path in paths) {
    if (File(path).exists()) {
        fileRooted = true
    }
}
```

**检查是否存在su命令**

```kotlin
var suCmdExest = false
var process: Process? = null
try {
    process = Runtime.getRuntime().exec(arrayOf("which", "su"))
    val reader = BufferedReader(InputStreamReader(process.inputStream))
    suCmdExest = reader.readLine() != null
} catch (e: Exception) {
    suCmdExest = false
} finally {
    process?.destroy()
}
```

**检查 `Build.TAGS` 里面是否存在 `test-keys`**

```kotlinvar testKeys = false
val buildTags = Build.TAGS
testKeys = buildTags != null && buildTags.contains("test-keys")
```

**执行su命令**

```kotlin
var suCmdExecute = false
var suprocess: Process? = null
try {
    suprocess = Runtime.getRuntime().exec("su")
    val out = suprocess.outputStream
    out.write("exit\n".toByteArray())
    out.flush()
    out.close()
    suCmdExecute = suprocess.waitFor() == 0
} catch (e: java.lang.Exception) {
    suCmdExecute = false
} finally {
    suprocess?.destroy()
}
```

**Magisk 文件是否存在**

```kotlin
var giskFile = false
val magiskPaths = arrayOf(
    "/sbin/.magisk",
    "/sbin/magisk",
    "/cache/.disable_magisk",
    "/cache/magisk.log",
    "/data/adb/magisk",
    "/data/adb/modules",
    "/data/magisk",
    "/data/magisk.img"
)

for (path in magiskPaths) {
    if (File(path).exists()) {
        giskFile = true
    }
}
```

所有条件综合判断：

```kotlin
val gotRoot = fileRooted || suCmdExest || testKeys || suCmdExecute || giskFile
```

## 网络情况
这个也是在应用当中经常会用到的一个属性，判断设备是连接的是wifi,还是连接的是2,3,4,5G网络，首先通过获取  `NetworkCapabilities` 来判断是否连接的是wifi还是移动网络。

```kotlin
private fun checkNetworkType(): String {
    var net = ""
    val cManager = getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    val capabilities: NetworkCapabilities? =
        cManager.getNetworkCapabilities(cManager.activeNetwork)
    capabilities?.let { cb ->
        if (capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)) {
            net = "WIFI"
        } else if (capabilities.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)) {
            net = getMobileNetworkType(cManager)
        }
    }
    return "网络情况：${net}"
}
```

当判断出非wifi网络的时候，再通过getMobileNetworkType函数来得出具体的网络类型

```kotlin
private fun getMobileNetworkType(cManager: ConnectivityManager): String {
    val networkInfo = cManager.getNetworkInfo(ConnectivityManager.TYPE_MOBILE)
    if (networkInfo != null) {
        val networkType = networkInfo.subtype
        return when (networkType) {
            TelephonyManager.NETWORK_TYPE_GPRS, TelephonyManager.NETWORK_TYPE_EDGE -> "2G"
            TelephonyManager.NETWORK_TYPE_UMTS, TelephonyManager.NETWORK_TYPE_HSPA -> "3G"
            TelephonyManager.NETWORK_TYPE_LTE -> "4G"
            TelephonyManager.NETWORK_TYPE_NR -> "5G"
            else -> "UNKNOWN"
        }
    }
    return "UNKNOWN"
}
```

