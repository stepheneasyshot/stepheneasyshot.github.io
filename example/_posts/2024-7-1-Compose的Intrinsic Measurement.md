---
layout: post
description: > 
  本文介绍了Jetpack Compose的固有特性测量解决嵌套卡顿的问题的原理
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
# Compose的Intrinsic Measurement
Compose 有一项规则，**子项只能测量一次，测量两次就会引发运行时异常**

但是，有时需要先收集一些关于子项的信息，然后再测量子项。

借助 Intrinsic Measurement 固有特性，您可以先 **查询子项** ，然后再进行实际测量。

对于可组合项，您可以查询其 intrinsicWidth 或 intrinsicHeight：

* (min|max)IntrinsicWidth：给定此宽度，可以正确绘制内容的最小/最大宽度是多少？
* (min|max)IntrinsicHeight：给定此高度，可以正确绘制内容的最小/最大高度是多少？

## View架构测量对比

有这么一个很常见的场景：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <View
        android:layout_width="match_parent"
        android:layout_height="48dp" />

    <View
        android:layout_width="120dp"
        android:layout_height="48dp" />

    <View
        android:layout_width="160dp"
        android:layout_height="48dp" />
</LinearLayout>
```

第一个View并没有给定宽度，是对齐父控件。而父控件的宽度又是 ```wrap_content``` 的配置。

这时候， LinearLayout 就会先以 0 为强制宽度测量一下这个子 View，并正常地测量剩下的其他子 View，然后再用其他子 View 里最宽的那个的宽度，二次测量这个 match_parent 的子 View，最终得出它的尺寸，并把这个宽度作为自己最终的宽度。

有些场景甚至会有三次及以上的测量。

更甚，如果是嵌套场景，层级每深一级，测量次数就会以指数级增长。

## Compose如何规避的

Compose在所有组合项尺寸都明确的情况下，也是不需要进行特殊处理。

在未明确指定尺寸的情况下，Compose会使用一个 固有特性测量 的机制，来规避掉父子组合项做出递归的多次测量。

所谓的 Intrinsic Measurement，指的是 Compose 允许父组件在对子组件进行测量之前， **先测量一下子组件的「固有尺寸」** ，直白地说就是「你内部内容的最大或者最小尺寸是多少」。

这是一种 **粗略的测量** ，虽说没有真正的「二次测量」模式那么自由，但功能并不弱，因为各种 Layout 里的重复测量，其实本来就是先进行这种「粗略测量」再进行最终的「正式测量」的——比如刚才说的那种「外面 wrap_content 里面 match_parent」的。

这种「粗略」的测量是很轻的，并不是因为它量得快，而是因为它在机制上不会像传统的二次测量那样，让组件的测量时间随着层级的加深而不断加倍。

当界面需要这种 Intrinsic Measurement——也就是说那个所谓的「固有特性测量」——的时候，Compose 会 **先对整个组件树进行一次 Intrinsic 测量** ，然后再对整体进行正式的测量。

**举例**

```kotlin
@Composable
fun IntrinsicTest() {
    Column(
        modifier = Modifier
            .wrapContentSize()
            .background(Color.Red)
    ) {
        Text(text = "Hello Test!", modifier = Modifier.fillMaxSize(1f))
    }
}
```

![demo0](/assets/img/blog/blogs_compose_intrin_demo0_1.png){:width="300" height="700" loading="lazy"}

这里和上面的View的例子是一样的，父组合项的size是wrap的，子组合项的size是对齐上一级的。

这时候运行这个Demo。我们可以看到，整个Column的大小是占满了整个屏幕的，和View架构的表现正好相反。

因为父组合项没有划定尺寸限制，那子组合项就会无限扩张自己的领地，最终对他的测量数据就是占满屏幕的宽高。

**使用固有尺寸测量参数**

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

结果：


![demo0](/assets/img/blog/blogs_compose_intrin_demo0_2.png){:width="300" height="700" loading="lazy"}

我将外部Column的高度参数设置为 ```IntrinsicSize.Min``` 就可以达到要求。

height(IntrinsicSize.Min) 可将其子项的高度强行调整为最小固有高度。由于该修饰符具有递归性，因此它将查询 Column 及其子项 minIntrinsicHeight。 而Text 元素的 minIntrinsicHeight 为 文本的固有宽高。

因此 Column 元素的 height 约束条件将和Text的最小占用的宽高一致。而Text设置fillMaxSize之后获取的高度，就会变成Text占用的最小高度了。

如果将 Min 改成 Max 呢？

那效果也是一致的，如果您查询具有无限 height 的 Text 的 minIntrinsicHeight，它将返回 Text 的 height，就好像该文本是在单行中绘制的一样。

## 实际使用场景
### 举例1 分割线自适应高度

要实现下面这个效果，两个文字中间画一条分割线：

![blogs_compose_intrinc_demo1](/assets/img/blog/blogs_compose_intrinc_demo1.png){:width="300" height="50" loading="lazy"}

我们该怎么做？我们可以将两个 Text 放在同一 Row，并在其中最大程度地扩展，另外在中间放置一个 Divider。我们需要将 Divider 的高度设置为与最高的 Text 相同，粗细设置为 width = 1.dp。


```kotlin
@Composable
fun TwoTexts(modifier: Modifier = Modifier, text1: String, text2: String) {
    Row(modifier = modifier) {
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(start = 4.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1
        )
        HorizontalDivider(
            color = Color.Black,
            modifier = Modifier.fillMaxHeight().width(1.dp)
        )
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(end = 4.dp)
                .wrapContentWidth(Alignment.End),

            text = text2
        )
    }
}
```

预览时，我们发现 Divider 会扩展到整个屏幕，这并不是我们想要的效果：

![blogs_compose_two_text_max](/assets/img/blog/blogs_compose_two_text_max.png){:width="300" height="100" loading="lazy"}

两个文本元素并排显示，中间用分隔线隔开，但分隔线向下延伸到文本底部下方

之所以出现这种情况，是因为 Row 会逐个测量每个子项，并且 Text 的高度不能用于限制 Divider。我们希望 Divider 以一个给定的高度来填充可用空间。为此，我们可以使用 height(IntrinsicSize.Min) 修饰符。

height(IntrinsicSize.Min) 可将其子项的高度强行调整为最小固有高度。由于该修饰符具有递归性，因此它将查询 Row 及其子项 minIntrinsicHeight。

将其应用到代码中，就能达到预期的效果：

```kotlin
@Composable
fun TwoTexts(modifier: Modifier = Modifier, text1: String, text2: String) {
    Row(modifier = modifier.height(IntrinsicSize.Min)) {
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(start = 4.dp)
                .wrapContentWidth(Alignment.Start),
            text = text1
        )
        HorizontalDivider(
            color = Color.Black,
            modifier = Modifier.fillMaxHeight().width(1.dp)
        )
        Text(
            modifier = Modifier
                .weight(1f)
                .padding(end = 4.dp)
                .wrapContentWidth(Alignment.End),

            text = text2
        )
    }
}

