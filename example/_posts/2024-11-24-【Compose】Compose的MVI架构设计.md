---
layout: post
description: > 
  本文抽离自内部的一次课题分享，普及Android平台新的声明式开发框架。
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
# 【Compose】Jetpack Compose的MVI架构设计
课题分享，对不熟悉 `Compose` 及其架构设计的老师，介绍一下 `Compose` 这个声明式UI框架，还有其官方推荐的MVI架构最佳实践。
## 声明式UI框架
‌Jetpack Compose‌是由Google在2019年推出的一个现代化的声明式UI工具包，旨在简化Android UI的开发过程。

其历史和发展可以追溯到2019年Google I/O大会上的公布，并在2021年7月29日正式发布1.0版本‌。

主要特点和优势：

* ‌声明式编程‌：使用声明式编程范式，代码更简洁、可读性更高‌
* Kotlin原生支持‌：完全使用Kotlin编写，与Kotlin语言特性无缝集成‌
* 简化UI开发‌：减少了样板代码，开发者可以更专注于UI逻辑‌
* 实时预览‌：支持实时预览功能，开发者可以即时查看UI效果‌
* 强大的社区支持‌：拥有丰富的文档、教程和社区资源‌

目前在较新版本的Android Studio里新建项目，默认排第一位的就是Compose的UI框架的项目。

![](/assets/img/blog/blogs_as_new_project.png)

下面是一个例子，在屏幕中央显示一个文本，并且可以直接在Android Studio的右侧预览实机画面：

```kotlin
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Box(
        modifier = modifier.fillMaxSize(1f),
        contentAlignment = Alignment.Center
    ) {
        Text(
            text = "Hello $name!",
            modifier = modifier
        )
    }
}
```

实时预览：

![](/assets/img/blog/blogs_as_compose_preview.png)

### 同View的写法差别
在View命令式UI架构中，对视图的创建，更新等都是设置一条条的命令来进行，每个View都是维护自己的一个状态，并且对外暴露get和set接口来供外接交互。

比如TextView的setText()方法，setBackground()方法，就是更改了这个TextView实例的mText文本，背景属性等。

```java
TextView tvTest = findViewById(R.id.test);

String userName = viewModel.getUserName();

tvTset.setText(userName);
```

对于Compose这种声明式的Ui架构，不会以对象的方式来提供组建，而是以可组合项的形式来使用。相对来说没有状态，其状态靠外部调用方的变量去维护。

例如显示一个可变字符串：

```kotlin
@Composable
fun ComposeDemo(){
    val textState = remember { mutableStateOf("Hello, Android!") }
    Text(
        text = textState.value,
        modifier = Modifier.padding(16.dp)
    )
}
```

Text可组合项和 `textState` 的设计可以理解为观察者模式，另外一个地方对textState进行修改，这个变化可以直接被Text可组合项接收到，并自动更新界面显示状态。

原理就是在编译期，Compose框架就可以分析出会受到这个 `textState` 变化所影响的代码块，并记录其引用，当此 `state` 变化时，会根据引用找到这些代码块并标记为 `Invalid` 。下一帧的渲染周期到来之前，触发重组，这个过程中就会执行这些标记 `Invalid` 的代码块，以达到更改视图内容的目的。

### Compose中的一般组件
View中的页面布局，外面使用的是一个个的Layout，像LinearLayout，FrameLayout等。利用ViewGroup来包裹View，在内部按照不同的Layout的特性，给子View设置不同的属性。

例如在LinearLayout中直接设置weight属性来实现分比例布局，在ConstraintLayout里，通过设置startToStart属性来进行相对约束布局设置。

在Compoe中，最常用的布局组件一般有Column，Row，Box几种，最近也增加了ConstraintLayout的Compose版本。

Column行布局，其内部的组件会沿着竖直方从上至下排列。Row则为水平方向从左至右排列。Box则是在原位置上，一层一层地叠加排列。

例如，我要显示一个简单的列表：

```kotlin
@Composable
fun ComposeDemo(){
    val textState = remember { mutableStateOf("Hello, Android!") }
    Column {
        repeat(8) {
            Text(
                text = textState.value,
                modifier = Modifier.padding(16.dp)
            )
        }
    }
}
```

Compose可以完美地使用Kotlin语音来编写，布局中可以无缝使用很多方便的api，这里就用到了repeat循环函数。我们可推算出 `Text()` 这个可组合函数，会被调用了8次，就会在屏幕上显示8个文本。

