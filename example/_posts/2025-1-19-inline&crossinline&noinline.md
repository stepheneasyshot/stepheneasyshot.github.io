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
# inline&crossinline&noinline
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

## crossinline
