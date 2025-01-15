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
## 什么是Activity
Activity是Android四大组件之一，主要用于与用户进行交互。它提供了一种标准的方式来处理用户的输入和输出，使得不同应用可以共享数据，而不需要了解对方的内部实现。

## Activity的使用
Activity的使用分为以下几个步骤：
1. 创建Activity的子类。
2. 在AndroidManifest.xml文件中注册Activity。
3. 在其他应用中使用Activity。

## Activity的实现
Activity的实现分为以下几个步骤：
1. 创建Activity的子类。
2. 实现Activity的抽象方法。
3. 在AndroidManifest.xml文件中注册Activity。
4. 在Activity的子类中实现处理逻辑。

## Activity的生命周期
Activity的生命周期分为以下几个阶段：
1. onCreate()：Activity被创建时调用。
2. onStart()：Activity被启动时调用。
3. onResume()：Activity被恢复时调用。
4. onPause()：Activity被暂停时调用。
5. onStop()：Activity被停止时调用。
6. onDestroy()：Activity被销毁时调用。

## Activity的分类
Activity分为以下几种类型：
1. 普通Activity：普通Activity是默认的Activity类型，它可以在屏幕上显示。
2. 对话框Activity：对话框Activity是一种特殊的Activity类型，它可以在屏幕上显示一个对话框。
3. 全屏Activity：全屏Activity是一种特殊的Activity类型，它可以在屏幕上显示一个全屏的窗口。

代码示例：

```java
public class MyActivity extends Activity {
    @Override
    public void onCreate() {
        super.onCreate();
    }

    @Override
    public void onStart() {
        super.onStart();
    }  
}

```