这在相对静态的View架构中是难以想象的。要显示一个列表视图，即使使用简化后的第三方库，比如像 `BaseRecyclerViewAdapterHelper` ，也至少需要创建一个 `list_item` 的xml布局，一个适配器 `Adapter` 类，有时候还需要写一个 `ViewHolder` 类。

使用Compose的列表预览效果如下：

![](/assets/img/blog/blogs_compose_repeat_text.png)

## 视图结构
### View视图结构

![](/assets/img/blog/blogs_view_window_frame.png){:width="300" height="400" loading="lazy"}

经典框架不做多余赘述。

### Compose视图结构
Composable可组合项在Android平台的实现，是利用 `ViewGroup` 来显示的，并且最终也是使用到Android的原生控件来显示内容。

通过打印堆栈可以看出，在页面布局的创建阶段，使用到了AndroidComposeView这个类。

![](/assets/img/blog/blogs_compose_window_frame.png)

ComposeView其实就是一个ViewGroup，它继承自AbstractComposeView，负责对Android平台的Activity的窗口进行适配。取而代之的是AndroidComposeView这个ViewGroup，Composable可组合项的内容就在这里面来渲染显示。

同View架构类似，Compose也是通过一个树形结构SlotTable来管理内部节点LayoutNode的。

View架构通过解析xml文件，得到页面的结构，再对内部组件进行测量布局绘制。Compose架构的第一步被替换为组合阶段，一个个的Composable可组合项，按照写好的声明式代码，添加到SlotTable中。

然后再进行测量放置，绘制。

### 固有特性测量
谈到Compose架构，这个是绕不过去的话题，固有特性测量的机制，也是为什么Compose可以采用疯狂嵌套而不会指数级影响测量时间的原因。

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <View
        android:layout_width="match_parent"
        android:layout_height="48dp" />

