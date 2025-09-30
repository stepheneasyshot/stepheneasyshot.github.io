---
layout: post
description: > 
  本文介绍了Fragment组件的进阶知识，包括Fragment的生命周期、Fragment间的通信方式等。
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
# 【Android基础】Fragment组件进阶
`Fragment` 是 Android UI 开发中一个非常重要的组件，用于构建模块化、可复用且灵活的用户界面。

**Fragment** 可以被视为一个**Activity 的一部分或行为**。它拥有自己的生命周期、布局和输入事件，但它必须托管在一个 `Activity` 中。一个 `Activity` 可以包含一个或多个 `Fragment`，也可以在不同的 `Activity` 中复用同一个 `Fragment`。
### Fragment 的主要作用
1.  **模块化 UI：** 可以将一个复杂的用户界面分解成独立的、可管理的模块。
2.  **UI 可复用性：** 可以在不同的 Activity 或同一 Activity 的不同配置（如横竖屏）中复用 Fragment。
3.  **适应不同屏幕尺寸：** 尤其在平板电脑等大屏幕设备上，可以同时显示多个 Fragment，例如列表-详情布局（List-Detail Flow）。
4.  **简化 Activity 代码：** 将 UI 逻辑和行为从 Activity 中分离出来，使 Activity 变得更轻量和专注于协调。
5.  **支持回退栈：** 可以像 Activity 一样管理 Fragment 的回退栈，实现前进和后退导航。

### 生命周期
Fragment 的生命周期与它所依附的 Activity 的生命周期**紧密相关**。理解这些回调方法对于正确管理 Fragment 的状态至关重要。

以下是 Fragment 生命周期中几个关键的方法及其大致顺序：

* **`onAttach()`**: 当 Fragment 与 Activity 关联时调用。此时可以获取到 `Context` 对象。
* **`onCreate()`**: Fragment 被创建时调用。在这里进行非 UI 的初始化，如变量设置、数据加载等。
* **`onCreateView()`**: 创建 Fragment 的用户界面（View）。在这里膨胀（inflate）布局并返回根视图。
* **`onViewCreated()`**: `onCreateView()` 返回后调用。在这里可以初始化 View 组件，设置监听器等。
* **`onActivityCreated()`**: 当宿主 Activity 的 `onCreate()` 方法完成时调用。可以在这里执行依赖于 Activity 已创建的代码。
* **`onStart()`**: Fragment 可见时调用。
* **`onResume()`**: Fragment 获得焦点并可与用户交互时调用。
* **`onPause()`**: Fragment 失去焦点，但仍然部分可见时调用（例如，另一个 Fragment 覆盖了它）。
* **`onStop()`**: Fragment 不再可见时调用。
* **`onDestroyView()`**: Fragment 的视图被移除时调用。在这里释放与 View 相关的资源。
* **`onDestroy()`**: Fragment 实例被销毁时调用。在这里释放所有非 View 相关的资源。
* **`onDetach()`**: Fragment 与 Activity 解除关联时调用。

### 使用流程
#### 1\. 创建 Fragment
创建一个继承自 `androidx.fragment.app.Fragment` 的 Java/Kotlin 类，并通常重写 `onCreateView()` 方法来提供其布局：

```kotlin
class MyFragment : Fragment() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Fragment 初始化逻辑
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        // 膨胀 Fragment 的布局
        return inflater.inflate(R.layout.fragment_my, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // 初始化 View 组件
        // val myTextView = view.findViewById<TextView>(R.id.myTextView)
        // myTextView.text = "Hello from Fragment!"
    }
}
```

对应的 `fragment_my.xml` 布局文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/myTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="My Fragment Content"
        android:textSize="24sp"
        android:layout_gravity="center" />

</FrameLayout>
```

#### 2\. 将 Fragment 添加到 Activity
有两种主要方式将 Fragment 添加到 Activity 中：
##### a. 在布局 XML 中声明

你可以在 Activity 的布局 XML 文件中直接声明一个 Fragment。这是 **静态添加** Fragment 的方式。

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <fragment
        android:id="@+id/my_static_fragment"
        android:name="com.example.yourapp.MyFragment" // 完整的 Fragment 类名
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```

