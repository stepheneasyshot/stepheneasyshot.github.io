---
layout: post
description: > 
  本文介绍了Android，Linux，Java相结合的一些IO模型和底层原理
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
# Android I/O 模型
## Java的I/O模型
Java 的 I/O 模型 指的是 Java 如何处理输入和输出（Input/Output）操作的方式，尤其是在涉及文件、网络、控制台等数据流时的读写机制。不同的 I/O 模型在性能、阻塞行为、线程使用等方面有显著差异。

Java 的 I/O 模型经历了几个发展阶段：传统的阻塞式 I/O（`java.io`）、NIO（`java.nio`）和 NIO.2（`java.nio.file`）。

### 阻塞 I/O (Blocking I/O - BIO)
Java 传统的 I/O 基于流的概念，将数据视为连续的字节序列或字符序列。 

**字节流** ，`InputStream` 和 `OutputStream` 是所有字节流的抽象基类，用于处理原始字节数据（例如图片、音频、视频文件）。常见的实现类有 `FileInputStream`, `FileOutputStream`, `BufferedInputStream`, `BufferedOutputStream`, `DataInputStream`, `DataOutputStream`, `ObjectInputStream`, `ObjectOutputStream` 等。

**字符流** ，`Reader` 和 `Writer` 是所有字符流的抽象基类，用于处理字符数据（例如文本文件）。它们处理字符编码，能够将字节流转换为字符流。常见的实现类有 `FileReader`, `FileWriter`, `BufferedReader`, `BufferedWriter`, `InputStreamReader`, `OutputStreamWriter` 等。

传统的 I/O 是 **阻塞式 (Blocking)** 的。当一个线程执行 I/O 操作（如读取文件或网络数据）时，它会 **一直等待直到操作完成** ，期间该线程无法执行其他任务。这在并发场景下会导致性能问题，因为每个连接可能需要一个独立的线程来处理。适用于客户端数量较少、连接时间较短的场景，例如简单的文件读写。

### NIO (Non-blocking I/O - NIO)
NIO 在 Java 1.4 中引入，旨在解决传统 I/O 的阻塞问题，提供更高效、可伸缩的 I/O 操作。

基于 **通道 (Channel)**，表示到实体（如文件、网络套接字）的连接，通过通道读写数据。它与传统 I/O 的流不同，通道是双向的，可以同时进行读写操作。

常见实现类有 `FileChannel`, `SocketChannel`, `ServerSocketChannel`, `DatagramChannel` 等。所有数据都 **通过缓冲区读写** 。数据从通道读取到缓冲区，或者从缓冲区写入通道。缓冲区提供了对数据的结构化访问，支持在内存中对数据进行高效操作。最常用的是 `ByteBuffer`，此外还有 `CharBuffer`, `IntBuffer` 等。

NIO模型还允许 **单个线程管理多个通道** 。通过选择器，一个线程可以监控多个通道上的 I/O 事件（如连接就绪、读就绪、写就绪），从而实现非阻塞 I/O。这大大减少了线程创建和切换的开销，提高了服务器的并发处理能力。

NIO 是 **非阻塞式 (Non-blocking)** 的。当 I/O 操作无法立即完成时，线程不会被阻塞，而是可以去执行其他任务。当 I/O 准备就绪时，选择器会通知线程。相较于传统I/O，其特点有：

* 数据读写都通过缓冲区。
* 非阻塞，I/O 操作不阻塞线程。
* 多路复用，单个线程可以处理多个 I/O 通道，提高并发性能。
* 复杂性，相对于传统 I/O，NIO 的编程模型更复杂，需要管理缓冲区、通道和选择器。

适用于高并发、大量连接的场景，如网络服务器。

### AIO (Asynchronous I/O)
AIO，也叫 `NIO.2` 在 Java 7 中引入，主要增加了异步 I/O (Asynchronous I/O) 功能，以及对文件系统操作的增强 (`java.nio.file` 包)。

异步通道允许在 I/O 操作完成后，通过回调函数或 `Future` 对象来处理结果，而不需要显式地等待。例如 `AsynchronousFileChannel`, `AsynchronousSocketChannel`, `AsynchronousServerSocketChannel`。还定义了 `CompletionHandler` 作为I/O 操作完成时调用的回调接口，可以处理成功或失败的情况。