</LinearLayout>
```

这个内部的View并没有给定宽度，而是对齐父控件最大宽度。而父控件的宽度又是 `wrap_content` 的配置。

这时候， LinearLayout 就会先以 0 为强制宽度测量一下这个子 View，并正常地测量剩下的其他子 View，然后再用其他子 View 里最宽的那个的宽度，二次测量这个 `match_parent` 的子 View，最终得出它的尺寸，并同时把这个宽度作为自己最终的宽度。

有些场景甚至会有三次及以上的测量。如果是嵌套场景，层级每深一级，测量次数就会以指数级增长。


固有特性测量实际是固有尺寸测量，在父可组合项对内部的子可组合项正式测量之前，会先遍历一遍内部所有的组件，得出他们的固有尺寸，就是显示内容所需的最小和最大尺寸究竟是多少，在正式测量没有给定具体宽高时，就使用这个尺寸来作为最终测量的数据。不会像View架构一样，首次遍历测量完成之后还要再次测量一遍。

使用固有测量获得的参数：

```kotlin
@Composable
fun IntrinsicTest() {
    Column(
        modifier = Modifier
            .height(IntrinsicSize.Min)
            .background(Color.Red)
    ) {
        Text(text = "Hello Test!", modifier = Modifier.fillMaxSize(1f))
    }
}
```

### 事件分发机制
Jetpack Compose 和传统的 View 架构在触摸事件分发方面有一些不同。

在 View 架构中，事件分发遵循"责任链"模式，从顶层 `ViewGroup` 开始，自上而下传递。

各级使用 dispatchTouchEvent()、onInterceptTouchEvent() 和 onTouchEvent() 方法来处理和分发事件。

在 Jetpack Compose 中，使用 Modifier 修饰符来处理触摸事件，没有一个明确的分发链。在Android平台，由于是基于ViewGroup来承载，事件机制经过测试也是类似的自上而下的分发。通过 Modifier.pointerInput() 或 Modifier.clickable() 等修饰符来添加触摸事件监听。

```kotlin
@Composable
fun ComposeDemo() {
    val textState = remember { mutableStateOf("Hello, Compose!") }
    Text(
        text = textState.value,
        fontSize = 70.sp,
        modifier = Modifier
            .padding(20.dp)
            .pointerInput(Unit) {
                detectTapGestures(
                    onPress = {
                        textState.value = "Tap detected"
                    },
                    onDoubleTap = {
                        textState.value = "Double tap detected"
                    },
                    onLongPress = {
                        textState.value = "Long press detected"
                    },
                    onTap = {
                        textState.value = "Tap detected"
                    }
                )
            }
    )
}
```

双击后的变化：

![](/assets/img/blog/blogs_compose_pointerinput.png)

#### 手势判断的简化
在View架构里想要监听手势，比如要自定义一个View，重写其onTouchEvent，对MOVE事件里的滑动方向进行计算后判断，或者将touch事件传递给GestureDetector对象。

Compose里的手势也有相应的简化，举例一个对手指左滑的监听，同样在pointerInput函数中，需要使用 `detectHorizontalDragGestures` 函数，根据 dragAmount参数的正负来判断手势方向，然后再进行对应的处理。

```kotlin
@Composable
fun ComposeDemo() {
    val textState = remember { mutableStateOf("Hello, Compose!") }
    Text(
        text = textState.value,
        fontSize = 70.sp,
        modifier = Modifier
            .padding(20.dp)
            .pointerInput(Unit) {
                detectHorizontalDragGestures { _, dragAmount ->
                    if (dragAmount < 0) {
                        // 左滑逻辑
                       textState.value = "Left swipe detected"
                    }
                }
            }
    )
}
```

## 同View架构性能对比
整体来看，Compose的性能表现依然比View要差，毕竟View框架已经经过了多年的迭代和优化。主要体现在初始化时长，滑动流畅度还有动画等方面。

初始化显示较慢的原因之一，是Jetpack Compose为了实现了compose和Android版本之间的向后兼容，设计为了一个单独的库，并不包含在Android操作系统中。因此，库中的代码应在首次运行时使用即时（JIT）编译。这使得它在本质上比基于Android View的代码慢，后者是使用的提前编译（AOT）策略，并且二进制文件存储在设备上的操作系统中。

还有过度重组导致的性能问题，在Compose中，当可组合项观测的状态发生变化时，会触发其重组，进一步会使所有相关的可组合项进行重绘。如果状态发生变化的频率非常高，那么就会导致UI频繁地重绘，从而影响性能。

另外Compose的动画实现逻辑，同样基于重组机制，相较于View也更加复杂，性能上会差一些。

### 优化方案
1. 尽可能缩小重组范围，遵循Google官方的最小重组范围实践。
2. 避免过度使用状态，使用状态时，尽量使用不可变的状态。
3. 使用 `mutableStateOf` 函数来创建可观察的状态。
4. 使用 `remember` 函数来缓存可组合项的状态，重组前后可以保存状态。
5. 使用LazyColumn和LazyRow时，使用Key来标记每个项，避免扩大重组范围。

## 协程基础使用
协程是Kotlin为异步任务设计的一个解决方案。在Android平台，其内部依然是基于Handler和线程池。

### 举例
最常见的使用方式，在 `ViewModel` 或者 `Controller` 里写业务逻辑，在 `Activity` 里调用，这样就可以在IO线程执行网络请求，拿到结果后自动切换到主线程更新UI。

```kotlin
// viewModel或者controller里获取数据逻辑
// 使用suspend限制在协程里使用；withContext切换调度器，指定在IO线程执行下面的任务
suspend fun getUserName() = withContext(Dispatchers.IO) {
    debugLog("thread name: ${Thread.currentThread().name}")
    ServiceCreator.createService<UserService>()
        .getUserName("2cd1e3c5ee3cda5a")
        .execute()
        .body()
}

// Activity调用处
override fun onCreate(savedInstanceState: Bundle?){
    // 最直接的声明方法，在主线程执行下面的逻辑
    lifeCycleScope.launch {
        // 相当于get这一半是在IO线程执行
        //拿到结果后的变量赋值这一半操作由调度器自动切换到主线程来执行了
        val userName = mViewModel.getUserName()
        infoLog("userName: $userName")
        binding.tvUserName.text = userName
    }
}
```

### 基础概念
四个主要概念：
* suspend function。即挂起函数，delay() 就是协程库提供的一个用于实现非阻塞式延时的挂起函数
* CoroutineScope。即协程作用域，GlobalScope 是 CoroutineScope 的一个实现类，用于指定协程的作用范围，可用于管理多个协程的生命周期，所有协程都需要通过 CoroutineScope 来启动
* CoroutineContext。即协程上下文，包含多种类型的配置参数。`Dispatchers.IO` 就是 CoroutineContext 这个抽象概念的一种实现，用于指定协程的运行载体，即用于指定协程要运行在哪类线程上
* CoroutineBuilder。即协程构建器，协程在 CoroutineScope 的上下文中通过 launch、async 等协程构建器来进行声明并启动。launch、async 均被声明为 CoroutineScope 的扩展方法

#### 挂起函数
内部有耗时逻辑的函数，都可以标记位suspend函数，挂起函数只能在另一个suspend函数或者协程中调用。这是实现协程非阻塞特性的关键。

#### 协程作用域
协程作用域是协程的容器，用于管理协程的生命周期。

* 顶级作用域：GlobalScope-->全局范围，不会自动结束执行，无法取消。
* 协同作用域：coroutineScope -->抛出异常会取消父协程
* 主从作用域：supervisorScope -->抛出异常，往下传递，不会取消父协程

三种作用域真正常用的其实只有主从作用域，谁也不想让自己写的协程挂了导致整个app崩溃。常用的主从作用域我们也肯定接触过：

* MainScope：主线程的作用域，全局范围，可以取消。
* lifecycleScope： 生命周期范围，用于activity等有生命周期的组件，在Desroyed的时候会自动结束。
* viewModelScope：ViewModel范围，用于ViewModel中，在ViewModel被回收时会自动结束。

在设置异常处理时，可以使用 `CoroutineExceptionHandler` ，作用类似Java的 `UncaughtExceptionHandler` ，来捕获协程中未捕获的异常。可以兜住其内部子协程所抛出的异常，防止整个app崩溃。

```kotlin
val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
    // 处理异常
}

