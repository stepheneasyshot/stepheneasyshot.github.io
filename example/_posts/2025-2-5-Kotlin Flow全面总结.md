---
layout: post
description: > 
  本文介绍了车载Android里的常用的View控件交互总结
image: 
  path: /assets/img/blog/blogs_flow_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_flow_cover.png
    960w:  /assets/img/blog/blogs_flow_cover.png
    480w:  /assets/img/blog/blogs_flow_cover.png
accent_image: /assets/img/blog/blogs_flow_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Kotlin Flow全面总结
Kotlin的Flow项目中其实在使用，现在借助郭神在CSDN上的三篇文章进行一个相对较为全面的总结。

最常见的场景就是在viewmodel里面使用```StateFlow``` ,在Ui层进行collect。

例如一个获取Github仓库列表的网络请求数据：

```korlin
    private val githubReposSate = MutableStateFlow(GithubReposState())
    val githubReposListStateFlow = githubReposSate.asStateFlow()
```

在Composable可组合项里进行消费：
```kotlin
    val githubReposState by viewModel.githubReposListStateFlow.collectAsState()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(MaterialTheme.colors.background)
    ) {
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .background(MaterialTheme.colors.background)
        ) {
            items(githubReposState.githubReposList) {
                GithubRepoItem(it)
            }
        }
    }
```

## 冷流
使用flow构造器直接创建的为冷流，只有在调用collect函数时才会开始执行往外发送数据。

```kotlin
val TAG = "FlowOne".apply {
    Log.i(this, "init")
}

val flow = flow<Int> {
    repeat(5) {
        delay(500)
        emit(it)
    }
}

fun startCollect() {
    CoroutineScope(Dispatchers.IO).launch {
        delay(3000L)
        Log.i(TAG,"startCollect")
        flow.collect {
            Log.i(TAG,"FlowOne collect $it")
        }
    }
}
```

外部测试调用：

```kotlin
startCollect()
```

打印可以看到，flow创建好之后，没有数据打印，而是在三秒后collect时，才会从头开始进行发送：

```
16:24:21.803  I  init
16:24:24.825  I  startCollect
16:24:25.328  I  FlowOne collect 0
16:24:25.830  I  FlowOne collect 1
16:24:26.332  I  FlowOne collect 2
16:24:26.833  I  FlowOne collect 3
16:24:27.334  I  FlowOne collect 4
```

### 再次collect
如果我们在startCollect函数里再次调用collect。

```kotlin
fun startCollect() {
    CoroutineScope(Dispatchers.IO).launch {
        delay(3000L)
        Log.i(TAG, "startCollect")
        flow.collect {
            Log.i(TAG, "FlowOne collect $it")
        }

        delay(6000L)
        flow.collect {
            Log.i(TAG, "FlowOne collect twice $it")
        }
    }
}
```

第一次收集完毕，延时6s，再次调用collect。打印结果如下：

```kotlin
16:26:13.232  I  init
16:26:16.256  I  startCollect
16:26:16.758  I  FlowOne collect 0
16:26:17.261  I  FlowOne collect 1
16:26:17.763  I  FlowOne collect 2
16:26:18.264  I  FlowOne collect 3
16:26:18.766  I  FlowOne collect 4
16:26:25.269  I  FlowOne collect twice 0
16:26:25.771  I  FlowOne collect twice 1
16:26:26.272  I  FlowOne collect twice 2
16:26:26.774  I  FlowOne collect twice 3
16:26:27.276  I  FlowOne collect twice 4
```

有两个值得关注的点，第一点是第二次收集开始的时间并不是调用函数开始后6s，而是第一次收集所有数据完毕之后，延时6s。说明collect函数是一个挂起的函数，只有在数据收集完毕之后才会继续往下执行。

即连续的两个collect操作会是串行的。如果想要并行，就需要切换到不同的协程作用域。

第二个点是，第二次收集的数据也是从头开始打印的，说明冷流的每一次操作，都会从头开始。

### collectLatest
有时候，collect数据的地方，数据的消费逻辑没有走完，导致数据积压，会出现数据过时的情况，使用collectLatest可以解决这个问题。

collectLatest函数，会在每次有新数据过来时，取消上一次还未执行完的逻辑，立即处理最新的这个数据。

例如，我们在collect函数里延时3s：

```kotlin
fun startCollect() {
    CoroutineScope(Dispatchers.IO).launch {
        delay(3000L)
        Log.i(TAG, "startCollect")
        flow.collectLatest {
            Log.i(TAG, "FlowOne collect $it")
            delay(3000L)
        }
    }
}
```

