---
layout: post
description: > 
  本文介绍了Compose的重组流程，主要是最小重组范围的界定和优化
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
# Kotlin的inline&crossinline&noinline关键字
## JVM常量编译时优化
Kotlin中，使用了 ```const val``` 关键字修饰的变量，在编译时会被视为常量，并且在编译时进行了优化。直接将其值复制到调用处，而不是像普通变量一样在运行时进行变量访问。这可以提高代码的执行效率，因为避免了变量调用的开销。

```kotlin
const val CONST_VAL = 10

fun main() {
    println(CONST_VAL)
}

// 编译后
fun main() {
    println(10)
}
```

## 内联函数
编译时同样被提前处理的还有内联函数，即使用了 ```inline``` 关键字修饰的函数。

JVM在编译时，会将inline函数内的代码直接复制到调用处，而不是像普通函数一样在运行时进行函数调用。听起来可能会对性能有优化，实际上少一层函数调用栈的优化是非常微小的。

而同时， **函数内联** 不同于 **常量内联** 的地方在于，函数体通常比常量复杂多了，而函数内联会导致函数体被拷贝到每个调用处，如果函数体比较大而被调用处又比较多，就会导致编译出的字节码变大很多。

### lambda参数实现方式
在Kotlin中，lambda参数的实现方式是使用了 **匿名内部类** ，而不是使用了 **函数指针** 。

在编译之后，可以看到lambda参数调用的地方，实际上是Kotlin帮我们生成了一个匿名内部类，然后在调用处调用这个匿名内部类的方法。

```kotlin
class LambdaTest {
    fun testInline(lambdaParams:()->Unit) {
        lambdaParams()
    }
}
```

经过反编译成Java代码之后：

```java
public final class LambdaTest {
   @NotNull
   public final LambdaTest testInline(@NotNull Function0 lambdaParams) {
      Intrinsics.checkNotNullParameter(lambdaParams, "lambdaParams");
      lambdaParams.invoke();
      return this;
   }
}
```
可以看到，lambdaParams的类型是 ```Function0``` ，这是一个接口。在运行过程中，就会生成一个匿名内部类，然后在调用处调用这个匿名内部类的方法。

### inline对lambda的优化
如果上述的testinline方法，在外部被高频循环调用。 

```kotlin
fun main() {
    val lambdaTest = LambdaTest()
    for (i in 0..100000) {
        lambdaTest.testInline {
            println("hello world")
        }
    }
}
```

内存占用会蹭的一下涨上来。

如果使用了这个接收lambda参数的方法使用了 ```inline``` 关键字修饰，就不会生成匿名内部类，而是直接将lambda的代码块里面的代码复制到调用处。

inline 关键字不止可以内联自己的内部代码，还可以内联自己内部的内部的代码，意思是什么呢，就是你的函数在被加了 inline 关键字之后，编译器在编译时不仅会把函数内联过来，而且会把它内部的函数类型的参数——那就是那些 Lambda 表达式——也内联过来。换句话说，这个函数被编译器贴过来的时候是完全展开铺平的:

kotlin源代码：

```kotlin
class LambdaTest {
    inline fun testInline(lambdaParams:()->Unit) {
        lambdaParams()
    }
}

fun main() {
    val lambdaTest = LambdaTest()
    for (i in 0..100000) {
        lambdaTest.testInline {
            println("hello world")
        }
    }
}

```

反编译之后：
```java
public final class LambdaTest {
   public final void testInline(@NotNull Function0 lambdaParams) {
      Intrinsics.checkNotNullParameter(lambdaParams, "lambdaParams");
      lambdaParams.invoke();
   }
}

public final class MainKt {
   public static final void main() {
      LambdaTest lambdaTest = new LambdaTest();
      int $i$iv = 0;
      int var3;
      for(var3 = 100000; $i$iv <= var3; ++$i$iv) {
         System.out.println("hello world");
      }
   }
}
```

高阶函数（Higher-order Functions）有它们天然的性能缺陷，我们通过 inline 关键字让函数用内联的方式进行编译，来减少参数对象的创建，从而避免出现性能问题。

### inline另类用法
在kotlin的 UMath.kt 工具类中，有一个max方法：

```kotlin
@SinceKotlin("1.5")
@WasExperimental(ExperimentalUnsignedTypes::class)
@kotlin.internal.InlineOnly
public inline fun max(a: UInt, b: UInt): UInt {
    return maxOf(a, b)
}
```

这个maxOf方法，来自于另一个工具类 ```UComparisonsKt``` ：