lifecycleScope.launch(exceptionHandler) {
    // 协程代码

}
```

#### 协程上下文
协程上下文是协程的配置参数，用于指定协程的运行载体。即用于指定协程要运行在哪类线程上。

主要使用以下三种：

* Dispatchers.IO：用于执行IO密集型任务，如网络请求、文件读写等。
* Dispatchers.Main：用于执行主线程任务，如UI更新、动画等。
* Dispatchers.Default：用于执行CPU密集型任务，如计算、数据处理等。

#### 协程构建器
协程构建器是协程的声明方式，用于声明并启动协程。常用以下两种

* launch：用于启动一个新的协程，返回一个 Job 对象，可以通过 Job 对象来控制协程的生命周期。
* async：用于启动一个新的协程，返回一个 Deferred 对象，可以通过 Deferred 对象来获取协程的返回值。

async方法启动举例：

```kotlin
// 耗时函数
suspend fun returnString(): String {
    delay(3000L)
    return "Hello, Compose!"
}

// 启动协程等待结果
val result = async { returnString() }.await()
println(result)
```

注意 `await()` 函数也为挂起函数，在结果返回之前，不会往下执行剩余代码。

## MVI架构
进入正题，MVI架构（Model-View-Intent），是一种用于构建用户界面的架构模式，它将应用程序的逻辑分为三个部分：Model、View和Intent。

Model：表示应用程序的数据和状态。它是应用程序的核心，负责管理应用程序的业务逻辑和数据。
View：表示应用程序的用户界面。它负责将Model中的数据呈现给用户，并接收用户的输入。
Intent：表示用户的操作或事件。它是View和Model之间的桥梁，负责将用户的操作转换为Model可以理解的格式。

![](/assets/img/blog/blogs_mvi_arch.png)

核心思想是保证唯一可信的单向数据流来更新UI，用户事件自上而下，数据自下而上。

### 举例
以网络请求一张图片为例，最简单的状态表达，可以设置一个加载态，一个成功后的展示态，一个失败提示。

首先定义状态数据传输的数据类：

```kotlin
data class ImageState(
    val loading: Boolean = false,
    val imageUrl: String? = null,
    val error: String? = null  
)
```

在数据层设置网络接口，发起网络请求，网络框架选用 `Jetbrains` 的 `Ktor` 。

```kotlin
class KtorClient {

    companion object {
        const val TAG = "KtorClient"
    }

    private val client = HttpClient(CIO) {
        install(Logging) {
            level = LogLevel.ALL
        }
        install(ContentNegotiation) {
            json(Json {
                prettyPrint = true
                isLenient = true
            })
        }
    }

    suspend fun getOneCatImage() = withContext(Dispatchers.IO) {
        client.get("https://api.thecatapi.com/v1/images/search").body<List<PicKtorItem>>()
    }
}
```

在ViewModel层，注入 `KtorClient` ，维护页面加载状态，发起网络请求，获取数据，然后更新状态。

```kotlin
class MainViewModel(private val ktorClient: KtorClient) : ViewModel() {

    private val _imageState = MutableStateFlow(ImageState())
    val imageState: StateFlow<ImageState> = _imageState.asStateFlow()

