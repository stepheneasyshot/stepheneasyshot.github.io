---
layout: post
description: > 
  本文介绍了四大组件之BroadcastReceiver的相关内容。
image: 
  path: /assets/img/blog/blogs_broadsact_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_broadsact_cover.png
    960w:  /assets/img/blog/blogs_broadsact_cover.png
    480w:  /assets/img/blog/blogs_broadsact_cover.png
accent_image: /assets/img/blog/blogs_broadsact_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 四大组件之BroadcastReceiver
## 什么是BroadcastReceiver
Android系统里面的广播可以看作村里的大喇叭。而BroadcastReceiver是Android四大组件之一，主要用于接收系统或其他应用发送的广播消息。

## 广播类型

* 标准广播（nor mal br oadcasts ）是一种完全异步执行的广播，在广播发出之后，所有的BroadcastR eceiver 几乎会在同一时刻收到这条广播消息，因此它们之间没有任何先后顺序可言。这种广播的效率会比较高，但同时也意味着它是无法被截断的。

    * 这是一种发散的，辐射状发送与接收。

* 有序广播（order ed br oadcasts ）则是一种同步执行的广播，在广播发出之后，同一时刻只会有一个BroadcastR eceiver 能够收到这条广播消息，当这个BroadcastR eceiver 中的逻辑执行完毕后，广播才会继续传递。所以此时的BroadcastR eceiver 是有先后顺序的，优先级高的BroadcastR eceiver 就可以先收到广播消息，并且前面的BroadcastR eceiver还可以截断正在传递的广播，这样后面的BroadcastR eceiver 就无法收到广播消息了。

    * 这是一种有序的，链式的发送与接收。

## BroadcastReceiver的作用

BroadcastReceiver的主要工作机制为：

1. 接收系统或其他应用发送的广播消息。
2. 处理接收到的广播消息。
3. 提供数据的安全性，使得数据只能被授权的应用访问。
4. 提供数据的可扩展性，使得数据可以被多个应用共享。

## 使用

### 静态注册
静态注册的方式是在AndroidManifest.xml文件中进行配置。在配置的过滤器中的广播消息发送之后，广播接收器就会接收到该广播消息。

```xml
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

### 动态注册

动态注册的方式是在代码中进行动态，用于在某个任务启动之后，动态监听某一个广播消息来做出反应。

```java
IntentFilter filter = new IntentFilter();
filter.addAction("android.intent.action.BOOT_COMPLETED");
registerReceiver(new MyBroadcastReceiver(), filter);
```

## 注意事项
广播接收器的动态注册方式，需要在任务销毁的时候，进行注销。一定需要是成对出现调用，否则会导致内存泄漏。

```java
// 注册监听
IntentFilter filter = new IntentFilter();
filter.addAction("android.intent.action.BOOT_COMPLETED");
registerReceiver(new MyBroadcastReceiver(), filter);

// 注销监听
unregisterReceiver(new MyBroadcastReceiver());
```

## 广播的发送
### 标准广播的发送方式

先定义一个接收器，然后再发送广播测试。

```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {
  override fun onReceive(context: Context, intent: Intent) {
    Toast.makeText(context, "received in MyBroadcastReceiver",Toast.LENGTH_SHORT).show()
  }
} 
```

Manifest文件注册：

```xml
<receiver
 android:name=".MyBroadcastReceiver"
 android:enabled="true"
 android:exported="true">
 <intent-filter>
 <action android:name="com.example.broadcasttest.MY_BROADCAST"/>
 </intent-filter>
 </receiver> 
```

然后在Activity中发送广播：

```kotlin
class MainActivity : AppCompatActivity() {
    ...
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        button.setOnClickListener {
            val intent = Intent("com.example.broadcasttest.MY_BROADCAST")
            intent.setPackage(packageName)
            sendBroadcast(intent)
        }
        ...
    }
    ...
} 
```

点击按钮之后，就可以看到Toast的提示了。

### 有序广播的发送方式

有序广播的发送方式和标准广播的发送方式类似，只不过需要在发送广播的时候，指定广播的优先级。

```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {
  override fun onReceive(context: Context, intent: Intent) {
    Toast.makeText(context, "received in MyBroadcastReceiver",Toast.LENGTH_SHORT).show()
  } 
}
```

Manifest文件注册：

```xml
<receiver
 android:name=".MyBroadcastReceiver"
 android:enabled="true"
 android:exported="true">
 <intent-filter android:priority="1000">
 <action android:name="com.example.broadcasttest.MY_BROADCAST"/>
 </intent-filter>
 </receiver>
```

然后在Activity中发送广播：

```kotlin
class MainActivity : AppCompatActivity() {
   ...
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        button.setOnClickListener {
            val intent = Intent("com.example.broadcasttest.MY_BROADCAST")
            intent.setPackage(packageName)
            sendOrderedBroadcast(intent, null)
        }  
    } 
}
```

可以看到，区别就是sendOrderedBroadcast方法。

如果有多个广播接收器接收这一条广播，那么优先级高的广播接收器会先接收到广播。注意，priority数值越大，优先级越高。
