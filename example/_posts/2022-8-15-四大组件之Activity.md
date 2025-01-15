---
layout: post
description: > 
  本文为Activity任务栈和四种启动模式的介绍
image: 
  path: /assets/img/blog/blogs_activity_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_activity_cover.png
    960w:  /assets/img/blog/blogs_activity_cover.png
    480w:  /assets/img/blog/blogs_activity_cover.png
accent_image: /assets/img/blog/blogs_activity_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Activity任务栈和启动模式

安卓四大组件分别是Activity、Service、Broadcast Receiver和Content Provider。以下是对它们的总结：

Activity（活动）：

作用：Activity是安卓应用中最基本的组件，用于实现用户界面。它提供了一个屏幕，用户可以在其中进行交互，如查看信息、输入数据、执行操作等。
特点：
一个Activity通常对应一个屏幕的内容。
可以通过Intent进行启动和切换。
具有生命周期，从创建到销毁会经历多个状态。
启动方式：
显式启动：通过指定目标Activity的类名来启动。
隐式启动：通过Intent的Action、Category等信息来匹配符合条件的Activity。
Service（服务）：

作用：Service用于在后台执行长时间运行的操作，不提供用户界面。它可以在不影响用户与应用交互的情况下，执行诸如音乐播放、文件下载、数据同步等任务。
特点：
运行在后台，不与用户直接交互。
可以通过startService()或bindService()方法启动。
具有生命周期，从创建到销毁会经历多个状态。
启动方式：
startService()：启动一个独立的Service，即使启动它的组件被销毁，Service仍会继续运行。
bindService()：将Service与启动它的组件绑定，组件可以与Service进行通信，当组件销毁时，Service也会随之销毁。
Broadcast Receiver（广播接收器）：

作用：Broadcast Receiver用于接收系统或应用发出的广播消息，并做出相应的响应。它可以用于实现应用内的事件通知、系统事件监听等功能。
特点：
可以注册为静态（在AndroidManifest.xml中声明）或动态（在代码中注册）。
可以接收系统广播（如电池电量变化、网络连接变化等）和自定义广播。
接收到广播后，会执行onReceive()方法进行处理。
注册方式：
静态注册：在AndroidManifest.xml中声明<receiver>标签，并指定要接收的广播类型。
动态注册：在代码中通过Context.registerReceiver()方法注册，并指定要接收的广播类型。
Content Provider（内容提供者）：

作用：Content Provider用于在不同的应用之间共享数据。它提供了一种标准的接口，使得应用可以安全地访问和操作其他应用的数据。
特点：
可以将应用的数据暴露给其他应用。
可以实现数据的增删改查操作。
可以通过ContentResolver来访问Content Provider提供的数据。
使用方式：
定义Content Provider：在应用中创建一个继承自ContentProvider的类，并实现其抽象方法。
访问Content Provider：通过ContentResolver的query()、insert()、update()、delete()等方法来操作Content Provider提供的数据。
安卓四大组件是安卓应用开发的核心，它们各自承担着不同的功能，通过合理地组合和使用这些组件，可以构建出功能丰富、交互性强的安卓应用。


## 代码说明
Activity（活动）
显式启动：

2022-8-15-四大组件之Activity.md
Apply
// 显式启动目标Activity
val intent = Intent(this, TargetActivity::class.java)
startActivity(intent)
隐式启动：

2022-8-15-四大组件之Activity.md
Apply
// 隐式启动符合条件的Activity
val intent = Intent()
intent.action = "com.example.ACTION_VIEW"
intent.addCategory(Intent.CATEGORY_DEFAULT)
startActivity(intent)
Service（服务）
startService()：

2022-8-15-四大组件之Activity.md
Apply

新功能


👋🏼 告别复制粘贴，使用 Apply 让 AI 帮您写代码吧！
AI 生成代码后，点击 Apply 将代码一键插入目标文件的预期位置，并形成 Diff 供您审阅，AI 编程体验更流畅！使用说明↗
// 启动一个独立的Service
val intent = Intent(this, MyService::class.java)
startService(intent)
bindService()：

2022-8-15-四大组件之Activity.md
Apply
// 绑定Service并与之通信
val intent = Intent(this, MyService::class.java)
bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)

private val serviceConnection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        // 与Service建立连接后的操作
    }

    override fun onServiceDisconnected(name: ComponentName?) {
        // 与Service断开连接后的操作
    }
}
Broadcast Receiver（广播接收器）
静态注册：
在AndroidManifest.xml中声明：


xml
Apply
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
动态注册：

2022-8-15-四大组件之Activity.md
Apply
// 动态注册广播接收器
val intentFilter = IntentFilter()
intentFilter.addAction("com.example.ACTION_CUSTOM")

val receiver = MyBroadcastReceiver()
registerReceiver(receiver, intentFilter)
Content Provider（内容提供者）
定义Content Provider：

2022-8-15-四大组件之Activity.md
Apply
class MyContentProvider : ContentProvider() {
    override fun onCreate(): Boolean {
        // 初始化Content Provider
        return true
    }

    override fun query(
        uri: Uri,
        projection: Array<String>?,
        selection: String?,
        selectionArgs: Array<String>?,
        sortOrder: String?
    ): Cursor? {
        // 处理查询请求
        return null
    }

    // 其他方法的实现...
}
访问Content Provider：

2022-8-15-四大组件之Activity.md
Apply
// 通过ContentResolver访问Content Provider
val contentResolver = contentResolver
val cursor = contentResolver.query(
    Uri.parse("content://com.example.provider/data"),
    null,
    null,
    null,
    null
)
以上示例展示了安卓四大组件的基本使用方法，你可以根据实际需求进行扩展和修改。