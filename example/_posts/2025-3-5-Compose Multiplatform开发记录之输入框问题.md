---
layout: post
description: > 
  本文介绍了CMP开发过程中，TextField可组合项对Enter按键监听不符合预期的问题
image: 
  path: /assets/img/blog/blogs_cmp_debugmanager_splash.png
  srcset: 
    1920w: /assets/img/blog/blogs_cmp_debugmanager_splash.png
    960w:  /assets/img/blog/blogs_cmp_debugmanager_splash.png
    480w:  /assets/img/blog/blogs_cmp_debugmanager_splash.png
accent_image: /assets/img/blog/blogs_cmp_debugmanager_splash.png
excerpt_separator: <!--more-->
sitemap: false
---
# Compose Multiplatform开发记录之输入框问题
此前发过一篇文章介绍了我开发的Desktop端端跨平台Android设备调试软件——DebugManager。

包含了基础设备信息，应用管理，文件管理，性能监测，主题切换等。

![blogs\_dark\_deviceinfo.png](/assets/img/blog/blogs_cmp_deviceinfo.png)

## 本次记录问题点

记录为开发AI大模型对话功能页面中，对TextField输入框回车键监听问题的解决。

页面如下：

![Snipaste\_2025-03-04\_20-47-00.png](/assets/img/blog/blogs_cmp_debugmanager_ai_chat.png)

普通用户在电脑程序中对于输入框的期望，就是按Enter键可以直接确认，按Alt+Enter可以输入换行符。

## 第一版——基础输入功能

对 Compose 官方的 TextField 可组合项进行简单封装：

```kotlin
@Composable
fun WrappedEditText(
    value: String,
    onValueChange: (String) -> Unit,
    tipText: String,
    modifier: Modifier = Modifier
) {
    TextField(
        value = value,
        textStyle = infoText,
        colors = TextFieldDefaults.textFieldColors(
            textColor = MaterialTheme.colors.onPrimary,
            cursorColor = MaterialTheme.colors.onPrimary,
            focusedIndicatorColor = MaterialTheme.colors.onPrimary,
            unfocusedIndicatorColor = MaterialTheme.colors.onSecondary
        ),
        label = { Text(tipText, color = MaterialTheme.colors.onSecondary) },
        onValueChange = { onValueChange(it) },
        modifier = modifier
            .widthIn(max = 200.dp, min = 100.dp)
            .clip(RoundedCornerShape(10.dp))
            .background(MaterialTheme.colors.secondary)
            .border(2.dp, MaterialTheme.colors.onSecondary, RoundedCornerShape(10.dp)),
    )
}
```

外部调用的时候，通过维护一个mutableStringState，和这里的onValueChange配合，来进行TextField显示内容和实际字符串变量的更新。

```kotlin
@Composable
fun AiModelPage() {
    BasePage("AI大模型对话") {
        val mainStateHolder by remember { mutableStateOf(GlobalContext.get().get<MainStateHolder>()) }

        val toastState = rememberToastState()

        val userInputSting = remember { mutableStateOf("") }

        WrappedEditText(
            value = userInputSting.value,
            tipText = "输入对话文字",
            onValueChange = { userInputSting.value = it },
            modifier = Modifier.padding(start = 10.dp, end = 10.dp).weight(1f),
        )
        CommonButton(
            "发送", onClick = {
                if (userInputSting.value.isEmpty()) {
                    toastState.show("请先输入对话内容")
                } else {
                    mainStateHolder.chatWithAI(userInputSting.value)
                    userInputSting.value = ""
                }
            },
            modifier = Modifier.padding(10.dp)
        )
    }
}
```

只有点击来发送按钮后，才会将对话内容发给大模型。

## 第二版——加入Enter事件回调

为了实现按下 `Enter` 按键就可以发送消息，我在Modifier修饰符参数里加入了对Enter的 KeyEvent 监听：

```kotlin
@Composable
fun WrappedEditText(
    value: String,
    onValueChange: (String) -> Unit,
    tipText: String,
    modifier: Modifier = Modifier,
    onEnterPressed: () -> Unit = {}
) {
    val focusRequester = remember { FocusRequester() }
    TextField(
        value = value,
        textStyle = infoText,
        colors = TextFieldDefaults.colors(
            focusedTextColor = MaterialTheme.colorScheme.onPrimary,
            cursorColor = MaterialTheme.colorScheme.onPrimary,
            focusedIndicatorColor = MaterialTheme.colorScheme.onPrimary,
            unfocusedIndicatorColor = MaterialTheme.colorScheme.onPrimary
        ),
        label = { Text(tipText, color = MaterialTheme.colorScheme.onSecondary) },
        onValueChange = { onValueChange(it) },
        modifier = modifier
            .widthIn(max = 200.dp, min = 100.dp)
            .clip(RoundedCornerShape(10.dp))
            .background(MaterialTheme.colorScheme.secondary)
            .border(2.dp, MaterialTheme.colorScheme.onSecondary, RoundedCornerShape(10.dp))
            .focusRequester(focusRequester)
            .onKeyEvent {
                if (it.key == Key.Enter) {
                    onEnterPressed()
                    return onKeyEvent true
                }
                false
            },
    )
}
```

在监测到Enter键按下时，执行外部的onEnterPressed这个Lambda块，外部调用配置的时候，在这里执行和点击右侧的发送按钮一样的逻辑。

问题就是，最后的这个换行符，连同输入的内容一起被添加到了输入框的UI，还有对话气泡中去了。

![huanhang.png](/assets/img/blog/blogs_cmp_debugmanager_ai_chat_enter.png)

