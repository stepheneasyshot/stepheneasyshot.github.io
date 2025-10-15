---
layout: post
description: > 
  本文介绍了Compose中的remember关键字，用法及简要原理
image: 
  path: /assets/img/blog/blogs_compose_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_compose_cover.png
    960w:  /assets/img/blog/blogs_compose_cover.png
    480w:  /assets/img/blog/blogs_compose_cover.png
accent_image: /assets/img/blog/blogs_compose_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Compose】remember关键字
`remember` 关键字，用于在 Jetpack Compose 中保存可组合函数在重组期间的状态。它通过在组合中缓存计算结果，确保每次重组时状态保持不变。

使用举例：

```kotlin
@Composable
fun RememberTestPage() {
    val rememberCount = remember { mutableIntStateOf(0) }

    Column {
        Text(text = "Remember Count: ${rememberCount.intValue}")
        Button(onClick = { rememberCount.intValue++ }) {
            Text(text = "Increment")
        }
    }
}
```

这个例子中，使用 `remember` 关键字来记录点击次数，并在每次点击按钮时更新状态。确保Text可以显示正确的值。

不只是可以记录一个 `IntState` ，也可以用来记录其他复杂数据，像列表，对象等。

```kotlin
@Composable
fun ListExample() {
    val items = remember { mutableStateListOf("Item 1", "Item 2") }

    Column {
        items.forEach { item ->
            Text(item)
        }
        Button(onClick = { items.add("Item ${items.size + 1}") }) {
            Text("Add Item")
        }
    }
}
```

对简单数据的记录，我经常使用的是另一种委托的写法，声明为var，这样可以不用使用 `value` 访问器，直接使用变量名。

```kotlin
@Composable
fun RememberTestPage() {
    var rememberCount by remember { mutableIntStateOf(0) }

    Column {
        Text(text = "Remember Count: $rememberCount")
        Button(onClick = { rememberCount++ }) {
            Text(text = "Increment")
        }
    }
}
```

## 运行流程解析
remember是Compose运行时库里的一系列重载的顶层方法。
### 值存储在哪里
`remember` 存储的值位于 **Compose 运行时 (Compose Runtime)** 的 **组合 (Composition)** 内部。

具体来说 `remember` 计算的值会在 **初始组合** 期间存储在 Compose 运行时维护的内存结构中，这个结构就是“组合”。

当可组合函数（Composable）因为状态变化而 **重组** 时，`remember` 会返回上次存储的值，而不是重新执行其 Lambda 表达式内的计算（除非它的 `key` 参数发生了变化）。

`remember` 所存储的值的生命周期与调用它的**可组合项**的生命周期绑定。只要该可组合项仍在“组合”中，值就会被保留。一旦该可组合项从组合中移除（例如，不再显示或使用 `if` 条件被移除），这个被记住的值就会**被遗忘**。

如果需要状态在 **配置更改** （例如屏幕旋转）或 **进程被系统终止** 后仍然保留，应该使用 `rememberSaveable`。`rememberSaveable` 内部会将值序列化并存储在 Android 的 **`Bundle`** 机制中（类似于 `onSaveInstanceState()`），从而实现跨配置更改和进程终止的持久化。
### 调用链分析
对于上面那个例子，对应下面这个重载方法。

```kotlin
@Composable
inline fun <T> remember(crossinline calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)
```

`calculation` 这个 lambda 表达式，是每个重载方法的共同的参数，当 **缓存无效或缓存不存在时** ，calculation 会被执行，并且其返回值会被存储在缓存中供后续使用。

将其调用链简化如下：

```kotlin
@Composable
fun <T> remember(vararg inputs: Any?, calculation: @DisallowComposableCalls () -> T): T {
    val composer = currentComposer
    return composer.cache(false) {
        calculation()
    }
}
```

首先获取当前的 `Composer` 对象，即当前正在进行重组的实例对象。在 `Composer` 中，有一个cache方法，专门供 `remember` 调用，用于缓存计算结果。

```kotlin
@ComposeCompilerApi
inline fun <T> Composer.cache(invalid: Boolean, block: @DisallowComposableCalls () -> T): T {
    @Suppress("UNCHECKED_CAST")
    return rememberedValue().let {
        if (invalid || it === Composer.Empty) {
            val value = block()
            updateRememberedValue(value)
            value
        } else it
    } as T
}
```

获取缓存时的这个 `invalid` 参数用于控制缓存是否失效。当 `invalid` 为true时，缓存会被视为无效，并且会重新计算。

具体的，当 `remember` 传输多个参数时，这个 `invalid` 的值就是这些传来的参数变化与否，当它们发生变化时， `invalid` 就是true，缓存就会失效，并重新计算替换旧值。

例如：

```kotlin
@Composable
fun RememberTestPage(intValue: Int) {
    var rememberCount = remember(intValue) { intValue + 2 }
}
```
