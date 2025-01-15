---
layout: post
description: > 
  本文介绍了四大组件之Activity的相关内容。
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
# 四大组件之Activity

## 概念
Activity是最常用的四大组件之一，主要用于显示用户界面，管理维护界面内的一系列控件和数据，并响应用户的屏幕操作事件。

## 创建
新建项目时可以自动选择Activity模板。Android Studio会自动帮我们生成Activity的代码，我们只需要在Activity的子类中实现处理逻辑即可。

此外，也可以在后续的实现过程中手动新建新的Activity类，这种方式需要手动修改AndroidManifest.xml文件，将新的Activity注册到清单文件中。

插入，Manifest文件的作用是：

1. 声明应用程序的组件，如Activity、Service、BroadcastReceiver等。
2. 配置应用程序的权限，如网络访问权限、存储访问权限等。
3. 配置应用程序的启动Activity。
4. 配置应用程序的主题。

每一个Activity都会绑定一个xml布局文件，加载自己的内容区域里面。

以下是我项目中的一个Activity代码：

```java
public class MainActivity extends AppCompatActivity {
    private Fragment loginfragment, signupfragment;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        loginfragment = new LoginFragment();
        signupfragment = new SignupFragment();

        getSupportFragmentManager().beginTransaction().add(R.id.fragment_container, loginfragment).commit();
    }
}
```

## 启动
Activity的启动方式有两种：

1. 显式启动。在代码中使用Intent对象指定要启动的Activity类，然后调用startActivity()方法启动Activity。
2. 隐式启动。在代码中使用Intent对象指定要启动的Activity的动作和类别，然后调用startActivity()方法启动Activity。

代码示例：

```java
// 显式启动
Intent intent = new Intent(this, MyActivity.class);
startActivity(intent);

// 隐式启动
Intent intent = new Intent("com.example.myaction");
intent.addCategory("com.example.mycategory");
startActivity(intent);
```

隐式启动在高版本Android中已经被弃用，建议使用显式启动。

## 传递数据
Activity之间的数据传递可以使用Intent对象。Intent对象可以携带数据，包括基本数据类型、字符串、对象等。

代码示例：

```java
// 传递基本数据类型
Intent intent = new Intent(this, MyActivity.class);
intent.putExtra("key", value);
startActivity(intent);

// 传递字符串
Intent intent = new Intent(this, MyActivity.class);
intent.putExtra("key", "value");
startActivity(intent);

// 传递对象
Intent intent = new Intent(this, MyActivity.class);
intent.putExtra("key", new MyObject());
startActivity(intent);
```

## 接收数据
Activity被启动后，可以在onCreate方法里面，检查Intent对象是否携带数据，然后获取数据。
如果Activity已经在前台，而外部又有启动这个Activity的操作，我们可以在onNewIntent方法里面，检查Intent对象是否携带数据，然后获取数据。


代码示例：

```java
public class MyActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my);  

        // 检查intent对象是否携带数据
        if (getIntent().hasExtra("key")) {
            // 获取数据
            String value = getIntent().getStringExtra("key");
        }
    } 

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);

        // 检查intent对象是否携带数据
        if (intent.hasExtra("key")) {
            // 获取数据
            String value = intent.getStringExtra("key");
        }
    }
}
```

## 销毁
Activity的销毁可以使用finish()方法。

代码示例：

```java 
public class MyActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my);

        // 通过一个按钮来销毁Activity
        Button button = findViewById(R.id.button);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                finish();
            }
        });
    } 
}

```

## Activity的状态
根据以上的简要流程，可以总结出Activity有四种状态：

1. 运行状态。Activity位于返回栈的栈顶，并且处于运行状态。
2. 暂停状态。Activity不再位于返回栈的栈顶，但是仍然可见，比如Activity被覆盖住了。
3. 停止状态。Activity不再位于返回栈的栈顶，并且不可见，比如Activity被另一个Activity覆盖住了。
4. 销毁状态。Activity不再位于返回栈的栈顶，并且被系统回收了。

## 启动模式
Activity的启动模式有四种：

1. standard。默认启动模式，每次启动Activity都会创建一个新的实例。
2. singleTop。如果Activity已经位于返回栈的栈顶，那么不会创建新的实例，而是复用已经存在的实例。
3. singleTask。如果Activity已经在返回栈中存在，那么不会创建新的实例，而是复用已经存在的实例，并将该Activity之上的所有Activity出栈。
4. singleInstance。如果Activity已经在返回栈中存在，那么不会创建新的实例，而是复用已经存在的实例，并将该Activity之上的所有Activity出栈。

可以在Manifest文件中配置Activity的启动模式，也可也在代码中拉起Activity的时候设置启动模式。

代码示例：

```xml
<activity android:name=".MyActivity" android:launchMode="singleTop" />
```

java代码示例：

```java
Intent intent = new Intent(this, MyActivity.class);
intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
startActivity(intent);
```

## Activity的生命周期

Activity 类中定义了7个回调方法，覆盖了Activity 生命周期的每一个环节：

* onCreate()。这个方法你已经看到过很多次了，我们在每个Activity 中都重写了这个方法，它会在Activity 第一次被创建的时候调用。你应该在这个方法中完成Activity 的初始化操作，比如加载布局、绑定事件等。
* onStart()。这个方法在Activity 由不可见变为可见的时候调用。onResume()。这个方法在Activity 准备好和用户进行交互的时候调用。此时的Activity 一定位于返回栈的栈顶，并且处于运行状态。
* onPause()。这个方法在系统准备去启动或者恢复另一个Activity 的时候调用。我们通常会在这个方法中将一些消耗CPU的资源释放掉，以及保存一些关键数据，但这个方法的执行速度一定要快，不然会影响到新的栈顶Activity 的使用。
* onStop()。这个方法在Activity 完全不可见的时候调用。它和onPause()方法的主要区别在于，如果启动的新Activity 是一个对话框式的Activity ，那么onPause()方法会得到执行，而onStop()方法并不会执行。
* onDestroy()。这个方法在Activity 被销毁之前调用，之后Activity 的状态将变为销毁状态。
* onRestart()。这个方法在Activity 由停止状态变为运行状态之前调用，也就是Activity被重新启动了。

代码示例，在生命周期回调方法里面加入Log打印，感知运行流程：

```java
public class MyActivity extends Activity {
    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG, "onCreate()");
    } 
    @Override
    public void onStart() {
        super.onStart();
        Log.i(TAG, "onStart()");
    }
    @Override
    public void onResume() {
        super.onResume();
        Log.i(TAG, "onResume()");
    }
    @Override
    public void onPause() {
        super.onPause();
        Log.i(TAG, "onPause()");
    }
    @Override
    public void onStop() {
        super.onStop();
        Log.i(TAG, "onStop()");
    }
    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i(TAG, "onDestroy()"); 
    }
}

```

## Activity的分类
根据运行时期的作用分类，一个是MainActivity，一个是普通Activity。

MainActivity是程序的入口，它是Android应用程序的主界面，从桌面点击图标之后创建的第一个Activity，用户也可以通过它来启动其他的Activity。

普通Activity是指除了MainActivity之外的其他Activity，它们可以在应用程序中被启动、暂停、停止和销毁。用来承载其他的一些页面功能。

代码示例：

```java
public class MyActivity extends Activity {
    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```

清单文件注册展示：

```xml
    <activity
        android:name=".activity.CameraActivity"
        android:exported="false" />
    <activity
        android:name=".activity.MainActivity"
        android:exported="true" >
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <activity android:name=".activity.MusicPlayActivity" />
```