这种方式下，Fragment 的生命周期会与 Activity 的生命周期紧密耦合，并且在 Activity 创建时就被实例化。
##### b. 运行时动态添加（推荐）
通过 `FragmentManager` 和 `FragmentTransaction` 在 Activity 运行时动态添加、移除、替换或显示/隐藏 Fragment。这是最常用的方式，因为它提供了更大的灵活性。

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 检查 Fragment 是否已经添加，避免重复添加（例如在配置变化后）
        if (savedInstanceState == null) {
            val fragmentManager: FragmentManager = supportFragmentManager
            val fragmentTransaction: FragmentTransaction = fragmentManager.beginTransaction()

            val myFragment = MyFragment()
            // 将 Fragment 添加到一个容器视图中 (例如一个 FrameLayout)
            fragmentTransaction.add(R.id.fragment_container, myFragment)
            // fragmentTransaction.addToBackStack(null) // 可选：添加到回退栈
            fragmentTransaction.commit()
        }
    }
}
```

> 初始化添加时最好是先行检查一下 `savedInstanceState` 是否为 `null`，避免重复添加 Fragment。如果是系统的配置变更，如语言和主题，我们知道Activity会自动重建，而FagmentManager 会在 Activity 重建时，自动恢复那些在 Activity 被销毁前已经存在的 Fragment 实例。如果此时又调用了 `fragmentTransaction.add()` 方法添加 Fragment，就会导致重复添加，引发异常，界面内容可能会重叠显示多个fragment。

对应的 `activity_main.xml` 布局需要一个用于容纳 Fragment 的容器：

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

### Fragment 的通信
由于 Fragment 之间是独立的，它们之间以及与宿主 Activity 之间需要明确的通信机制。

1.  **Fragment 到 Activity：**

      * **推荐方式：** 定义一个接口，让 Activity 实现该接口。Fragment 通过 `onAttach()` 获取 Activity 实例并将其转换为接口类型，然后调用接口方法。

    <!-- end list -->

    ```kotlin
    // Fragment
    class MyFragment : Fragment() {
        interface OnMessageListener {
            fun onMessageFromFragment(message: String)
        }

        private var listener: OnMessageListener? = null

        override fun onAttach(context: Context) {
            super.onAttach(context)
            if (context is OnMessageListener) {
                listener = context
            } else {
                throw RuntimeException("$context must implement OnMessageListener")
            }
        }

        // ... 某个事件触发时
        fun sendMessage() {
            listener?.onMessageFromFragment("Hello from Fragment!")
        }

        override fun onDetach() {
            super.onDetach()
            listener = null
        }
    }

    // Activity
    class MainActivity : AppCompatActivity(), MyFragment.OnMessageListener {
        override fun onMessageFromFragment(message: String) {
            Log.d("MainActivity", "Received message: $message")
        }
        // ...
    }
    ```

    * **ViewModel (推荐，尤其是 Fragment 间通信)：** 使用共享的 `ViewModel` 可以非常方便地在 Fragment 和 Activity 之间共享数据和通信，尤其是在导航组件的场景下。

2.  **Activity 到 Fragment：**

    * **调用 Fragment 公开方法：** Activity 可以获取 Fragment 实例并直接调用其公共方法。

    <!-- end list -->

    ```kotlin
    // Activity
    fun sendMessageToFragment(message: String) {
        val myFragment = supportFragmentManager.findFragmentById(R.id.fragment_container) as? MyFragment
        myFragment?.updateText(message)
    }

    // Fragment
    class MyFragment : Fragment() {
        fun updateText(message: String) {
            // 更新 TextView
        }
    }
    ```

    * **通过 Bundle 传递参数：** 在创建 Fragment 实例时，通过 `setArguments(Bundle)` 方法传递参数。

    <!-- end list -->

    ```kotlin
    // Activity
    val args = Bundle().apply {
        putString("key_message", "Data from Activity")
    }
    val myFragment = MyFragment().apply {
        arguments = args
    }

    // Fragment
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val message = arguments?.getString("key_message")
        // 使用 message
    }
    ```

3.  **Fragment 到 Fragment：**

    * **通过宿主 Activity 中转（旧版，不推荐）：** 一个 Fragment 通知 Activity，Activity 再通知另一个 Fragment。
    * **共享 ViewModel (推荐)：** 多个 Fragment 可以观察同一个 `ViewModel` 中的 `LiveData`，实现数据共享和通信。
    * **Parent-to-Child FragmentManager (如果存在嵌套 Fragment)：** 可以通过 `getParentFragmentManager()` 或 `getChildFragmentManager()` 获取对应的 `FragmentManager`。
    * **Navigation Component (推荐)：** 使用 Android Navigation 组件是处理 Fragment 之间导航和参数传递的现代化且强大的方式。

### FragmentTransaction 和回退栈
当使用 `FragmentManager` 动态管理 Fragment 时，`FragmentTransaction` 是执行操作（如添加、移除、替换）的批处理API。

* **`add(containerId, fragment)`:** 将一个 Fragment 添加到容器中。
* **`remove(fragment)`:** 移除一个 Fragment。
* **`replace(containerId, fragment)`:** 移除容器中现有 Fragment，然后添加新的 Fragment。
* **`hide(fragment)` / `show(fragment)`:** 隐藏或显示一个 Fragment，但不会销毁其 View。
* **`addToBackStack(name)`:** 将当前 `FragmentTransaction` 添加到 Activity 的回退栈中。当用户按返回键时，会依次弹出栈中的 Fragment 事务，回退到之前的 Fragment 状态。
* **`commit()`:** 提交事务。这是异步操作。
* **`commitNow()`:** 提交事务。这是同步操作，但可能会阻塞 UI 线程，除非确定操作很快。通常不推荐。
* **`commitAllowingStateLoss()`:** 提交事务，即使 Activity 状态已保存，允许状态丢失。一般不推荐，除非你清楚这样做的后果。

### 最佳实践与注意事项
1.  **避免 Fragment 嵌套过多：** 复杂的 Fragment 嵌套会导致生命周期管理变得困难，并可能引入性能问题。
2.  **使用 `setArguments()` 传递参数：** 避免在 Fragment 构造函数中传递参数，因为系统可能会在屏幕旋转等情况下重新创建 Fragment 而不调用自定义构造函数。
3.  **Fragment 应该尽可能独立和可复用：** 它们不应该直接依赖于特定的 Activity 类型，而是通过接口或 ViewModel 进行通信。
4.  **处理配置变更：** Fragment 在 Activity 重建时也会重建。确保在 `onSaveInstanceState()` 和 `onCreate()`/`onCreateView()` 中正确保存和恢复 Fragment 的状态。
5.  **内存泄漏：** 注意在 `onDestroyView()` 或 `onDestroy()` 中释放不再需要的引用（尤其是对 View 的引用），以避免内存泄漏。例如，清理在 `onCreateView()` 中创建的监听器。
6.  **`getChildFragmentManager()` vs `getFragmentManager()` / `getParentFragmentManager()`：**
      * `getParentFragmentManager()` (原 `getFragmentManager()`)：用于获取管理当前 Fragment 的 `FragmentManager`。
      * `getChildFragmentManager()`：用于获取管理当前 Fragment 内部嵌套 Fragment 的 `FragmentManager`。
      * 在使用 `FragmentContainerView` 或 `supportFragmentManager.beginTransaction()` 动态添加 Fragment 时，请确保使用正确的 `FragmentManager`。
7.  **Navigation Component：** 对于复杂的导航和 Fragment 间的通信，强烈推荐使用 Android Jetpack 的 Navigation Component。它简化了 Fragment 的管理、深层链接和安全参数传递。
8.  **View Binding 或 Data Binding：** 在 Fragment 中使用 View Binding 或 Data Binding 可以更安全、高效地访问 View 组件，避免 `findViewById` 带来的空指针异常。

### 常见用例
* **标签页（Tabbed Layouts）：** 每个标签页内容可以是一个 Fragment。
* **滑动视图（Swipe Views / ViewPager2）：** `ViewPager2` 经常与 `FragmentStateAdapter` 结合使用，每个页面都是一个 Fragment。
* **大屏幕设备布局：** 例如，在平板上，一个 Fragment 显示列表，另一个 Fragment 显示详情。
* **底部导航栏（Bottom Navigation）：** 每个导航项对应一个 Fragment。
* **向导流（Wizard Flows）：** 多个 Fragment 按顺序引导用户完成任务。