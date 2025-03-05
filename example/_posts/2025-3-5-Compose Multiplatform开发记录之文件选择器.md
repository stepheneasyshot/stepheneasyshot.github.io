---
layout: post
description: > 
  本文介绍了CMP开发过程中，对Desktop端文件和文件夹选取的控件的封装
image: 
  path: /assets/img/blog/blogs_cmp_debugmanager_splash.png
  srcset: 
    1920w: /assets/img/blog/blogs_cmp_debugmanager_splash.png
    960w:  /assets/img/blog/blogs_cmp_debugmanager_splash.png
    480w:  /assets/img/blog/blogs_cmp_debugmanager_splash.png
accent_image: /assets/img/blog/blogs_cmp_debugmanager_splash.png
excerpt_separator: <!--more-->
sitemap: false
---
# Compose Multiplatform开发记录之文件选择器
刚刚写完一篇TextField输入框按键监听的文章，趁热打铁，记录一下我简单封装桌面端的的文件选择器组件。依然是来自于跨平台Android设备调试软件DebugManager里的功能。这里相关的是apk文件的选取安装，电脑文件的推送，日志文件的选取自动分析等。

## 文件选择

文件选择为java.awt包下的FileDialog组件，初始化显示完，直接通过实例化的 `FileDialog` 对象来获取最终的文件选择路径。有directory和file两部分。

```kotlin
val fileChooser = FileDialog(
    Frame(),
    "Select a file",
    FileDialog.LOAD
).apply {
    file = fileType
}
fileChooser.isVisible = true
// 判断是否未选文件
if (fileChooser.file != null) {
    onPathSelect(fileChooser.directory + fileChooser.file)
}
```

## 文件夹选择器

文件夹这里用的是swing包下的 `JFileChooser` ，用法几乎和上面的 `FileDialog` 一样。

```kotlin
// 选择文件夹
val fileChooser = JFileChooser()
fileChooser.fileSelectionMode = JFileChooser.DIRECTORIES_ONLY
// 显示对话框并等待用户选择
val result = fileChooser.showOpenDialog(null);
// 如果用户选择了文件夹
if (result == JFileChooser.APPROVE_OPTION) {
    // 获取用户选择的文件夹
    onPathSelect(fileChooser.selectedFile.absolutePath)
}
```

这两种选取电脑文件的方法，之前都是在一个Text组件的 `clickable` 回调里面来配置的，把结果赋值给一个String泛型的State，再执行文件的推送或者apk的安装。

## 拖动选择文件

之前看另一位博主也提供了个拖拽的方法 `onExternalDrag` ，可惜已经严重过期了，弃用无法使用。

```kotlin
@Deprecated(
    level = DeprecationLevel.ERROR,
    message = "Use the new drag-and-drop API: Modifier.dragAndDropTarget"
)
@Suppress("DEPRECATION_ERROR")
@ExperimentalComposeUiApi
@Composable
fun Modifier.onExternalDrag(
    enabled: Boolean = true,
    onDragStart: (ExternalDragValue) -> Unit = {},
    onDrag: (ExternalDragValue) -> Unit = {},
    onDragExit: () -> Unit = {},
    onDrop: (ExternalDragValue) -> Unit = {},
)
```

根据这里的提示“Use the new drag-and-drop API: Modifier.dragAndDropTarget”

```kotlin
@ExperimentalFoundationApi
fun Modifier.dragAndDropTarget(
    shouldStartDragAndDrop: (startEvent: DragAndDropEvent) -> Boolean,
    target: DragAndDropTarget,
): Modifier {
    return this then DropTargetElement(
        target = target,
        shouldStartDragAndDrop = shouldStartDragAndDrop,
    )
}
```

根据官方文档和方法签名，了解到这个方法的具体用法。

> 第一个参数为一个 DragAndDropEvent 类型的参数，并返回一个布尔值。
> 作用：这个函数允许可组合项（Composable）根据启动拖放会话的 DragAndDropEvent 来决定是否要参与该拖放会话。当一个拖放操作开始时，系统会调用这个函数，传入表示拖放开始事件的 DragAndDropEvent 对象。如果该函数返回 true，则表示当前可组合项愿意参与这个拖放会话；如果返回 false，则表示不参与。

> 作用：这个对象是拖放会话的目标，它将接收与拖放会话相关的事件。当拖放操作发生在当前可组合项上时，系统会将相关的拖放事件发送给这个 DragAndDropTarget 对象，以便进行相应的处理。

