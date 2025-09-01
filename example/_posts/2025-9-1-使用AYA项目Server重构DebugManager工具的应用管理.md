---
layout: post
description: > 
  本文记录了跨平台Android设备调试工具DebugManager，在使用AYA项目重构应用管理功能的流程与分析
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
# 使用AYA项目Server重构DebugManager工具的应用管理
这个工具绝大部分功能都是基于 `adb` 命令来实现的，在ProcessBuilder中执行命令，获取输出流，对输出的内容进行解析，在结构化地展示到界面。

然而没有直接获取应用图标文件这样的命令，只能从android内部下手，获取png文件再发送到Desktop端。本文介绍一下应用管理功能，图标文件获取这个实现的发展历程和最新的使用AYA服务端更改的版本。
## 发展历程
### 一、车机阶段
这个工具在最初开发的时候，是面向车机平台。

由于我可以获取到我们平台的系统签名文件，所以是直接编写了一个带系统签名的APP，内部有一个服务，在 `onCreate()` 中获取安装所有的应用信息：

* label
* packageName
* versionName
* versionCode
* lastUpdateTime
* ...

最麻烦的需要存储的是应用的ICON图标文件，因为想要在电脑端的JVM应用 `DebugManager` 中显示应用的图标，是没有adb命令可以使用的，只能从Android平台上获取到 `Bitmap` 数据，转换为png存储下来。最后拉取到Desktop端，读取文件显示图标。

编写玩服务代码，在安装这个APK后，通过 `am startService` 来运行这个服务，服务端获取信息，将所有信息准备好后再存储到应用内部路径的文件中。

```
/data/data/com.stephen.appinfoservice/files/appinfo.json
/data/data/com.stephen.appinfoservice/icons/*.png
```

![](/assets/img/blog/blogs_cmp_appmanage_1.png)

### 二、手机阶段
在面对更普遍的手机端的应用管理功能时，我发现了没有系统签名的应用，是不可以直接使用 `am startservice` 命令来拉起服务的。而且不同手机平台还有权限上面的区别，比如类原生的系统上（Pixel平台，LineageOS），是不用动态申请就可以获取安装的应用信息，但是在国产OS上，有更严格的管理，需要处理读取应用列表权限。

当然可以专门做一个Activity来交互，让用户同意读取应用列表的权限。但是我更希望这个过程是无感的。

所以还是要探索其他的办法。
## AYA项目服务端
AYA也是一个基于adb命令来显示Android设备信息，进行调试的项目，同样为跨平台的产品形态。

它是使用比较火的Electron框架，`TypeScript` 语言编写的。看到他们有类似的应用管理页面，可以显示APP图标，我想他们肯定也是在Android设备上有一个服务端的。看看他们服务端的实现方式，并将其改写成适配 `DebugManager` 项目的数据传输方案。

![](/assets/img/blog/blogs_aya_screenshots.png)

AYA的服务端不是承载于一个普通的Android应用进程上，而是类似于一个DEAMON守护进程。

![](/assets/img/blog/blogs_aya_server_code.png)

### 代码运行分析
这是一个 Android 应用，但它并不是一个常规的 APK，而是一个在 Android 设备上通过 `dex` 文件直接运行的程序。这种方式通常用于需要更高权限或者直接访问系统服务的工具类应用，类似于 adb shell 上的一个服务。

在项目的Gradle脚本中，配置了一个任务，这个任务会将项目的代码build生成的apk里面的 `dex` 文件提取出来，然后通过 `adb` 命令，推送到设备的 `data/local/tmp` 路径下，使用 `CLASSPATH` 指定运行。

```groovy
android.applicationVariants.all { variant ->
    variant.outputs.all {
        outputFileName = "aya-server-v${versionName}.apk"

        def dexPath = rootProject.rootDir.path
        variant.assembleProvider.get().doLast {
            copy {
                def file = zipTree(file(outputFile)).matching { include 'classes*.dex' }.singleFile

                from file
                into dexPath
                rename { String fileName ->
                    fileName.replace(file.getName(), "aya.dex")
                }
            }
        }
    }
}
```

从整个流程看，这个项目是一个用于在 Android 设备上获取和管理应用信息的工具。它利用了 **`LocalSocket`** 进行进程间通信，通过 **反射** 访问 Android 隐藏的 API 来获取详细的系统和应用信息，并将这些信息缓存到设备的文件系统中。

对原来的代码进行了改写，AYA项目继承了protobuf通信框架，直接将所有的应用信息，全部通过protobuf协议进行传输，包括图标文件（base64编码后）。

> protobuf是Google开发的一种语言无关、平台无关、可扩展的序列化数据结构的方法，它可以将数据结构序列化为二进制格式，用于在网络上传输或存储。

我选择了更简单的通信方案，直接使用JSON字符串，在客户端使用Kotin序列化解析。然后服务端在运行时，将图标png存储到固定位置，拉取到电脑端再显示。这种方式打包的dex文件体积缩小了90%。

代码简要分析：

#### 服务入口 `Server.kt`
`Server.kt` 的 `main` 方法是整个程序的入口点。