打印结果：
```
16:39:09.467  I  init
16:39:12.494  I  startCollect
16:39:12.997  I  FlowOne collect 0
16:39:13.502  I  FlowOne collect 1
16:39:14.007  I  FlowOne collect 2
16:39:14.510  I  FlowOne collect 3
16:39:15.013  I  FlowOne collect 4
```

可以看到数据仍然是按照源头的时间间隔来发送的。

## 操作符
### map
可以理解为一个拦截器，将flow的原数据，经过拦截器处理之后，转换成另一个数据发送出去。map接受一个lanmbda参数，最后一行的数据，就是map的返回值。

这里以平方为例：

```kotlin
fun mapTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flowOf(1, 2, 3, 4, 5).map {
            it + 1
        }.collectLatest {
            Log.i(TAG, "mapTest collect $it")
        }
    }
}
```

打印结果：

```
16:44:00.302  I  init
16:44:00.345  I  mapTest collect 1
16:44:00.353  I  mapTest collect 4
16:44:00.356  I  mapTest collect 9
16:44:00.357  I  mapTest collect 16
16:44:00.365  I  mapTest collect 25
```

### filter
filter函数，用于过滤数据，只将满足条件的数据发送出去。filter接受一个lanmbda参数，最后一行的需要返回一个Boolean类型，用于判断是否发送数据。

```kotlin
fun filterTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flowOf(3, 6, 9, 11, 14).filter {
            it % 3 == 0
        }.collectLatest {
            Log.i(TAG, "filterTest collect $it")
        }
    }
}
```

打印可以发现，只有三的倍数通过了过滤器被接收。

### onEach
onEach函数，用于在每次数据发送之前，执行一些操作。可以打印查看原始的数据是否符合预期。

```kotlin
fun onEachTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flowOf(1, 2, 3, 4, 5).onEach {
            Log.i(TAG, "onEachTest onEach $it")
        }.map { it + 10 }.collect {
            Log.i(TAG, "onEachTest collect $it")
        }
    }
}
```

打印结果：

```
16:51:51.942  I  init
16:51:51.982  I  onEachTest onEach 1
16:51:51.984  I  onEachTest collect 11
16:51:51.984  I  onEachTest onEach 2
16:51:51.984  I  onEachTest collect 12
16:51:51.985  I  onEachTest onEach 3
16:51:51.986  I  onEachTest collect 13
16:51:51.987  I  onEachTest onEach 4
16:51:51.987  I  onEachTest collect 14
16:51:51.988  I  onEachTest onEach 5
16:51:51.993  I  onEachTest collect 15
```

### debounce
debounce函数，用于在一段时间内，只发送最后一次数据。两次数据的时间间隔太近，前一次的数据就会被丢弃，后一次数据，在延时这段时间后发送。类似Handler的remove和postDelayed的防抖操作。

```kotlin
@OptIn(FlowPreview::class)
fun debounceTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flow {
            emit(1)
            delay(100)
            emit(2)
            delay(1000)
            emit(3)
            delay(100)
            emit(4)
            delay(100)
            emit(5)
        }.debounce(500).collectLatest {
            Log.i(TAG, "debouneTest collect $it")
        }
    }
}
```

打印结果：

```
16:58:41.402  I  init
16:58:42.062  I  debouneTest collect 2
16:58:42.767  I  debouneTest collect 5
```

流程：

 > collect开始后，数据1立即发送，开始为期500ms的监测，100ms后数据2发送了，这时候数据1就被丢弃，再开始500ms监测。500ms后没有新数据来，将2发送出去。可以看到从init到第一次数据打印，就是耗时600ms。同理，2和5之间，间隔700ms。

 ### sample
 sample函数，作用类似debounce。sample有一个采样期，采样期结束，会将采样期内最后一次数据发送出去。

 ```kotlin
@OptIn(FlowPreview::class)
fun sampleTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flow {
            emit(1)
            delay(150)
            emit(2)
            delay(150)
            emit(3)
            delay(150)
            emit(4)
            delay(150)
            emit(5)
        }.sample(200).collect {
            Log.i(TAG, "debouneTest collect $it")
        }
    }
}
```

每150ms发送一次数据，采样期为200ms，所以每200ms，会将采样期内最后一次数据发送出去。打印结果：

```
17:13:34.656  I  init
17:13:34.891  I  debouneTest collect 2
17:13:35.092  I  debouneTest collect 3
17:13:35.295  I  debouneTest collect 4
```

取值的过程如下图：

![blogs_flow_sample](/assets/img/blog/blogs_flow_sample.png)