    fun loadCatPicture() {
        viewModelScope.launch {
            _imageState.value = _imageState.value.copy(loading = true) 
            try {
                val catPictures = ktorClient.getOneCatImage()
                if (catPictures.isNotEmpty()) {
                    _imageState.value = _imageState.value.copy(
                        loading = false,
                        imageUrl = catPictures[0].url
                    ) 
                } 
            } catch (e: Exception) {
                _imageState.value = _imageState.value.copy(
                    loading = false,
                    error = e.message
                ) 
            }
        } 
    }
}
```

在Activity里，对viewmodel维护的状态的消费与界面展示：

```kotlin
class ComposeTestActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            NetDataDemoTheme {
                val mainStateHolder: MainStateHolder by viewModel()
                val imageState = mainStateHolder.imageStateFlow.collectAsState()

                LaunchedEffect(Unit) {
                    mainStateHolder.loadCatPicture()
                }
                
                ImageTest(
                    loading = imageState.value.loading,
                    imageUrl = imageState.value.imageUrl,
                    error = imageState.value.error
                )
            }
        }
    }
}

@Composable
fun ImageTest(loading: Boolean, imageUrl: String?, error: String?) {
    Column {
        Text(text = "loading: $loading, imageUrl: $imageUrl, error: $error")
        AsyncImage(
            model = imageUrl,
            contentDescription = "cat pic",
            modifier = Modifier
                .fillMaxWidth(1f)
                .weight(1f)
        )
    }
}
```

运行效果：
![](/assets/img/blog/blogs_compose_toturial_cat_pic.png){:width="400" height="800" loading="lazy"}

可以看到各项数据被成功取到并展示，进一步设计界面还可以根据loading, imageUrl, error的不同状态，展示不同的界面。比如loading时加入加载态蒙层，失败后加入toast提示。

### 链路传递原理
#### 数据类data class
在Kotlin中，数据类（data class）自动为我们提供了一个copy方法。这个方法的主要作用是 **创建一个当前对象的副本** ，并且可以 **选择性地修改** 其中的某些属性。

在上面的例子中，ImageState是一个数据类，使用copy方法可以方便地更新_imageState的状态，而不需要手动创建一个新的ImageState对象并复制所有的属性。

#### MutableStateFlow
在Kotlin中，StateFlow 是一种响应式数据流，它会保存一个当前值，并且可以在这个值发生变化时通知所有的订阅者。MutableStateFlow 则是可以支持允许你通过 value 属性来修改这个当前值，从而触发更新通知。

在上面的示例中，_imageState 是一个 MutableStateFlow 类型的变量，它被初始化为 ImageState() 的一个实例。这意味着 _imageState 会持有一个 ImageState 类型的对象，并且可以在这个对象的状态发生变化时通知所有订阅者。

在Java语境中，StateFlow的作用甚至用法，都和LiveData几乎完全一致。

#### asStateFlow
上面已经维护了一个MutableStateFlow的变量，为了防止使用方更改，需要将其转换成一个只读类型的StateFlow。所以下面紧随其后定义了一个 `imageStateFlow` ，使用 `asStateFlow()` 方法将其转换成一个只读的StateFlow。

#### collectAsState
以上在ViewModel里的两个步骤，在View框架的也是可以通用的。使用collect收集Flow数据再操作View更新属性显示界面。例如：

```kotlin
mainStateHolder.imageStateFlow.collect {
    if(it.error!=null){
        Toast.makeText(thisComposeTestActivity, it.error, Toast.LENGTH_SHORT).show()
    }
}
```

在Jetpack Compose中，对这个Flow使用 `collectAsState()` ，它主要用于将一个 Flow 类型的数据流转换为一个可观察的 State 对象。

在上面获取图片url的示例中，collectAsState 函数的作用是将 MainStateHolder 类中的 imageStateFlow 这个 Flow 类型的数据流转换为一个 State 对象，这个 State 对象可以在Compose的UI中使用，并且当 imageStateFlow 中的数据发生变化时，Compose会自动重新组合UI以反映这些变化。

具体来说，collectAsState 函数做了以下几件事情：
* 订阅数据流：它会订阅 imageStateFlow 这个 Flow，开始接收其中的数据更新。
* 保存最新状态：每当 imageStateFlow 发出一个新的值时，collectAsState 会将这个新值保存到一个 State 对象中。
* 触发UI更新：由于Compose是响应式的，当 State 对象的值发生变化时，Compose会自动重新组合依赖于这个 State 对象的UI组件，从而实现UI的自动更新。

以上就是简要的关于Jetpack Compose的MVI架构的分享，实际使用中，最好配合依赖注入，模块化等方案进一步解耦，使代码架构更清晰易于维护。