```kotlin
fun start(args: Array<String>) {
    Log.i(TAG, "Start server")

    val server = LocalServerSocket("aya")
    Log.i(TAG, "Server started, listening on ${server.localSocketAddress}")

    while (true) {
        val conn = Connection(server.accept())
        Log.i(TAG, "Client connected")
        executor.submit(conn)
    }
}
```

* `Server.main` 方法被调用，创建一个 `Server` 实例并调用其 `start` 方法。
* `start` 方法首先创建一个名为 `"aya"` 的 `LocalServerSocket`。`LocalSocket` 是 Android 上一种特殊的 IPC（Inter-Process Communication，进程间通信）机制，它允许同一个设备上的不同进程通过本地套接字进行通信。
* 服务器进入一个无限循环 `while (true)`，等待客户端连接。
* 每当有客户端连接时 (`server.accept()`)，服务器会创建一个新的 `Connection` 实例，并将该实例提交给一个缓存线程池 `executor` 来执行。这意味着每个客户端连接都会在一个独立的线程中处理，从而实现了并发处理。

#### 请求处理 `Connection.kt`
`Connection.kt` 负责与单个客户端进行通信。

* `run` 方法获取客户端的输入流和输出流，用于读取请求和发送响应。
* 它进入一个循环，从输入流中读取客户端发送的 JSON 字符串请求。
* 读取到的 JSON 字符串被解析，然后调用 `handleRequest` 方法来处理具体的请求。

`handleRequest` 方法根据请求中的 `method` 字段来分发不同的处理逻辑。

```kotlin
when (method) {
    "getVersion" -> {
        put("version", getVersion())
    }

    "getPackageInfos" -> {
        put("packageInfos", getPackageInfos(JSONObject(params)))
    }

    "saveAllInfoToFile" -> {
        put("saveResult", saveAllInfoToFile(params))
    }

    else -> {
        Log.e(TAG, "Unknown method: $method")
        put("error", "Unknown method: $method")
    }
}
```

调用的 `getPackageInfo` 方法是核心逻辑，它利用反射机制获取系统服务并查询应用信息。

* **获取系统服务**:
    * 通过 `ServiceManager.packageManager` 和 `ServiceManager.storageStatsManager` 获取 `PackageManager` 和 `StorageStatsManager` 的实例。
    * `ServiceManager.kt` 中使用反射调用了隐藏的 `android.os.ServiceManager.getService` 方法，从而获取系统的 `IPackageManager` 和 `IStorageStatsManager` 服务。这种方式通常需要特殊的权限或者在 root 环境下才能成功。
* **查询信息**:
    * 使用 `ServiceManager.packageManager.getPackageInfo` 获取 `PackageInfo` 对象，其中包含了应用版本、安装时间等基本信息。
    * 从 `PackageInfo` 中获取 `ApplicationInfo`，进而获取应用的 `apkPath` 和 `flags`。通过 `flags` 判断应用是否为系统应用。
    * 如果设备版本在 Android 8.0（Oreo）及以上，会使用 `ServiceManager.storageStatsManager.queryStatsForPackage` 获取应用的存储统计信息，包括应用数据大小、缓存大小等。
* **获取应用名称和图标**:
    * 利用 `apkPath`，通过反射创建 `AssetManager` 并加载 APK 资源，然后获取应用的名称（label）和图标（icon）。
    * 如果图标存在，会将其转换为 PNG 格式并保存到 `/data/local/tmp/aya/icons` 目录下。
* **组织和返回数据**:
    * 所有获取到的信息都被封装成一个 `JSONObject` 返回。

#### 生命周期与数据流转
普通 Android 应用（APK）的生命周期是由系统 **PackageManager** 和 **ActivityManager** 严格管理的。而这个服务端则是一个 **"裸" 进程**。

它的生命周期管理方式更接近于一个传统的 Linux 守护进程（daemon）。它没有标准的 Android 应用入口点（如 Launcher Activity）。它通常需要通过 `adb shell` 或其他特殊工具（如 Magisk 模块）来手动启动，命令通常类似于 `app_process /system/bin io.liriliri.aya.Server`。

它的生命周期完全由启动它的进程控制。只要启动它的 shell 进程或父进程不被终止，这个服务端进程就会一直运行。它没有 `onStop`、`onPause` 等 Android 生命周期回调。由于它不是一个常规的应用进程，系统不会像管理普通应用那样主动去管理它。如果它没有被其他应用组件绑定，系统通常不会轻易终止它，除非设备内存极度紧张。

使用 **反射** 来调用系统隐藏的 API 。这种方式绕过了标准的权限检查。

这个服务端运行在**一个单独的 JVM（Java Virtual Machine）进程**上。

具体来说：

* **独立的进程**: AYA的服务端代码有一个 `main` 方法在 `Server.kt` 中。在 Android 平台上，当一个 Java/Kotlin 程序通过 `CLASSPATH` 运行并带有 `main` 方法时，它会被系统启动为一个独立的进程。这个进程会拥有自己的 Dalvik 或 ART（Android Runtime）虚拟机实例。
* **本地套接字服务器**: 这个进程创建了一个 `LocalServerSocket`。`LocalSocket` 是一种 Android 内部的 IPC（进程间通信）机制，它允许同一个设备上的不同进程进行通信。你的服务端进程会监听这个套接字，等待其他客户端进程（例如，一个通过 `adb shell` 启动的客户端，或者一个单独的 Android 应用）来连接。