```kotlin
@SinceKotlin("1.5")
@WasExperimental(ExperimentalUnsignedTypes::class)
public fun maxOf(a: UInt, b: UInt): UInt {
    return if (a >= b) a else b
}
```

这里就通过内联的方式，将maxOf方法的代码块内联到了调用处。

可以直接通过方便的顶层函数的方式，来使用工具类，不需要创建实例或者带外部类名。

## noinline
inline 是内联，而 noinline 就是不内联。不过它不是作用于函数的，而是作用于函数的参数：对于一个标记了 inline 的内联函数，你可以对它的任何一个或多个函数类型的参数添加 noinline 关键字。添加了之后，这个参数就不会参与内联。

函数类型的参数，它本质上是个对象。我们可以把这个对象当做函数来调用，这也是最常见的用法。但同时我们也可以把它当做对象来用。比如把它当做返回值：

```kotlin
inline fun testInline(lambdaParams:()->Unit) {
    lambdaParams()
    return lambdaParams
}
```

但当我们把函数进行内联的时候，它内部的这些参数就不再是对象了，因为他们会被编译器拿到调用处去展开。

当一个函数被内联之后，它内部的那些函数类型的参数就不再是对象了，因为它们的壳被脱掉了。换句话说，对于编译之后的字节码来说，这个对象根本就不存在。一个不存在的对象，你怎么使用？

所以当你要把一个这样的参数当做对象使用的时候，Android Studio 会报错，告诉你这没法编译

noinline 就是用来局部地、指向性地关掉函数的内联优化的。既然是优化，为什么要关掉？因为这种优化会导致函数中的函数类型的参数无法被当做对象使用，也就是说，这种优化会对 Kotlin 的功能做出一定程度的收窄。而当你需要这个功能的时候，就要手动关闭优化了。这也是 inline 默认是关闭、需要手动开启的另一个原因：它会收窄 Kotlin 的功能。

## crossinline
看这样一个情景：

一个内联函数，它的参数是一个函数类型的参数。

```kotlin
inline fun lambdaReturnTest(insertAction: () -> Unit) {
    insertAction()
}
```

调用处加一个return：

```kotlin
override fun onCreate() {
    super.onCreate()

    Log.i("sdvgsrhbTAG", "before erftgyujhf")
    lambdaReturnTest {
        println("Hello World")
        return
    }
    Log.i("sdvgsrhbTAG", "after erftgyujhf")
}
```

这时候结束的不是这个lambdaReturnTest方法，而是onCreate方法。因为lambdaReturnTest方法被内联了，会直接铺平展开到调用处，连带里面的return。

这样的话，我们每次在lambda里面使用return还需要确认这个函数是否是内联函数，才可以确认这个return结束的是哪一个函数。为此Kotlin规定 **不允许在lambda参数中使用return，除非这个使用lambda参数的函数是内联函数**。


那这样的话规则就简单了：
* Lambda 里的 return，结束的不是直接的外层函数，而是外层再外层的函数；
* 但只有内联函数的 Lambda 参数可以使用 return。

> 目前的Kotlin版本其实也可以在return后面使用\@来指明返回的哪一级的函数。

### 双层嵌套的lambda场景

```kotlin
inline fun lambdaReturnTest(insertAction: () -> Unit) {
    doubleLambda { insertAction() }
}

fun doubleLambda(insertAction: () -> Unit) {
    insertAction()
}
```

doubleLambda方法是一个普通函数，非内联函数，它的参数是一个函数类型的参数。

如果像这样带两层lambda调用，那么其中使用return就又会无法判断结束的到底是哪一层函数。 **这里Kotlin是直接禁止了这种写法。** 

如果确实要有这种间接调用需求，那么可以使用crossinline来解决。当你给一个需要被间接调用的参数加上 crossinline，就对它进行了局部加强内联，相当于insertAction还是会被展开铺平到调用处，解除了这个限制，从而就可以对它进行双层间接调用了。

但是又会有return结束层级不确定性，所以Kotlin规定了使用了crossinline的函数，不能在lambda参数中使用return。

只能二选一了。

## 总结
结论就是：
* inline 可以让你用内联——也就是函数内容直插到调用处——的方式来优化代码结构，从而减少函数类型的对象的创建；
* noinline 是局部关掉这个优化，来摆脱 inline 带来的「不能把函数类型的参数当对象使用」的限制；
* crossinline 是局部加强这个优化，让内联函数里的函数类型的参数可以被当做对象使用。