可以理解为一个enable开关，一个callback回调。我们的关注度应该放到calllback回调事件上，主要目的就是需要拖动过来的文件路径。

```kotlin
val callback = remember {
    object : DragAndDropTarget {
        override fun onDrop(event: DragAndDropEvent): Boolean {
            val dragData = event.dragData()
            if (dragData is DragData.FilesList) {
                dragData.readFiles().firstOrNull()?.let { filePath ->
                    val file = File(URI.create(filePath))
                    LogUtils.printLog("选取文件：${file.absolutePath}")
                    if (fileType.isNotEmpty() && fileType.split('.').last() != file.extension) {
                        onErrorOccur("请选择正确的文件类型")
                        return false
                    }
                    onPathSelect(file.absolutePath)
                }
            }
            return true
        }
    }
}
```

在 `onDrop` 即鼠标拖动松手后，解析文件路径出来，判断是否是我们需要的。

## 三合一封装

为了统一设计，并且使用一个组件支持以上三种文件功能，对一个Text组件进行包装。

配置的几个参数如下代码所示，一个提示语字段，一个string类型的文件路径和其更改的lambda，一个变量用来判断是需要接受文件还是文件夹，一个为需要的文件类型。最后的onError代码块为拖动来的文件不符合要求时，供调用方弹出Toast所用。

```kotlin
/**
 * @param tintText 提示文本
 * @param path 路径
 * @param onPathSelect 路径选择回调
 * @param isChooseFile 是否选择文件，默认为 false
 * @param fileType 文件类型
 * @param onErrorOccur 错误消息回调
 */
@OptIn(ExperimentalFoundationApi::class, ExperimentalComposeUiApi::class)
@Composable
fun FileChooseWidget(
    tintText: String,
    path: String,
    modifier: Modifier = Modifier,
    isChooseFile: Boolean = false,
    fileType: String = "",
    onErrorOccur: (String) -> Unit = {},
    onPathSelect: (String) -> Unit,
) {

    val callback = remember {
        object : DragAndDropTarget {
            override fun onDrop(event: DragAndDropEvent): Boolean {
                val dragData = event.dragData()
                if (dragData is DragData.FilesList) {
                    dragData.readFiles().firstOrNull()?.let { filePath ->
                        val file = File(URI.create(filePath))
                        LogUtils.printLog("选取文件：${file.absolutePath}")
                        if (fileType.isNotEmpty() && fileType.split('.').last() != file.extension) {
                            onErrorOccur("请选择正确的文件类型")
                            return false
                        }
                        onPathSelect(file.absolutePath)
                    }
                }
                return true
            }
        }
    }

    CenterText(
        text = path.ifEmpty { tintText },
        style = defaultText,
        modifier = modifier.border(2.dp, MaterialTheme.colorScheme.onSecondary, RoundedCornerShape(10.dp))
            .clip(RoundedCornerShape(10.dp))
            .background(MaterialTheme.colorScheme.secondary).clickable {
                // 选择文件
                if (isChooseFile) {
                    val fileChooser = FileDialog(
                        Frame(),
                        "Select a file",
                        FileDialog.LOAD
                    ).apply {
                        file = fileType
                    }
                    fileChooser.isVisible = true
                    // 判断是否未选文件
                    if (fileChooser.file != null) {
                        onPathSelect(fileChooser.directory + fileChooser.file)
                    }
                } else {
                    // 选择文件夹
                    val fileChooser = JFileChooser()
                    fileChooser.fileSelectionMode = JFileChooser.DIRECTORIES_ONLY
                    // 显示对话框并等待用户选择
                    val result = fileChooser.showOpenDialog(null);
                    // 如果用户选择了文件夹
                    if (result == JFileChooser.APPROVE_OPTION) {
                        // 获取用户选择的文件夹
                        onPathSelect(fileChooser.selectedFile.absolutePath)
                    }
                }
            }.dragAndDropTarget(
                shouldStartDragAndDrop = { event -> true },
                target = callback
            ).padding(10.dp)
    )
}
```

## 实现的效果如下所示

### 拖动文件

![blogs_cmp_debugmanager_file_drag.webp](/assets/img/blog/blogs_cmp_debugmanager_file_drag.webp)

### 点击触发选择窗

![blogs_cmp_debugmanager_file_choose.webp](/assets/img/blog/blogs_cmp_debugmanager_file_choose.webp)
