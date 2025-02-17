---
layout: post
description: > 
  使用Compose Multiplatform开发了一款跨平台的电脑端Android设备调试工具。本文简单介绍了开发背景，功能点，特殊问题解决等信息。
image: 
  path: /assets/img/blog/blogs_compose.png
  srcset: 
    1920w: /assets/img/blog/blogs_compose.png
    960w:  /assets/img/blog/blogs_compose.png
    480w:  /assets/img/blog/blogs_compose.png
accent_image: /assets/img/blog/blogs_compose.png
excerpt_separator: <!--more-->
sitemap: false
---

# 使用CMP开发跨平台的Android调试工具
## 背景
最近对CMP跨平台很感兴趣，为了练手，在移动端做了一个Android和IOS共享UI和逻辑代码的天气软件，简单适配了一下双端的深浅主题切换，网络状态监测，刷新调用振动器接口。

做了两年多车机Android开发，偶尔玩下手机端跨平台也蛮有意思。

然后又了解到CMP不仅仅是移动端的，还可以做web和desktop端。

在我们日常的开发过程中，对于车机设备的adb调试操作很多，一大半全是固定的流程。使用bat脚本的话又不那么灵活，体验也不好。所以我很早就想要做一个带界面的Android设备调试工具。在移动端上写纯原生的Compose界面比较熟悉了，想着这个估计也差不多的，就开启了为期一个多月的Compose for Desktop开发。开发体验可以算中上，很多的问题在stackoverflow和官网上都能找到方案。软件命名为DebugManager。
## 架构设计
我没有开发Desktop端的经验，不知道最优的架构设计是什么样的。使用CMP的话Google推崇的MVI模式依然可以通用，所以最初制定的技术路线就是使用响应式的架构。

由于功能单一，几乎所有操作都是执行一些命令行，获取反馈结果，所以没有抽象的很厉害，数据层直接使用单例类，使用adb工具获取数据透传到StateHolder。StateHolder为界面的状态State管理层，在Composable方法初入时，触发StateHolder的数据获取逻辑，数据拿取到之后，更新State状态，通过界面收集监听的stateflow通知composable方法刷新UI。

![Google_mvi](/assets/img/blog/blogs_mvi.png){:width="400" height="300" loading="lazy"}

即用户事件从上到下，数据状态从下到上，确保唯一可信数据流。
## gradle配置
这一步决定DebugManager项目面向的各个平台的配置，软件版本，安装包。

由于这个软件面向不同岗位，不同操作系统，目标是一套代码适配Windows，Linux，MacOS，达到多端通用。而且目前没有交叉编译，只能在各自的系统上打包，windows打exe，ubuntu上打deb，macos上打dmg，所以我现在给使用不同系统的同事发布软件时，都是三端各打一遍。

Windows端有配置是否显示在开始菜单，桌面快捷方式，uuid用于更新识别，自行选择安装目录。

```kotlin
// 开始菜单
menu = true
// 桌面快捷方式
shortcut = true
// 可自行选择安装目录
dirChooser = true
// 可单独为当前用户安装，不需要管理员权限
perUserInstall = true
// 设置图标
iconFile.set(project.file("launcher/icon.ico"))
// uuid用于更新识别
upgradeUuid = "xxxx-xxxxxxx-xxxxx"
```

