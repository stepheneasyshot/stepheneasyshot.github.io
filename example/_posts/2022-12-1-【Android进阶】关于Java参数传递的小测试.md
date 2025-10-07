---
layout: post
description: > 
  本文介绍了Java方法参数传递过程中的一些规则，理清了一些流程
image: 
  path: /assets/img/blog/blogs_java_common_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_java_common_cover.png
    960w:  /assets/img/blog/blogs_java_common_cover.png
    480w:  /assets/img/blog/blogs_java_common_cover.png
accent_image: /assets/img/blog/blogs_java_common_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】关于Java参数传递的小测试
很早之前就了解到，Java方法传递参数是值传递，不是引用传递。

在 `C++` 中，我们可以把方法的参数设置为外部变量的引用，就可以直接通过这个引用操作外部变量。例如C++的函数参数，如果在合适的时机，按引用传递，可以省去变量复制的步骤，优化性能。

```cpp
#include <iostream>
using namespace std;

void GetSquare(int& number)
{
   number *= number;
}

int main()
{
   cout << "Enter a number you wish to square: ";
   int number = 0;
   cin >> number;

   GetSquare(number);
   cout << "Square is: " << number << endl;

   return 0;
}
```

Java是没有这样的机制的。
## 一、Java 是值传递，不是引用传递
首先明确一点：

> **Java 中所有的参数传递都是值传递（pass by value），没有引用传递（pass by reference）。**

这句话还有两个扩展结论：
- 当你传递一个基本类型（如 `int`, `boolean` 等）给方法时，传递的是它的值的副本。
- 当你传递一个对象（如 `String`, `Activity` 等）给方法时，传递的是该对象的**引用的副本**，而不是对象本身的副本。

举个例子：

```java
void modifyObject(MyObject obj) {
    obj.setValue(100);  // 修改的是原对象的内容
    obj = new MyObject(); // 修改的是局部变量 obj 的引用，不影响外部
}

MyObject myObj = new MyObject();
modifyObject(myObj);
// myObj 指向的对象被修改了，但 myObj 本身还是原来的引用
```

在这个例子中：

- `obj` 是 `myObj` **引用的一个副本**，它们指向同一个对象。
- 所以通过 `obj.setValue(100)` 可以修改原对象的内容。
- 但是 `obj = new MyObject()` 只是让局部变量 `obj` 指向了一个新对象，**不会影响外部的 `myObj`**。


## 二、 使用Activity 作为参数
如果是按照值的副本传递，那么Activity对象被当作参数，传递给外部方法并引用，在Activity销毁时是不会产生泄露现象的，正式因为传递的是引用的副本，所以这个引用关系仍然存在。

> 在 Android 中，当把一个 `Activity` 作为参数传递给某个方法时，似乎没有复制一个新的 `Activity` 对象，而是直接操作了原来的 `Activity`。

这 **正是 Java 值传递的表现**。

### 具体解释：

假设你有如下代码：

```java
startSomeProcess(MainActivity.this);
```

这里的 `MainActivity.this` 是当前 `Activity` 的引用（即指向 `Activity` 对象的一个指针）。当你把这个引用作为参数传递给方法时：

```java
void startSomeProcess(Activity activity) {
    activity.setTitle("New Title"); // 修改的是原 Activity 的标题
    activity = new Activity();     // 这里只是修改了局部变量 activity 的指向
}
```

- `activity` 是 `MainActivity.this` 引用的一个副本（即引用的值被复制了一份）。
- 所以 `activity.setTitle("New Title")` 修改的是原来的 `Activity` 对象。
- 但是 `activity = new Activity()` 只是让方法内部的局部变量 `activity` 指向了一个新的 `Activity` 对象，**不会影响外部的 `MainActivity.this`**。

这完全符合 Java 的值传递机制。但是下面几点需要注意：

### 1. `Activity` 是一个重量级对象，通常不应该作为方法参数频繁传递

- `Activity` 本身包含大量状态信息、视图层次结构、生命周期管理等。
- 将 `Activity` 作为参数传递，尤其是跨组件传递（如从 Fragment 传递到工具类、Service 等），是一种不好的实践，可能导致内存泄漏或逻辑混乱。

### 2. `Activity` 持有 `Context`，而 `Context` 是与生命周期强相关的

- 如果你在一个**长生命周期对象**（如单例、静态变量、Service 等）中持有 `Activity` 的引用，可能会导致 `Activity` 无法被回收，从而引发内存泄漏。
- 这也是为什么 Android Lint 会对 `static` 字段持有 `Activity` 发出警告（`StaticFieldLeak`）。

---

### 正确做法建议
尽量避免直接传递 `Activity` 对象，而是通过接口、回调或者 `Context`（非 `Activity` 类型）来解耦。

使用 Application Context 替代 Activity Context，比如加载资源、启动 Service 等操作可以使用 `getApplicationContext()`，避免持有 `Activity`。

必要时，使用弱引用（WeakReference）来持有 Activity。如果确实需要在某个长生命周期对象中引用 `Activity`，可以使用 `WeakReference<Activity>`，这样即使 `Activity` 被销毁，也不会阻止垃圾回收。

示例：

```java
private WeakReference<Activity> activityRef;

public void setActivity(Activity activity) {
    this.activityRef = new WeakReference<>(activity);
}

public void doSomething() {
    Activity activity = activityRef.get();
    if (activity != null && !activity.isFinishing()) {
        activity.setTitle("Safe Title Change");
    }
}
```