Path/Files，提供了更现代、功能更丰富的文件系统 API，解决了 `java.io.File` 类的一些局限性，例如更好的异常处理、符号链接支持、元数据访问等。

AIO 是**异步式 (Asynchronous)** 的。I/O 操作由操作系统执行，当操作完成时，操作系统会通知应用，应用可以注册回调函数来处理结果。这进一步减少了应用层面的线程管理负担。

适用于需要极致性能和扩展性的大型应用，例如高并发网络服务器、大数据处理。

## 常见分类标准
### 阻塞和非阻塞

> 阻塞IO是指在执行IO操作时，如果没有数据可用或者IO操作还没有完成，那么当前线程会被挂起，直到数据准备好或者IO操作完成。这种方式会导致线程阻塞，无法执行其他任务，适用于需要等待IO操作完成的场景。

> 非阻塞IO是指在执行IO操作时，如果没有数据可用或者IO操作还没有完成，当前线程不会被阻塞，而是会立即返回一个状态码或者错误码，告诉调用者当前IO操作还没有完成。这种方式不会导致线程阻塞，适用于需要同时处理多个IO操作的场景。

Java 中的 NIO 是非阻塞 IO，当用户发起读写的时候，线程不会阻塞，之后，用户可以通过轮询或者接受通知的方式，获取当前 IO 调度的结果。
### 缓冲IO和直接IO

> 非直接IO也称为缓冲IO，是大多数操作系统默认的文件访问方式。数据先被 **复制到内核缓冲区** ，然后再 **从内核缓冲区复制到用户空间** 。可以减少实际磁盘操作次数，利用预读(read-ahead)和延迟写(write-behind)优化性能。但是数据需要在内核和用户空间之间 **多次拷贝** ，在某些场景下可能增加延迟。适合小文件或随机访问

> 直接IO绕过内核缓冲区，直接在用户空间和存储设备之间传输数据。数据直接在用户空间和设备间传输，可以 **减少数据拷贝次数** 。这种方式，每次IO操作都是实际的设备操作。同时要求IO大小和内存对齐符合设备要求。适合大文件传输或已知访问模式的应用

**缓冲是针对标准库的**

Linux 标准库定义了很多操作系统的基础服务，比如输入/输出、字符串处理等等。Android 操作系统的标准库是 `Bionic` ，它可是应用层联系内核的桥梁，我们也可以通过 NDK 访问 Bionic。

使用标准库进行 IO 我们称为缓冲 IO，我们读文件的时候，经常遇到，读完一行才会让输出，在 Android 内部也做了类似的处理。

**直接是针对内核的**

使用 Binder 跨进程传递数据的时候，需要 **将数据从用户空间传递到内核空间** ，非直接 IO 也这样，内核空间会多做一层页缓存，如果做直接 IO，应用程序会直接调用文件系统。

> Android 的 Binder 通信 既不是 直接 I/O，也不是 非直接 I/O ，而是一种 进程间通信 机制。它主要依赖 **内存映射（mmap）** 和 **内核缓冲区** 来实现高效的数据传输，而不是传统的文件 I/O 方式。Binder 使用 mmap() 在内核和用户空间之间建立 共享内存区域，避免数据在用户态和内核态之间的多次拷贝。发送方（Client）和接收方（Server）通过这块共享内存进行数据交换。Binder 驱动维护一个 内核缓冲区，用于临时存储跨进程传递的数据。数据先写入内核缓冲区，再通过共享内存传递给目标进程。Binder 通过 mmap 实现 一次拷贝（用户空间 → 内核缓冲区 → 目标进程用户空间），提高效率。Binder 不属于传统IO，因为它 **不涉及磁盘读写，而是纯内存操作** （进程间通信）。

缓冲和非直接 IO 就像 IO 调度的一级和二级缓存，为什么要做这么多缓存呢？因为操作磁盘本身就是消耗资源的，不加缓存频繁 IO 不仅会耗费资源也会耗时。

### Android平台的 IO
在 Android 平台上，IO（输入/输出）操作是应用与外部世界（如文件系统、网络、硬件设备）进行交互的基础。