服务端程序可以被看作是一个后台服务，它作为一个独立的进程在 Android 系统中运行，并通过本地套接字提供服务。运行成功后，再通过 `adb forward tcpip:xxxx localabstract:aya` 命令，把这个设备内部的本地套接字通信，映射到电脑的 TCP/IP 端口上。

以下是完整的通信流程：

* 你的电脑上的客户端程序（例如，一个 Python 脚本或一个 C++ 程序）会尝试连接到 `localhost:xxxx`。这个连接请求是一个标准的 TCP/IP 连接请求。
* 当 `adb` 发现一个连接到你电脑 `xxxx` 端口的请求时，它会拦截这个请求。`adb` 作为一个桥梁，将这个 TCP/IP 流量通过 USB 数据线或 Wi-Fi 发送到你连接的 Android 设备。
* 设备端的 `adb` 守护进程（`adbd`）接收到这个连接请求。它会识别出这个请求是为 `localabstract:aya` 设定的转发规则。
* `adbd` 守护进程在设备上作为一个新的客户端，主动去连接 `localabstract:aya` 这个本地套接字。
* `Server.kt` 里的 `LocalServerSocket` 正在监听 `localabstract:aya`。当 `adbd` 发出连接请求时，`server.accept()` 会返回一个与 `adbd` 建立连接的 `LocalSocket`。

基于以上分析路线，电脑上的客户端发送的所有数据，都会通过以下路径传输：

**PC 客户端** → **PC 的 `adb`** → **USB 数据线/Wi-Fi** → **设备上的 `adbd`** → **`localabstract:aya` 本地套接字** → **你的服务端进程 (`Server.kt`)**

同时，Android服务端发送的响应数据也会通过这条路径原路返回。

这种方式允许你像调试一个常规的网络服务一样，直接在电脑上与运行在 Android 设备上的程序进行交互。你不需要在设备上安装一个完整的网络服务器，也不需要担心防火墙或其他网络配置问题。`adb` 巧妙地为你解决了跨进程、跨设备的通信问题，将本地 IPC 流量无缝地转发到了你的电脑上。

### DebugManager对接
电脑端的代码，在数据处理上几乎没有改动，将原来的安装apk流程和启动服务流程，改为了推送dex文件到设备，使用

```bash
CLASSPATH=/data/local/tmp/aya/aya.dex app_process /system/bin io.liriliri.aya.Server
```

来启动服务，再通过`adb forward tcpip:xxxx localabstract:aya`命令，把这个设备内部的本地套接字通信，映射到电脑的TCP/IP端口上。

DebugManager内部，通过 `adb shell pm list packages` 命令，获取到所有安装的应用列表。解析出每一个包名，再调用服务端的 `getPackageInfo` 方法，获取应用label，版本信息。最后通过coil的AsyncImage组件，填入icon文件路径，异步加载图标。

## 界面升级
此前的交互也进行了同步升级，再有限的窗口内展示更多的信息，全局缩小了字号和模块之前的padding，将列表类型改为了图标矩阵，使用 `LazyVerticalGrid` 组件来承接应用图标展示。

![](/assets/img/blog/blogs_debugmanager_app_info.png)

同时为了缩小重组范围，使用应用的packageName作为key，来标识每一个item，还可以以此来实现每一个item的移动动效，比图标的体验更丝滑。

```kotlin
LazyVerticalGrid(columns = GridCells.Adaptive(minSize = 105.dp)) {
    items(appListState.sortedBy { it.label }, key = { it.packageName }) {
        Box(
            Modifier.animateItem(
                fadeInSpec = null,
                fadeOutSpec = null,
                placementSpec = tween(300)
            )
        ) {
            GridAppItem(
                label = it.label,
                iconFilePath = mainStateHolder.getIconFilePath(it.packageName),
                modifier = Modifier.padding(5.dp)
                    .size(100.dp).clip(RoundedCornerShape(10))
                    .padding(5.dp)
                    .bounceClick().clickable(
                        indication = null,
                        interactionSource = remember { MutableInteractionSource() }
                    ) {
                        dialogInfoItem.value = it
                    },
                onClickShowInfo = {
                    dialogInfoItem.value = it
                },
                onClickOpen = {
                    mainStateHolder.startMainActivity(it.packageName)
                },
                onForceStop = {
                    mainStateHolder.forceStopApp(it.packageName)
                },
                onExtractApk = {
                    mainStateHolder.pullInstalledApk(it.packageName, it.versionName)
                },
            )
        }
    }
}
```

在动效方面，直接使用 `animateItem` 函数就可以实现列表项的移动动效。

```kotlin
Modifier.animateItem(
    fadeInSpec = null,
    fadeOutSpec = null,
    placementSpec = tween(300)
)
```

![](/assets/img/blog/blogs_debugmanager_app_info_animate.gif)