// @Preview
@Composable
fun TwoTextsPreview() {
    MaterialTheme {
        Surface {
            TwoTexts(text1 = "Hi", text2 = "there")
        }
    }
}

```

这时候的结果就是我们需要的了。

### 举例2 兄弟组合项对齐数据
需求是在屏幕上显示左右两个栏目，两边的内容不一定一样多，但是背景色块需要一样高。

![blogs_compose_intrinc_demo2](/assets/img/blog/blogs_compose_intrin_demo2_0.jpg)

我们使用row来分栏，然后在每个column里填数据，不主动设置高度。

```kotlin
@Composable
fun IntrinsicTest() {
    val shortList = remember { shortList }
    val longList = remember { longList }
    Row {
        Column(
            modifier = Modifier
                .weight(0.5f)
                .background(Color.Red)
        ) {
            shortList.forEach { Text(text = it) }
        }
        Column(
            modifier = Modifier
                .weight(0.5f)
                .background(Color.Blue)
        ) {
            longList.forEach { Text(text = it) }
        }
    }
}
```

结果：

![blogs_compose_intrinc_demo21](/assets/img/blog/blogs_compose_intrin_demo2_1.png){:width="300" height="700" loading="lazy"}

我们发现两个Column的高度是不一致的。

如果我为了使两侧高度显示一致，直接将两边的高度值写死，那么在不同屏幕上的自适应又会出问题。

这时候我们使用 `IntrinsicSize.Max` 来解决这个问题。设置为max，父组合项的高度会取子项中最大的高度。然后让两个子项的高度直接 `fillMaxHeight` 。

```kotlin
@Composable
fun IntrinsicTest() {
    val shortList = remember { shortList }
    val longList = remember { longList }
    Row(modifier = Modifier.height(IntrinsicSize.Max)) {
        Column(
            modifier = Modifier
                .weight(0.5f)
                .fillMaxHeight(1f)
                .background(Color.Red)
        ) {
            shortList.forEach { Text(text = it) }
        }
        Column(
            modifier = Modifier
                .weight(0.5f)
                .fillMaxHeight(1f)

                .background(Color.Blue)
        ) {
            longList.forEach { Text(text = it) }
        }
    }
}
```

结果：

![blogs_compose_intrinc_demo22](/assets/img/blog/blogs_compose_intrin_demo2_2.png){:width="300" height="700" loading="lazy"}

可以看到两个column的高度是一样的了。