> 首先确定一个原则，Android 平台的任何耗时操作，都应该在后台线程中进行，主线程需要尽量保持不出现耗时大于16ms的非UI任务，避免出现卡顿现象。主线程的事件循环如果被卡顿5s以上，还会出现ANR的问题。

以下是 Android 平台上常见的 IO 模型及其特点：

#### **阻塞 IO (Blocking I/O)**
这是最简单、最直观的 IO 模型。当一个线程执行 IO 操作时（例如，从文件中读取数据，或者向网络发送数据），该线程会被阻塞，直到 IO 操作完成。编程模型简单，易于理解和实现。

需要注意的是，如果在 Android 的主线程（UI 线程）上执行阻塞 IO 操作，会导致应用无响应（ANR - Application Not Responding）错误，给用户带来糟糕的体验。在高并发场景下，每个连接都需要一个线程，线程切换的开销很大。

所以，仅适用于 **非常小的、不频繁的** IO 操作，或者在专门的后台线程中进行。例如应用启动时，需要从 res/raw 或 assets 目录读取一个非常小的、固定的配置字符串，比如一个 API Key 或者一个版本号，并且这个读取操作是在一个 **独立的后台线程中** 进行的。

#### **非阻塞 IO (Non-blocking I/O)**
线程在进行 IO 操作时，如果数据尚未准备好，IO 调用会立即返回一个状态码，而不是阻塞线程。开发者需要通过轮询（polling）来检查 IO 操作是否完成。避免了线程阻塞，提高了线程的利用率。

这种非阻塞的模式，需要不断轮询，增加了编程的复杂性。而且频繁的轮询会消耗大量的 CPU 资源。

在 Android 开发中，直接使用纯非阻塞 IO 的场景相对较少。例如要自定义一个 **网络服务器框架的底层** ，网络通信库或服务器框架（例如，一个模拟 Netty 行为的 Android 本地服务器），你可能会利用 Java NIO 的 SocketChannel 在非阻塞模式下读写数据。这种模式的编程难度很高，需要开发者手动管理缓冲区、检查返回状态码，并且处理各种边缘情况。

#### **多路复用 IO (I/O Multiplexing)**
这种模型允许单个线程同时监听多个 IO 流（或文件描述符）的事件。当任何一个 IO 流准备好读写时，系统会通知该线程，然后线程可以对相应的 IO 流进行操作。常见的实现包括 `select`、`poll` 和 `epoll`（在 Linux 内核中）。

一个线程可以处理多个连接，避免了大量线程创建和切换的开销。可以避免不必要的阻塞，线程只在有 IO 事件发生时才被唤醒。同样的，编程模型相对复杂，需要对底层系统调用有一定了解。

例如在 Android 应用中，如果需要实现一个轻量级的 **本地服务器或者 P2P 连接** ，多路复用 IO 是一种高效的选择。

还有 **异步网络请求库的底层** ，许多流行的网络请求库（如 OkHttp）底层都可能利用了多路复用 IO 的思想来管理并发网络连接。

#### **异步 IO (Asynchronous I/O - AIO)**
线程发起 IO 操作后，立即返回，无需等待 IO 完成。当 IO 操作真正完成时，系统会通知发起者（通常通过回调函数、事件或 Future/Promise 模式）。

线程无需等待 IO 完成，可以立即执行其他任务，进一步提高了并发性。相对于轮询， **回调函数** 的方式可以简化编程逻辑。如果是 RxJava 这种框架，当嵌套的回调过多，可能导致代码难以阅读和维护（Callback Hell）。

使用上，例如 **Retrofit + OkHttp + Coroutines** 推荐组合。Retrofit 定义接口，OkHttp 执行实际的网络请求，Coroutines (suspend 函数和 Dispatchers.IO) 负责在后台线程执行请求并在请求完成后通知 UI 线程。

还有像大文件和频繁的读写。

```kotlin
suspend fun saveLargeFile(data: ByteArray, fileName: String) {
    withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        FileOutputStream(file).use { fos ->
            fos.write(data)
        }
    }
}
```

### 扩展：Android 阻塞IO 调用流程

