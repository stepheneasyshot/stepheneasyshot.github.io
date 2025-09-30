---
layout: post
description: > 
  本文介绍了JVM平台上的try-catch机制实现方式及使用中的注意事项
image: 
  path: /assets/img/blog/blogs_jvm_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_jvm_cover.png
    960w:  /assets/img/blog/blogs_jvm_cover.png
    480w:  /assets/img/blog/blogs_jvm_cover.png
accent_image: /assets/img/blog/blogs_jvm_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【通用开发】Java平台的try-catch机制简介
每天都在使用try-catch，但是没有对其设计理念和工作机制做一个比较详细的了解，现稍微总结下。

JVM 上的 try-catch 机制的实现是基于 Java 字节码层面的一些特殊指令和数据结构来完成的。

Java 中的异常是指程序运行时发生的意外情况，它会中断正常的指令流。Java 将异常分为两大类：
* ​​检查型异常(Checked Exception)​​：必须显式捕获或声明抛出(如 IOException)
* ​​非检查型异常(Unchecked Exception)​​：包括 RuntimeException 及其子类(如 NullPointerException)

try-catch 机制提供了一种结构化的方式来捕获并处理可能发生的异常，防止程序因未处理的异常而突然终止，斌且在异常发生后执行清理或恢复操作。
## 实现原理
### 大体流程
当异常被抛出时，JVM 的执行流程如下：

* 创建异常对象(调用异常类的构造函数)
* 查找当前方法的异常处理表：
    * 如果找到匹配的 catch 块，跳转到对应的处理代码
    * 如果没有找到，弹出当前栈帧，回到调用者方法
* 重复步骤2，直到找到匹配的 catch 块或到达线程栈底(此时线程终止)

### **1. 异常表 (Exception Table)**

这是 JVM 实现 try-catch 机制的关键。在 Java 字节码中，每个方法都会有一个或多个异常表条目。每个异常表条目通常包含以下信息：

* **start_pc (start program counter):** try 块开始的字节码指令偏移量。
* **end_pc (end program counter):** try 块结束的字节码指令偏移量（不包含此指令）。
* **handler_pc (handler program counter):** catch 块开始的字节码指令偏移量。
* **catch_type (catch type):** 捕获的异常类型（例如 `java.lang.ArithmeticException`）。如果为 0，则表示捕获所有 `Throwable` 异常（类似于 `finally` 块或泛型异常捕获）。

当 Java 编译器将 `.java` 文件编译成 `.class` 文件时，它会为 `try-catch` 结构生成相应的字节码和异常表条目。

### **2. 异常的抛出 (Throwing an Exception)**

当程序执行到 `try` 块内，如果发生异常（例如除零错误、空指针等），或者代码中显式地使用 `throw` 语句抛出异常时，JVM 会根据抛出的异常类型**创建一个相应的异常对象**（例如 `ArithmeticException`）。

然后从当前方法的异常表中，从下往上（或者从内到外，取决于异常表的组织方式）查找与当前执行位置和异常类型匹配的异常表条目。异常发生的 `pc` 值必须在 `start_pc` 和 `end_pc` 之间。抛出的异常类型必须是 `catch_type` 指定的异常类型或其子类。

如果找到了匹配的 `catch` 块，JVM 会将程序计数器（`pc`）设置为该 `catch` 块的 `handler_pc`，并将其异常对象压入操作数栈的顶部。然后，控制流将转移到 `catch` 块的代码继续执行。

如果当前方法没有找到匹配的 `catch` 块，JVM 会将异常沿着调用栈向上抛出，直到找到一个能够处理该异常的方法。如果一直抛到 `main` 方法，仍然没有被捕获，那么 JVM 会终止程序的执行，并打印异常的堆栈信息。

### **3. finally 块的实现**

`finally` 块是用来确保其中的代码无论是否发生异常都会执行。

在编译阶段，编译器会将 `finally` 块的代码 **复制到所有可能的出口点** ，包括 `try` 块正常结束、`try` 块中发生异常被 `catch` 块处理后、以及 `try` 或 `catch` 块中使用了 `return`、`break` 或 `continue` 等语句提前退出时。

如果 `finally` 块是在一个异常被捕获后执行的，并且 `finally` 块自身没有抛出新的异常，那么原来的异常会重新抛出（如果它没有被 `catch` 块完全处理）。如果 `finally` 块自身抛出了新的异常，则会覆盖掉之前的异常。

### **4. try-with-resources 的实现**

`try-with-resources` 是 Java 7 引入的语法糖，用于自动管理资源。在编译时，它会被转换成包含隐式 `try-finally` 结构的字节码，确保资源在使用完毕后（无论是否发生异常）都被正确关闭。这通常通过调用资源的 `close()` 方法来实现。

### 使用建议
* ​捕获特定异常​​：避免捕获过于宽泛的 Exception，尽量捕获具体的异常类型
* ​不要忽略异常​​：空的 catch 块是糟糕的做法，至少应该记录日志
* 合理使用 finally​​：用于资源释放等必须执行的操作
* ​避免在 finally 中抛出异常​​：这会掩盖原始异常
* ​考虑异常链​​：抛出新的异常时，保留原始异常信息(使用带 cause 的构造函数)
* ​合理使用自定义异常​​：为特定业务场景创建有意义的异常类型
* 性能方面：
    * ​​异常实例化开销​​：创建异常对象比普通对象开销大(需要填充栈轨迹)​* 控制流改变​​：异常处理会导致控制流跳转，比正常的条件判断慢
    * 不要用异常进行常规流程控制
    * 对于可预见的错误情况，优先使用条件判断
    * 只在真正异常的情况下使用 try-catch