## 第三版——AI提供的传参数方案
查看官方文档，提供的几个api都会和上面那个按键监听策略一样的问题，换行符和内容混到了一起。

询问Gemini给出了一个方法，通过自定义 `keyboardOptions` 和 `keyboardActions` 两个参数，并在keyboardActions的onDone回调里调用onEnterPressed代码块。

```kotlin
@Composable
fun WrappedEditText(
    value: String,
    onValueChange: (String) -> Unit,
    tipText: String,
    modifier: Modifier = Modifier,
    onEnterPressed: () -> Unit = {}
) {
    val focusRequester = remember { FocusRequester() }

    TextField(
        value = value,
        textStyle = infoText,
        colors = TextFieldDefaults.colors(
            focusedTextColor = MaterialTheme.colorScheme.onPrimary,
            cursorColor = MaterialTheme.colorScheme.onPrimary,
            focusedIndicatorColor = MaterialTheme.colorScheme.onPrimary,
            unfocusedIndicatorColor = MaterialTheme.colorScheme.onPrimary
        ),
        label = { Text(tipText, color = MaterialTheme.colorScheme.onSecondary) },
        onValueChange = { onValueChange(it) },
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
        keyboardActions = KeyboardActions(
            onDone = {
                onEnterPressed()
            }
        ),
        modifier = modifier
            .widthIn(max = 200.dp, min = 100.dp)
            .clip(RoundedCornerShape(10.dp))
            .background(MaterialTheme.colorScheme.secondary)
            .border(2.dp, MaterialTheme.colorScheme.onSecondary, RoundedCornerShape(10.dp))
            .focusRequester(focusRequester),
    )
}
```

实测发现并没有成功监听到Enter键的事件。

为了搞清楚按键的顺序，恢复到第二版的方案后，通过在 `onValueChange` 和 `onKeyEvent` 里打印log看到：

* 在普通按键按下时，KeyEvent可以拦截，先手回调。
* 然而按下Enter键时，KeyEvent却在onValueChange的后面回调，即输入框的内容已经吃掉了换行符，这样就无法提前对onValueChange回调之前进行操作。

## 第四版——使用内部状态来多重判断

先上代码：

```kotlin
@Composable
fun WrappedEditText(
    value: String,
    onValueChange: (String) -> Unit,
    tipText: String,
    modifier: Modifier = Modifier,
    onEnterPressed: () -> Unit = {}
) {
    val focusRequester = remember { FocusRequester() }

    var ctrlPressed by remember { mutableStateOf(false) }

    var altPressed by remember { mutableStateOf(false) }

    TextField(
        value = value,
        textStyle = infoText,
        colors = TextFieldDefaults.colors(
            focusedTextColor = MaterialTheme.colorScheme.onPrimary,
            cursorColor = MaterialTheme.colorScheme.onPrimary,
            focusedIndicatorColor = MaterialTheme.colorScheme.onPrimary,
            unfocusedIndicatorColor = MaterialTheme.colorScheme.onPrimary
        ),
        label = { Text(tipText, color = MaterialTheme.colorScheme.onSecondary) },
        onValueChange = {
            // 如果此时使用了ctrl或者alt键，那么就不做处理
            // 否则就处理，丢弃掉最后一个换行符
            onValueChange(if (!ctrlPressed && !altPressed) it.processText() else it)
        },
        modifier = modifier
            .widthIn(max = 200.dp, min = 100.dp)
            .clip(RoundedCornerShape(10.dp))
            .background(MaterialTheme.colorScheme.secondary)
            .border(2.dp, MaterialTheme.colorScheme.onSecondary, RoundedCornerShape(10.dp))
            .focusRequester(focusRequester)
            .onKeyEvent {
                // 只有单独按下enter键才触发，其余组合键只换行
                if (it.isCtrlPressed) {
                    ctrlPressed = true
                    return onKeyEvent false
                } else {
                    ctrlPressed = false
                }
                if (it.isAltPressed) {
                    altPressed = true
                    return onKeyEvent false
                } else {
                    altPressed = false
                }
                if (it.key == Key.Enter) {
                    onEnterPressed()
                    return onKeyEvent true
                }
                false
            },
    )
}

/**
 * 用来兜底TextField的bug，暂时没有找到更好的解决方案
 * 手动丢弃掉最后一个换行符
 */
private fun String.processText(): String {
    return if (this.endsWith("\n")) {
        // 如果是单一个换行符，直接置空
        // 如果非单换行符，就丢弃最后一个字符
        if (this.length == 1) ""
        else this.dropLast(1)
    } else this
}
```

KeyEvent里面提供了几个重要按键按下的状态回调，我使用内部State来记录Ctrl和Alt这两个按键的按键状态，isPressed时置为true，没有按下时置为false，这样就可以在onValueChange时对回调过来的字符串进行加工处理。即，在Ctrl按键和Alt按键按下时，如实地回调键盘事件给输入框，这两个按键都没有按时，对字符串的最后一个字符进行检查。

处理方法如 `String.processText()`，如果以换行符结尾，再判断这个字符串是不是就只有一个换行符，这种情况就直接置为空字符串，如果有多个字符，就把最后一个换行符给去掉，再传递给外部的调用方，保证了输入框的UI和实际的字符串里都不会显示异常。

![blogs_cmp_debugmanager_chat_fixed.png](/assets/img/blog/blogs_cmp_debugmanager_chat_fixed.png)

最终实现组合按键正常换行，单独换行键直接发送对话。后续计划持续跟进，看看这里是不是跨平台库中的一个BUG，还有就是有没有官方封装完善的方案来直接使用。