更详细的Gradle属性配置参考可以看官方github仓库的教程文档：
[JetBrains官方配置文档](https://github.com/JetBrains/compose-multiplatform/blob/master/tutorials/README.md) 

### 图标配置
关于三个平台应用图标的设置，我们需要手动制作三端的图标文件。

* Linux使用的是png格式。可以作为源文件，来制作Windows和MacOS的图标。

* Windows端的图标为ico格式。可以通过这个在线网站来生成：

```
https://www.butterpig.top/icopro/
```

* MacOS端的图标为icns格式。

注意 Mac 端的图标需要使用苹果电脑才能生成。

首先我们切到png格式的图片所在的目录，执行下面三组命令即可生成Mac端的图标文件了：

> 创建输出文件夹
mkdir MyIcon.iconset

> 生成图标
sips -z 16 16     original.png --out MyIcon.iconset/icon_16x16.png
sips -z 32 32     original.png --out MyIcon.iconset/icon_16x16@2x.png
sips -z 32 32     original.png --out MyIcon.iconset/icon_32x32.png
sips -z 64 64     original.png --out MyIcon.iconset/icon_32x32@2x.png
sips -z 128 128   original.png --out MyIcon.iconset/icon_128x128.png
sips -z 256 256   original.png --out MyIcon.iconset/icon_128x128@2x.png
sips -z 256 256   original.png --out MyIcon.iconset/icon_256x256.png
sips -z 512 512   original.png --out MyIcon.iconset/icon_256x256@2x.png
sips -z 512 512   original.png --out MyIcon.iconset/icon_512x512.png
sips -z 1024 1024 original.png --out MyIcon.iconset/icon_512x512@2x.png

> 合成不同尺寸的图标
iconutil -c icns MyIcon.iconset


之后就可以看到一个后缀为icns的文件了，将其复制到项目中，gradle脚本里配置为应用图标。

#### 图标配置过程中的bug
目前还发现一个奇怪的bug，就是有的png图标经过转换，配置到项目中，打包exe出来是正常大小，大概90M。

有的图片生成完毕之后，Windows平台打包后的 EXE 安装包大小直接暴涨到了2个G，甚至3个G，目前不确定什么原因导致的。还在排查和寻求官方的帮助。

**解决**

经过几轮尝试排查，问题应该出在那个windows平台的转换网站上:

```
https://www.butterpig.top/icopro/
```

通过 IDE 打开生成的ico文件，发现其实际的文件类型是JPEG，并不是显示的ico文件。

“假icon！！”

![blogs_cmp_wrong_ico_file](/assets/img/blog/blogs_cmp_wrong_ico_file.png){:width="500" height="400" loading="lazy"}

目前怀疑图标类型错误，导致安装包暴涨。会产生这个现象的原因，可能是CMP所使用的Windows打包器的一个bug或者说一个规则吧。

使用Python的Pillow库来转换，发现生成的图标文件显示的是我需要的ICO类型了。

转换脚本很简单，如下：

```python
from PIL import Image

def png_to_ico(png_path, ico_path):
    # 打开PNG图像
    image = Image.open(png_path)

    # 将图像转换为ICO格式
    image.save(ico_path, format='ICO', sizes=[(image.width, image.height)])

# 调用函数并传入PNG图像路径和ICO文件路径
png_to_ico('C:\\Users\\stephen\\Desktop\\logo.png', 'output.ico')
```

转换后的图标文件：

![blogs_cmp_right_ico_file](/assets/img/blog/blogs_cmp_right_ico_file.png){:width="500" height="400" loading="lazy"}

将这个 ico 文件配置到项目之后，打包的大小已经恢复正常的90余M。

## Multiplatform适配
开发Desktop跨平台碰到的的第一个问题，就是不同平台的路径连接符不一致：

在Windows上是反斜杠  `\`

在unix like的系统上是一个正斜杠  `/`

经过探索，JVM系的应用其实可以使用 `File.separator` 来获取这个连接符拼到路径字符串里。

而且关于平台类型的区分，Java也给我们提供了一个 `System.getProperty` 接口。

### 单例模式
在Windows平台上，多次通过入口来运行exe文件，会产生多个进程，对于本软件是没有必要的，甚至有可能导致bug。

所以需要像任务管理器那样，不管有多少次的打开动作，始终只有一个进程一个界面。

这里通过文件锁的方式来实现。

刚开启进程就创建一个文件，并将其锁定。在JVM关闭的时候，释放并删除这个文件。这样如果软件已经有一个进程在运行了，再次打开时尝试去获取这个文件的独占锁，如果获取不到，就说明已经有一个实例在运行，直接退出后打开的这个进程。

```kotlin
class SingleInstanceApp {

    private var lock: FileLock? = null
    private var channel: FileChannel? = null

    fun initCheckFileLock(lockFilePath: String) {
        LogUtils.printLog("initCheckFileLock")
        val file = File(lockFilePath)
        channel = RandomAccessFile(file, "rw").getChannel()
        lock = channel?.tryLock()
        if (lock == null) {
            LogUtils.printLog("Another instance is already running.", LogUtils.LogLevel.ERROR)
            exitProcess(1)
        }
        // 添加JVM关闭时的钩子，释放锁
        Runtime.getRuntime().addShutdownHook(Thread(Runnable {
            runCatching {
                lock?.let {
                    it.release()
                    channel?.close()
                    file.delete()
                }
            }.onFailure { e ->
                e.printStackTrace()
            }
        }))
    }
}
```

通过依赖注入到平台化的管理类中去，init方法中，当配置文件夹一创建完毕，就进行获取锁的操作。

```kotlin
class PlatformAdapter(private val singleInstanceApp: SingleInstanceApp) {

    init {
        println("PlatformAdapter init")
    }

    fun init() {
        createInitTempFile()
        singleInstanceApp.initCheckFileLock(lockFilePath)
    }
}
```

### 平台渠道管理
首先，定义一个枚举类来设定平台类型：

```kotlin
enum class PlatformType {
    UNKNOWN,
    WINDOWS,
    MAC,
    LINUX,
}
```
在应用初始化时，通过接口获取平台名称，解析出哪一个平台：

```kotlin
/**
 * 获取当前平台类型
 */
private fun getPlatformType(): PlatformType {
    val osName = System.getProperty("os.name").lowercase(Locale.getDefault())
    return when {
        osName.contains("win") -> PlatformType.WINDOWS
        osName.contains("mac") -> PlatformType.MAC
        osName.contains("nix") || osName.contains("nux") || osName.contains("aix") -> PlatformType.LINUX
        else -> PlatformType.UNKNOWN
    }
}
```

后面在涉及平台差分化的时候，可以使用此方法来获取，执行不同操作。

比如打开不同平台上的文件管理器：

```kotlin
/**
 * 打开一个文件夹
 */
fun openFolder(path: String) {
    when (getPlatformType()) {
        PlatformType.WINDOWS, PlatformType.UNKNOWN -> {
            executeTerminalCommand("explorer.exe $path")
        }

        PlatformType.MAC -> {
            executeTerminalCommand("open $path")
        }

        PlatformType.LINUX -> {
            executeTerminalCommand("xdg-open $path")
        }
    }
}
```

对于各个平台上执行终端命令，使用的两个方法是相同的，无需结果就直接exec()，需要执行结果就是用ProcessBuilder来执行，等待结果。

```kotlin
/**
 * 执行终端命令
 */
fun executeTerminalCommand(command: String) {
    runCatching {
        Runtime.getRuntime().exec(command)
    }.onFailure { e ->
        LogUtils.printLog("执行出错：${e.message}", LogUtils.LogLevel.ERROR)
    }
}

/**
 * 执行命令，获取输出
 */
suspend fun executeCommandWithResult(command: String) = withContext(Dispatchers.IO) {
    val processBuilder = ProcessBuilder(*command.split(" ").toTypedArray())
    val process = processBuilder.start()

    val reader = BufferedReader(InputStreamReader(process.inputStream))
    val output = StringBuilder()
    var line: String?
    while (reader.readLine().also { line = it } != null) {
        output.append(line).append("\n")
    }
    // 等待进程结束
    process.waitFor()
    // 关闭输入流
    reader.close()
    output.toString()
}
```

## 窗口框架
新项目的应用入口如下：

```kotlin
fun main() = application {

    Window(
        onCloseRequest = {

        },
        title = "DebugManager",
        undecorated = true,
        state = windowState,
        icon = painterResource("image/icon.png"),
    ) {
       ....
    }
}
```

我们主要的内容区就在Window这个 `Composable` 方法里。

通过 `windowState` ，我们可以设置窗口初始大小，窗口最大最小化。

`undecorated` 参数，这个可以配置软件界面是否选择系统默认的标题栏。由于我希望在三端上的设计语言都可以统一，都使用我自定义的标题栏，所以这里改为设置 **true** 。

有意思的一点是，在最开始将 `undecorated` 设为 **true** 后，我发现用上的Compose自定义的标题栏后，无法使用鼠标拖动窗口了，一度试了很多方案都不行。

这里最后还是咨询谷歌的 `Gemini` ，它给我展示了一个 `WindowDraggableArea` 组件，居然直接套用即可完美解决，里面的区域就是支持拖动移动的。把标题栏的 Composable 方法放在这个 `WindowDraggableArea` 可组合项里面，就可以鼠标拖动标题栏来移动窗口了。

`WindowDraggableArea` 源码方法签名如下：

```kotlin
@androidx.compose.runtime.Composable
@androidx.compose.runtime.ComposableInferredTarget
public fun androidx.compose.ui.window.WindowScope.WindowDraggableArea(
    modifier: androidx.compose.ui.Modifier = COMPILED_CODE,
    content: @androidx.compose.runtime.Composable () -> kotlin.Unit = COMPILED_CODE
): kotlin.Unit { /* compiled code */
}
```

关于各个页面之间的导航切换，我是使用的官方扩展的跨平台版本的 `navigation` 库。定义导航图，然后使用 `navController` 来切换页面。

```kotlin
val navController = rememberNavController()
NavHost(navController = navController, startDestination = "device_info") {
    composable("device_info") {
        DeviceInfoScreen(navController = navController)
    } 

    ...
}
```

每次启动应用，DebugManager 应用开屏页面，做的简单的延时跳转，timeout后自动进入主页面。

![splash](/assets/img/blog/blogs_debugmanager_splash_screen.png)

## 功能划分
下面简单介绍下各个页面的调试功能有哪些。

在公司，一般的开发流程里有产品设计，有交互设计，UI设计，给我传达需求，输出资源。

1. 产品的功能设计上，这个软件自己心血来潮要做。我结合日常工作中的调试痛点，还参考了 adb 的命令介绍，选取了一些热门的组合功能和单次功能，分类添加到了界面内。
2. 在界面UI设计风格上，我是直接参考了每天打开的 Android Studio 里的主题插件， **Atom One Dark** 的颜色风格。

### 设备信息展示
一进入界面，首页当然是所连接设备的基本信息展示。

![device_info](/assets/img/blog/blogs_cmp_deviceinfo.png){:width="600" height="300" loading="lazy"}

大致的实现思路如下，关于界面状态，先定义 `UiState` ：

```kotlin
data class DeviceState(
    val name: String? = null,
    val manufacturer: String? = null,
    val sdkVersion: String? = null,
    val systemVersion: String? = null,
    val buildType: String? = null,
    val innerName: String? = null,
    val resolution: String? = null,
    val density: String? = null,
    val cpuArch: String? = null,
    val serial:String? = null,
    val isConnected: Boolean = false
) {
    fun toUiState() =
        DeviceState(
            name = name,
            systemVersion = systemVersion,
            manufacturer = manufacturer,
            sdkVersion = sdkVersion,
            buildType = buildType,
            innerName = innerName,
            resolution = resolution,
            cpuArch = cpuArch,
            density = density,
            serial = serial,
            isConnected = isConnected
        )
}
```
定义好界面所需要展示的字段，再在StateHolder里维护一个StateFlow，同时对界面层暴露一个只读的字段，用于刷新界面数据。

```kotlin
// 单个设备信息
private val _deviceState = MutableStateFlow(DeviceState())
val deviceStateStateFlow = _deviceState.asStateFlow()
```
进来界面后，在协程中获取数据，界面拿到update后的数据之后自动更新信息：

```kotlin
   CoroutineScope(Dispatchers.IO).launch {
                prepareEnv()
                val deviceName = .....

                _deviceState.update {
                    it.copy(
                        name = deviceName,
                        manufacturer = manufacturer,
                        sdkVersion = sdkVersion,
                        systemVersion = systemVersion,
                        buildType = buildType,
                        density = displayDensity,
                        innerName = innerName,
                        resolution = displayResolution,
                        cpuArch = architecture,
                        serial = serialNum
                    )
                }
                _deviceState.value = _deviceState.value.toUiState()
                // 初始化获取文件列表
                getFileList()
            }
```
右侧的一堆按钮，是一些高频使用的功能。

简单的像reboot，root等，还有使用am打开隐藏app的界面，使用perfetto抓取trace，自动拉取到电脑端。

其中执行qnx命令为车机特有，现在市面上车机Android大多是运行在QNX系统上的子系统，DebugManager还可以直接桥接到QNX系统，执行更底层更精准的命令，比如执行reset重启整个IVI系统，而不只是reboot重启Android子系统。

录屏，截屏很实用，不用掏出手机到处找角度。我们提前设置好时长，通过自动执行多条指令，等操作完毕，可以直接将截屏录屏文件导出到电脑进行分享，也是我认为最好用的功能之一。

最下面还有一些基础的音量加减，模拟输入法输入等。

### 轮询查询机制
值得一提的是，我加入了循环获取连接设备数量和当前连接状态的机制，当电脑端的adb服务一初始化成功，就立即开启一个死循环的协程，每2s会查询一次当前设备的连接状态，设备数量。

```kotlin
 private fun recycleCheckConnection() {
        CoroutineScope(Dispatchers.IO).launch {
            while (true) {
                delay(2000L)
                runCatching {
                    // 通过系统命令，检索连接设备的数量是否变化
                    val deviceCount = ....
                    if (deviceCount != _deviceMapState.value.deviceMap.size) {
                        getDeviceMap()
                        MainScope().launch {
                            delay(800L)
                            getCurrentDeviceInfo()
                        }
                    }

                    // 检索当前设备连接状态
                    val result = ....
                    // 从断开到成功连接，主动刷新一次设备信息
                    if (!isConnected) {
                        getCurrentDeviceInfo()
                    }
                    isConnected = true
                    _deviceState.update {
                        it.copy(
                            isConnected = true,
                        )
                    }
                    _deviceState.value = _deviceState.value.toUiState()
                }.onFailure { error ->
                    LogUtils.printLog("${error.message}", LogUtils.LogLevel.ERROR)
                    isConnected = false
                    _deviceState.update {
                        it.copy(
                            isConnected = false,
                        )
                    }
                    _deviceState.value = _deviceState.value.toUiState()
                }
            }
        }
    }
```

1. 当增减设备时，刷新设备列表，左上角展开后可以选择不同的设备进行调试。
2. 当现在操作的设备断开连接时，会自动切换成其他设备，如果没有其他设备，就弹出警告弹窗，不允许继续操作页面了。

这两个都是轮询的。所以在重新连接设备后，会将当前状态通过state发送到界面，警告弹窗会自动消失。

### 软件安装管理
这个功能是耗时最长的板块之一，主要是Android系统里面每个包的信息如何展示，如何进一步对其进行替换，结合工作中积累的命令，在全网收集了很多指令，来实现软件包的管理功能。

APP列表加入了全部包扫描和三方包扫描，对于公司定制的包，也添加到了单独的筛选规则，可以自由选择查看全量信息和精简信息。

![app_manage](/assets/img/blog/blogs_cmp_appmanage_1.png){:width="600" height="300" loading="lazy"}

最上面是安装功能，是使用adb install进行的操作，适合第三方app进行验证时，或者改bug进行非正式环境的验证时使用。下拉框展开后，可以选择覆盖安装，测试安装等，对应-r,-t等带参数的 install 操作。

界面展示了app的图标，版本号，包名，更新时间等。

**应用图标怎么拿到的？**
网上大多数的方案是说抠出apk，使用apktool解包，找到图标文件，再拿来显示。可行的确可行，但是这个速度要等到天荒地老了。

因为我之前做过一个Android端的简单的app管理应用，我选择的路线是，提前在AndroidStudio里开发一个服务app，里面设置一个Service，启动后扫描所有的已安装的app，将应用图标，应用label，包名都存到Android本地。再将这个apk内置到DebugManager安装目录的resources目录下，将其安装进系统，准备好资源后，通过adb pull拉出所需要的资源到电脑端，再读取png文件来显示到界面上。

**单个app的操作**

![app_manage](/assets/img/blog/blogs_cmp_appmanage_2.png){:width="600" height="300" loading="lazy"}

对于选中的单个app，提供了打开应用界面，卸载，提取apk，对于系统应用，还可以push替换apk等操作。我们的测试同事在做非全量的发版验证时非常有用，不用再使用一条条繁琐的命令来替换apk升级了。

### 文件管理器
由于我在Android端也没有写过文件管理器应用，所以在这个页面，有些操作也是一拍脑袋想出来的，可能不算规范的解法。仍然是MVI架构，界面去监听StateHolder里面的UiState的Flow，切换目录时重新获取列表数据，update到界面来刷新UI。

![file_manage](/assets/img/blog/blogs_cmp_filemanage_1.png){:width="600" height="300" loading="lazy"}

最开始的展示列表我是直接执行了"ls /"将列表发送到界面，显示根目录，解析出其中的文件文件夹，继续往子目录的话就把路径拼接起来，比如进入sdcard，就执行"ls /sdcard"，继续深入则再次拼接。同时最上方设置了返回上级，回到根目录和priv-app快捷按钮。

展示文件列表的就是@Composable LazyComuln方法。

有意思的是，我在加入item的双击和单击的区分时，最初想给Modifier定义一个扩展方法，直接实现双击回调。但是发现必须经过clickable方法来实现，这样会把外部的单机的clickable给挤掉。所以双击判断还是写在了同一个clickable里面，通过时间间隔判断的工具类来区分，单击则选中对应的文件/文件夹，双击则进入文件夹。

```kotlin
modifier = Modifier.clickable {
    // 点击则设置即将操作的path
    MainStateHolder.setSelectedFilePath(it.path)
    androidSelectedFile = MainStateHolder.selectedFilePath
    // 双击，执行操作
    if (DoubleClickUtils.isFastDoubleClick()) {
        if (it.isDirectory)
            destinationCall(it.path)
        else
            println("点击文件：${it.path}")
    }
}
```
android内的文件操作也是使用命令行的形式，cp mv rm等。

还可以将文件pull到电脑端，将电脑端的文件推送到Android端等。

### 命令模式
![cmd](/assets/img/blog/blogs_cmp_cmdexecute.png){:width="600" height="300" loading="lazy"}

这一页比较简单，大家看到的输入框也是Compose原生的TextField方法，还自带动画，性价比蛮高。

主要实现就是将输入框的内容，拼接后直接通过Runtime.getRuntime().exec(command)执行即可。

除了最基础的adb命令透传，配合系统厂商Android端的可执行二进制程序，可以模拟车载信号的回调操作。还有语音部门的通过广播来调试的路径，整合到了DebugManager里面，一键发送广播，模拟可见扫描的点击。

### 关于页
![about](/assets/img/blog/blogs_cmp_about.png){:width="600" height="300" loading="lazy"}

最后就是关于页了，显示软件版本，缓存文件目录等。通过PlatformAdapter工具类获取路径，执行打开界面即可。

## 开源计划
这个软件最初是基于公司业务来设计开发的，有关于公司内部的信息需要抹除。
等后续有时间我会将其功能进行略微删减，改成通用性质的Android调试工具之后，会开源到Github。对CMP跨平台感兴趣的朋友，可以加关注稍作等待，后面一起进行技术交流。

12月25日已完成剥离修改开源：
[DebugManager开源地址](https://github.com/stepheneasyshot/DebugManager) 

## Material Design主题切换
目前进一步导入了两套主题方案，深色和浅色。

将最高级的 `Composable` 可组合项使用 `MaterialTheme` 包裹起来，初始化获取theme的值。主题值的下发设置在了 **关于页面** ，操作后的存储使用跨平台版本的 `DataStore` 来做键值对存储。

并通过StateHolder管理器来维护这个主题状态。在切换之后，最顶级的 Composable 组合项可以立即作出反应，切换色值资源。

```kotlin
 MaterialTheme(
            colors = when (themeState.value) {
                ThemeState.DARK -> DarkColorScheme
                ThemeState.LIGHT -> LightColorScheme
                else -> if (isSystemInDarkTheme()) DarkColorScheme else LightColorScheme
            }
        ) {
            SplashScreen {
                XXXXXXXXXXX
            }
        }
```

MainStateHolder.kt

```kotlin
// 主题
    private val _themeState = MutableStateFlow(ThemeState.DEFAULT)
    val themeStateStateFlow = _themeState.asStateFlow()
    private val themePreferencesKey = stringPreferencesKey("ThemeState")

    /**
     * 下发主题切换，存储在dataStore中
     */
    fun setThemeState(themeState: Int) {
        CoroutineScope(Dispatchers.IO).launch {
            dataStoreHelper.dataStore.edit {
                it[themePreferencesKey] = themeState.toString()
            }
        }
        _themeState.update {
            themeState
        }
    }

    /**
     * 获取本地存储的主题
     */
    fun getThemeState() {
        CoroutineScope(Dispatchers.IO).launch {
            dataStoreHelper.dataStore.data.collect {
                val themeState = it[themePreferencesKey]?.toInt() ?: ThemeState.DARK
                LogUtils.printLog("getThemeState-> themeState:$themeState", LogUtils.LogLevel.INFO)
                _themeState.update {
                    themeState
                }
            }
        }
    }
```

DataStoreHelper.kt

```kotlin
class DataStoreHelper {

    lateinit var dataStore: DataStore<Preferences>

    fun init(path: String) {
        dataStore = createDataStore(path)
    }

    private fun createDataStore(path: String): DataStore<Preferences> {
        return PreferenceDataStoreFactory.createWithPath(
            corruptionHandler = null,
            migrations = emptyList(),
            produceFile = { path.toPath() }
        )
    }
}
```

Main.kt

```kotlin
    val themeState = mainStateHolder.themeStateStateFlow.collectAsState()

    LaunchedEffect(Unit) {
        // 获取存储的主题设置
        mainStateHolder.getThemeState()
    }
```

### 运行截图记录
#### 开屏动画

![splash](/assets/img/blog/blogs_dark_splash.png)

![splash](/assets/img/blog/blogs_light_splash.png)

#### 设备信息

![device](/assets/img/blog/blogs_dark_deviceinfo.png)

![device](/assets/img/blog/blogs_light_deviceinfo.png)

#### 关于页

![about](/assets/img/blog/blogs_dark_about.png)

![about](/assets/img/blog/blogs_light_about.png)