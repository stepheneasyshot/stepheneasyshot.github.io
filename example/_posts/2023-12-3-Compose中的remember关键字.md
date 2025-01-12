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
# Compose中的remember关键字

## 使用

```remember``` 关键字，用于在 Jetpack Compose 中保存可组合函数在重组期间的状态。它通过在组合中缓存计算结果，确保每次重组时状态保持不变。

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

这个例子中，使用remember关键字来记录点击次数，并在每次点击按钮时更新状态。确保Text可以显示正确的值。

不只是可以记录一个IntState，也可以用来记录其他复杂数据，像列表，对象等。

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

对简单数据的记录，我经常使用的是另一种委托的写法，声明为var，这样可以不用使用value访问器，直接使用变量名。

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

remember是Compose运行时库里的一系列重载的顶层方法，

对于上面那个例子，对应下面这个重载方法。

```
@Composable
inline fun <T> remember(crossinline calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)

```

```calculation``` 这个 lambda 表达式，是每个重载方法的共同的参数，当缓存无效或缓存不存在时，calculation 会被执行，并且其返回值会被存储在缓存中供后续使用。

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

首先获取当前的Composer对象，即当前正在进行重组的实例对象。在Composer中，有一个cache方法，专门供remember调用，用于缓存计算结果。

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

获取缓存时的这个 ```invalid``` 参数用于控制缓存是否失效。当invalid为true时，缓存会被视为无效，并且会重新计算。

具体的，当remember传输多个参数时，这个invalid的值就是这些传来的参数变化与否，当它们发生变化时，invalid就是true，缓存就会失效，并重新计算替换旧值。

例如：

```kotlin
@Composable
fun RememberTestPage(intValue: Int) {
    var rememberCount = remember(intValue) { intValue + 2 }
}
```