## 不同线程的try-catch
Android开发中，经常有第三方依赖库的设计问题，单独开辟另一个线程，导致报错直接在其内部抛出，没有给到外部调用方进行处理，导致程序崩溃。

**不同线程之间不能直接使用 `try-catch` 块来捕获另一个线程中抛出的异常。**

每个线程都有自己的 **独立的执行栈** 。当一个线程抛出异常时，JVM 会沿着该线程的调用栈向上查找匹配的 `catch` 块。如果当前线程的调用栈中没有找到能够处理该异常的 `catch` 块，该线程就会终止。

**为什么不能直接捕获？**

想象一下，如果一个线程可以捕获另一个线程的异常，那么这会带来很多复杂性和不确定性：

* **时间同步问题:** 哪个线程应该先捕获？如果多个线程都尝试捕获同一个异常，谁会成功？
* **状态不一致:** 一个线程抛出异常可能意味着它的内部状态已经损坏。如果另一个线程捕获了这个异常并继续执行，可能会导致数据不一致或其他不可预测的行为。
* **程序的复杂性:** 线程间的异常捕获会使程序的控制流变得非常复杂和难以理解。

**如何在多线程环境中处理异常？**

尽管不能直接跨线程 `try-catch`，但 Java 提供了多种机制来在多线程环境中处理异常：

1. **在线程内部使用 `try-catch`：** 这是最常见和推荐的做法。每个线程都应该在其 `run()` 方法内部（或 `Callable` 的 `call()` 方法内部）使用 `try-catch` 块来处理它自己可能抛出的异常。这样可以确保即使线程内部发生错误，也不会导致整个应用程序崩溃，并且可以在线程内部进行适当的错误日志记录或恢复操作。

   ```java
   class MyRunnable implements Runnable {
       @Override
       public void run() {
           try {
               // 线程内部的业务逻辑，可能抛出异常
               int result = 10 / 0; // 抛出 ArithmeticException
               System.out.println("Result: " + result);
           } catch (ArithmeticException e) {
               System.err.println("线程内部捕获到 ArithmeticException: " + e.getMessage());
               // 进行错误日志记录、清理操作等
           } catch (Exception e) {
               System.err.println("线程内部捕获到通用异常: " + e.getMessage());
           }
       }
   }

   public class Main {
       public static void main(String[] args) {
           Thread thread = new Thread(new MyRunnable());
           thread.start();
           // main 线程不会捕获 MyRunnable 线程内部的异常
           System.out.println("Main thread finished.");
       }
   }
   ```

2. **`Thread.UncaughtExceptionHandler`：** 如果一个线程内部没有捕获异常，并且该异常导致线程终止，JVM 会调用该线程的 `UncaughtExceptionHandler`。你可以为每个线程或为所有线程设置一个默认的未捕获异常处理器。这对于日志记录未预期的异常非常有用。

   ```java
   class MyRunnable implements Runnable {
       @Override
       public void run() {
           // 这里没有 try-catch
           int result = 10 / 0; // 抛出 ArithmeticException
           System.out.println("Result: " + result);
       }
   }

   public class Main {
       public static void main(String[] args) {
           Thread thread = new Thread(new MyRunnable());

           // 为特定线程设置未捕获异常处理器
           thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
               @Override
               public void uncaughtException(Thread t, Throwable e) {
                   System.err.println("线程 " + t.getName() + " 发生了未捕获异常: " + e.getMessage());
                   e.printStackTrace(); // 打印堆栈信息
               }
           });

           // 也可以设置全局的默认未捕获异常处理器
           // Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
           //     @Override
           //     public void uncaughtException(Thread t, Throwable e) {
           //         System.err.println("全局捕获到线程 " + t.getName() + " 的未捕获异常: " + e.getMessage());
           //     }
           // });

           thread.start();
           System.out.println("Main thread finished.");
       }
   }
   ```

3. **使用 `Callable` 和 `Future`：** 如果你使用 `ExecutorService` 来管理线程，并且希望从子线程中获取结果或捕获异常，可以使用 `Callable` 接口而不是 `Runnable`。`Callable` 的 `call()` 方法可以抛出检查异常，并且它的结果（包括异常）可以通过 `Future` 对象来获取。

   ```java
   import java.util.concurrent.*;

   class MyCallable implements Callable<String> {
       @Override
       public String call() throws Exception {
           // 线程内部的业务逻辑
           if (Math.random() > 0.5) {
               throw new RuntimeException("随机抛出的异常！");
           }
           return "任务完成";
       }
   }

   public class Main {
       public static void main(String[] args) {
           ExecutorService executor = Executors.newSingleThreadExecutor();
           Future<String> future = executor.submit(new MyCallable());

           try {
               String result = future.get(); // 获取任务结果，如果发生异常，会抛出 ExecutionException
               System.out.println("任务结果: " + result);
           } catch (InterruptedException e) {
               Thread.currentThread().interrupt(); // 恢复中断状态
               System.err.println("主线程被中断: " + e.getMessage());
           } catch (ExecutionException e) {
               System.err.println("任务执行异常: " + e.getCause().getMessage()); // 获取实际的异常
               e.getCause().printStackTrace();
           } finally {
               executor.shutdown();
           }
       }
   }
   ```