### reduce
reduce函数，用于迭代操作，在上一次计算结果的基础上，拿当前的值再进行下一步计算。

例如，将1到5的数字相乘：

```kotlin
fun reduceTest() {
    CoroutineScope(Dispatchers.IO).launch {
        val totalResult = flowOf(1, 2, 3, 4, 5).reduce { acc, value ->
            acc * value
        }
        Log.i(TAG, "reduceTest collect $totalResult")
    }
}
```

结果打印为120.

### fold
fold和reduce基本一致，只是多了一个初始值。

```kotlin
fun foldTest() {
    CoroutineScope(Dispatchers.IO).launch {
        val totalResult = flowOf(1, 2, 3, 4, 5).fold(10) { acc, value ->
            acc * value
        }
        Log.i(TAG, "foldTest collect $totalResult")
    }
}
```

打印结果为1200.

reduce和fold不仅可以用于数字，还可以用于字符串的拼接。

### flatMapConcat
以flatMap开头的操作符函数，分别是flatMapConcat、flatMapMerge和flatMapLatest。

flatMap的核心，就是将两个flow中的数据进行映射、合并、压平成一个flow，最后再进行输出。

flatMapConcat，是将两个flow中的数据进行合并，然后再进行输出。侧重点是按顺序拼接，类比C++里面两个数组的组合遍历。

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
fun flatMapConcatTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flowOf(1, 2, 3).flatMapConcat {
            flowOf("a$it", "b$it")
        }.collect {
            Log.i(TAG, "flatMapConcatTest collect $it")
        }
    }
}
```

```
17:40:40.835  I  init
17:40:40.859  I  flatMapConcatTest collect a1
17:40:40.859  I  flatMapConcatTest collect b1
17:40:40.859  I  flatMapConcatTest collect a2
17:40:40.860  I  flatMapConcatTest collect b2
17:40:40.860  I  flatMapConcatTest collect a3
17:40:40.861  I  flatMapConcatTest collect b3
```

实际应用中，例如账号登陆获取用户数据的网络请求，需要先登录获取一个token，然后再拿这个token去获取用户数据。

```kotlin
fun getUserInfo() {
    CoroutineScope(Dispatchers.IO).launch {
        sendGetTokenRequest()
            .flatMapConcat { token ->
                sendGetUserInfoRequest(token)
            }
            .flowOn(Dispatchers.IO)
            .collect { userInfo ->
                println(userInfo)
            }
    }
}
```

### flatMapMerge
flatMapMerge，同样是将两个flow中的数据进行合并，然后再进行输出。但是他的侧重点是并行的，即flow1每个数据的操作不是串行的，而是并行的。

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
fun flatMapMergeTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flowOf(300, 200, 100)
            .flatMapMerge {
                flow {
                    delay(it.toLong())
                    emit("a$it")
                    emit("b$it")
                }
            }
            .collect {
                Log.i(TAG, "flatMapMergeTest collect $it")
            }
    }
}
```

打印结果：

```
17:55:37.695  I  init
17:55:37.864  I  flatMapMergeTest collect a100
17:55:37.865  I  flatMapMergeTest collect b100
17:55:37.955  I  flatMapMergeTest collect a200
17:55:37.956  I  flatMapMergeTest collect b200
17:55:38.055  I  flatMapMergeTest collect a300
17:55:38.058  I  flatMapMergeTest collect b300
```

可以看到flatMapMerge处理之后，优先把耗时更少的数据添加到新的flow里面去进行发送。如果这里用的是flatMapConcat，那么结果就是按照300，200，100顺序发送。

### flatMapLatest
flatMapLatest，同样是将两个flow中的数据进行合并，然后再进行输出。但是他的侧重点是，只保留最新的一个数据。

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
fun flatMapLatestTest() {
    CoroutineScope(Dispatchers.IO).launch {
        flow {
            emit(1)
            delay(150)
            emit(2)
            delay(50)
            emit(3)
        }.flatMapLatest {
            flow {
                delay(100)
                emit("$it")
            }
        }.collect {
            Log.i(TAG, "flatMapLatestTest collect $it")
        }
    }
}
```

和collectLatest类似，如果使用flatMapLatest来合并多个flow，当flow1的前一个数据给到了，但是flow2没有及时合并完成，flow1的下一个数据又过来了，那么前一个数据的处理逻辑就会被掐断丢弃，直接处理最新的这个数据。

打印结果：

```
17:59:19.282  I  init
17:59:19.444  I  flatMapLatestTest collect 1
17:59:19.657  I  flatMapLatestTest collect 3
```