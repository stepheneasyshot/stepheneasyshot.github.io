---
layout: post
description: > 
  本文介绍了Android中特有的Handler消息处理机制
image: 
  path: /assets/img/blog/blogs_handler_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_handler_cover.png
    960w:  /assets/img/blog/blogs_handler_cover.png
    480w:  /assets/img/blog/blogs_handler_cover.png
accent_image: /assets/img/blog/blogs_handler_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Handler消息机制的上下层设计与流程详解
Android UI 更新是线程不安全的，也就是说，你不能直接在子线程中操作 UI。所有 UI 操作都必须在主线程（或称 UI 线程） 中进行，Activity的所有生命周期回调和UI更新操作都发生在应用程序的主线程中。

这就引出了一个问题：如果后台线程完成了数据加载或计算，怎么才能通知主线程更新 UI 呢？答案就是 Handler 消息机制。

简单来说， **Handler 消息机制** 是一套允许你将任务（消息或可运行对象）发送到另一个线程的消息队列中，并在该线程中执行这些任务的框架。
## 使用场景
#### UI 线程更新
> 安卓规定，所有对 UI 的操作都必须在主线程（UI 线程）中进行。当你在子线程中执行耗时操作（如网络请求、数据库查询）并需要更新 UI 时，就不能直接在子线程操作 UI 元素。这时，Handler 就派上了用场。你可以在子线程中发送一个 **Message** 或 **Runnable** 到主线程的 Handler，然后 Handler 会在主线程中处理这个消息或运行这个 Runnable，从而安全地更新 UI。

#### 线程间通信
> 除了 UI 更新，Handler 机制也是实现线程间通用通信的重要方式。例如两个子线程之间的消息传递，还有Service与其他组件的通信，或者单个组件内部的消息处理。

#### 延迟任务和定时任务
> Handler 提供了 `postDelayed()` 和 `sendMessageDelayed()` 等方法，可以让你在指定的时间后执行某个任务或发送某个消息。例如，等待某个界面加载完成后再开始播放动画。每隔一段时间从服务器获取最新数据。应用发送验证码后的倒计时。还可以用于防抖过滤，在短时间内多次点击某个按钮时，可以使用 Handler 延迟处理点击事件，只响应第一次点击。

#### 异步任务的封装
> 许多安卓框架和库在底层也利用了 Handler 机制来处理异步任务，例如AsyncTask，虽然已被官方标记为 deprecated，但其内部实现就包含了 Handler 来处理子线程与主线程的通信。

#### 其他特定场景
> 当用户在 EditText 中输入时，系统会通过 Handler 将输入事件传递给对应的 View。另外，部分动画框架会使用 Handler 来调度动画的每一帧。还有安卓系统内部的很多事件（如触摸事件、生命周期事件）的分发都可能间接涉及到 Handler 机制。

可以看出，Handler最主要的任务就是实现线程间的消息传递和处理。这其中的核心需求，就是就是如何 **高效地等待消息**。

在 Android 应用中，主线程（UI 线程）负责处理用户界面更新和所有用户输入事件。一旦系统检测到主线程发生了长时间的阻塞，就会弹出ANR（Application Not Responding）弹窗并终结应用。为了避免ANR， **主线程不能被耗时的操作阻塞** 。

Looper 与 Handler 协同工作，为每个线程维护一个消息队列。Looper 会不断检查这个队列是否有新消息需要处理。

这里的关键挑战在于：Looper 如何 **在没有消息时高效地等待** ，而不是持续消耗 CPU 资源进行busy-waiting。传统的 **忙等待** 会极大地浪费电量并降低系统性能。因此，需要一种能够让线程在无事可做时进入睡眠状态，并在有新事件发生时被唤醒的机制。
## Linux体系和文件描述符简介
Linux系统中，应用程序到系统内核的体系架构：

![](/assets/img/blog/blogs_linux_app_arch.png)

Linux 操作系统的体系架构分为**用户态和内核态**（用户空间和内核空间），内核本质上可以说属于一种桥接软件，往下控制着计算机的硬件资源，往上又给上层应用程序提供运行的环境。

而平常所说的 **用户态** 就是上层应用程序的活动空间，应用程序的执行又依托于内核提供的资源，包括 **CPU 资源、存储资源、I/O 资源** 等，为了让上层应用能够访问这些资源，内核必须为上层应用提供访问的接口，也就是 **系统调用（system call）** 。

**系统调用** 是受控的内核入口，借助这一机制，进程可以请求内核以自己的名义去执行某些动作，以 API 的形式，内核提供有一系列服务供程序访问，包括创建进程、执行 I/O 以及为进程间通信创建管道等。
### 文件描述符
Linux 继承了 UNIX **一切皆文件** 的思想，在 Linux 中，所有执行 I/O 操作的系统调用都以文件描述符指代已打开的文件，包括**管道（pipe）、FIFO、Socket、终端、设备和普通文件等**。

**文件描述符（File Descriptor）** 是 Linux 中的一个索引值，系统在运行时有大量的文件操作，内核为了高效管理已被打开的文件会 **创建索引** ，用于 **指向被打开的文件** ，这个索引就是文件描述符。

文件描述符往往是数值很小的非负整数，获取文件描述符一般是通过系统调用 open() ， **在参数中指定 I/O 操作目标文件的路径名** 。
### 事件文件描述符enevtfd
**eventfd** 可以用于线程或父子进程间通信，内核通过 eventfd 也可以向用户空间发送消息，其核心实现是 **在内核空间维护一个计数器** ，向用户空间暴露一个与之关联的匿名文件描述符，不同线程通过 **读写该文件描述符通知或等待对方** ，内核则通过写该文件描述符通知用户程序。

在 Linux 中，很多程序都是事件驱动的，也就是通过 select/poll/epoll 等系统调用在一组文件描述符上进行监听，当文件描述符的状态发生变化时，应用程序就调用对应的事件处理函数，有的时候需要的 **只是一个事件通知** ，没有对应具体的实体，这时就可以使用 eventfd 。

与管道（pipe）相比，管道是半双工的传统 IPC 方式，两个线程就需要两个 pipe 文件，而 eventfd 只要打开一个文件，而文件描述符又是非常宝贵的资源，linux 的默认值也只有 1024 个。eventfd 非常节省内存，可以说就是一个计数器，是 **自旋锁 + 唤醒队列** 来实现的，而管道一来一回在用户空间有多达 4 次的复制，内核还要为每个 pipe 至少分配 4K 的虚拟内存页，就算传输的数据长度为 0 也一样。这就是为什么**只需要通知机制的时候优先考虑使用eventfd** 。

#### eventfd具体工作流程
eventfd 它创建了一个内核维护的 64 位计数器，并将其关联到一个文件描述符。这个文件描述符可以像普通文件描述符一样进行读写操作，并支持 select()、poll() 和 epoll() 等 I/O 多路复用机制，从而实现事件的等待和通知。

使用 `eventfd()` 系统调用可以创建一个 `eventfd` 对象，并返回一个文件描述符。你可以指定一个初始值 `initval` 来初始化计数器。

```cpp
#include <sys/eventfd.h>

int eventfd(unsigned int initval, int flags);
```

##### **核心机制：读写计数器**
* **写入 (write)**:
    * 当你向 `eventfd` 对应的文件描述符写入一个 8 字节的 `uint64_t` 值时，这个值会被加到 `eventfd` 内部的计数器上。
    * 如果写入会导致计数器溢出（超过 `uint64_t` 的最大值），`write()` 操作会阻塞，直到有 `read()` 操作消耗了计数器的一部分值，或者如果文件描述符是非阻塞模式，则会立即失败并返回 `EAGAIN`。
* **读取 (read)**:
    * 当你从 `eventfd` 对应的文件描述符读取一个 8 字节的 `uint64_t` 值时，会发生以下情况：
        * **非信号量模式（默认）**: `read()` 操作会返回当前计数器的值，并将计数器重置为 0。
        * **信号量模式（使用 `EFD_SEMAPHORE` 标志创建）**: `read()` 操作会返回 1，并将计数器减 1。
    * 如果计数器为 0，`read()` 操作会阻塞，直到有 `write()` 操作增加计数器的值，或者如果文件描述符是非阻塞模式，则会立即失败并返回 `EAGAIN`。

`eventfd` 的强大之处在于 **它与 Linux 的 I/O 多路复用机制（如 `select()`、`poll()` 和 `epoll()`）的集成** 。

  * 当 `eventfd` 计数器 **大于 0** 时，它被认为是 **可读的**。这意味着你可以使用 `select()`、`poll()` 或 `epoll()` 监测这个文件描述符的读事件，当事件发生时（计数器非零），你的程序就会被唤醒。
  * 当 `eventfd` 计数器 **小于 `0xfffffffffffffffe` (即 `uint64_t` 的最大值 - 1)** 时，它被认为是 **可写的**。这意味着你可以安全地向它写入一个 1 而不会导致溢出。

总的来说，`eventfd` 提供了一种简单、高效且灵活的事件通知机制，特别适用于需要通过文件描述符进行事件等待和信号传递的场景。

## epoll机制
**如何实现上述这种高效地等待呢？** 需要先了解下Linux上的三种IO机制：**select，poll和epoll**。

在Linux中，poll、select 和 epoll 都是用于​​I/O多路复用（I/O Multiplexing）​​的机制。它们的主要作用是让一个进程/线程能够 **同时监控多个文件描述符** （File Descriptors，FDs），比如套接字（sockets）、管道（pipes）等，以判断这些描述符是否可读、可写或发生异常，从而高效地处理多个I/O事件，而无需为每个FD创建单独的线程或进程。

这三种机制的核心目标都是：
* ​**​避免阻塞**​​：当一个进程需要同时监听多个I/O操作（如多个socket连接）时，不用为每个连接创建一个线程或进程，从而节省系统资源。
* ​**​提高效率**​​：通过一次系统调用，就可以检查多个FD的状态，而不是对每个FD逐一调用read/write等函数。

**1. select**

程序将自身所关注的 **文件描述符（FD）** 集合传给内核，内核检查这些FD是否有事件（可读、可写、异常），然后返回。由于FD集合大小有限（通常1024），且每次调用都需要重新传入FD集合，效率较低。​主要​缺点​​有三点，FD数量受限。每次调用都要传递整个FD集合，即使只有一个FD发生变化。内核使用线性扫描判断FD状态，效率低。

**2. poll**

​​工作原理​​与select类似，但使用pollfd结构体数组来表示FD集合，不再有1024的限制。可以支持更多的FD。​​缺点​​就是每次调用**仍然需要传递整个FD集合**。内核依然使用线性扫描，效率没有本质提升。

### `epoll`：可扩展的 I/O 事件监控器
**Android Native** 层的 Looper 实现利用了 Linux 内核提供的 **`epoll`** 系统调用。`epoll` 是一种高级的 I/O 事件通知机制，允许一个线程高效地 **监控多个文件描述符** ，以判断它们是否准备好进行 I/O 操作。

在 Looper 的上下文中，**`epoll_wait()`** 函数用于让线程进入睡眠状态，直到有事件发生。相较于早期机制如 `select` 和 `poll`，`epoll` 具有显著的优势：

* **可扩展性（Scalability）**：`epoll` 的性能随着被监控文件描述符数量的增加而保持良好。与 `select` 和 `poll` 每次调用都需要内核遍历所有文件描述符列表不同，`epoll` 在内核中维护了一个 **“兴趣列表”** 。当事件发生时，内核直接提供一个**已就绪的文件描述符的列表**，避免了昂贵的线性扫描。这对于 Android 这样复杂的系统至关重要，因为一个线程可能需要等待各种不同的事件源。
* **效率（Efficiency）**：通过将监控任务委托给内核，线程可以在不需要时保持低功耗的睡眠状态，直到真正需要被唤醒，从而节省电量并提高系统性能。
* **触发模式（Trigger Modes）**：`epoll` 支持边沿触发（Edge-Triggered, ET）和水平触发（Level-Triggered, LT）两种模式，为事件处理提供了更精细的控制。水平触发通知就是文件描述符上可以非阻塞地执行 I/O 调用，这时就认为它已经就绪。边缘触发通知就是文件描述符自上次状态检查以来有了新的 I/O 活动，比如新的输入，这时就要触发通知。

### `eventfd`：轻量级的唤醒信号
尽管 `epoll` 提供了高效的等待机制，但当另一个线程向目标线程的消息队列发送新消息时，需要一种方式来唤醒处于睡眠状态的 Looper 线程。这就是 **`eventfd`** 发挥作用的地方。

上文介绍过，eventfd被创建时，会维护一个64位的计数器，当线程向eventfd写入数据时，会将计数器加1，当线程从eventfd读取数据时，会将计数器减1。当这个值大于0，就说明是可读状态，当这个值小于溢出的最大值就认为其处于可写状态。

而在 Android Handler 框架中，`eventfd` 的工作原理如下：

1.  **创建**：当一个 Looper 在线程上初始化时，它的底层原生实现会创建一个 `eventfd` 文件描述符。
2.  **监控**：这个 `eventfd` 文件描述符随后被添加到 Looper 的 `epoll` 实例中，`epoll_wait()` 将会监控它。
3.  **唤醒**：当另一个线程向目标线程的 **`MessageQueue`** 发布消息时，原生 `MessageQueue` 代码会向 `eventfd` 写入一个 `1`。这个写入操作会触发 `eventfd`，使其变为“可读”状态。
4.  **解除阻塞**：阻塞 Looper 线程的 `epoll_wait()` 调用会立即返回，表明 `eventfd` 上有事件发生。
5.  **消息处理**：Looper 随后从其队列中检索并处理新消息。

### epoll实例及其核心api
epoll API 的核心数据结构称为 **epoll 实例** ，它与一个 **打开的文件描述符** 关联，这个文件描述符不是用来做 I/O 操作的，而是**内核数据结构的句柄**，这些内核数据结构实现了记录兴趣列表和维护就绪列表两个目的。

那么这两个列表里面都是一些什么内容呢？
#### 兴趣列表 (Interest List)
**兴趣列表** 是 `epoll` 实例维护的、内核中的一个数据结构。它记录了 `Looper` 线程**想要监听的所有文件描述符及其对应的事件类型**。

当 `Looper` 被创建时，或者添加新的监听事件源时，它会通过 `epoll_ctl()` 系统调用，将对应的文件描述符及其感兴趣的事件（例如读事件 `EPOLLIN`）添加到 `epoll` 实例的兴趣列表中。

在 Android Looper 的实现中，**兴趣列表**里主要包含两种类型的数据：

核心就是 **用于唤醒 Looper 的 `eventfd` 事件文件描述符** 。当其他线程（比如通过 `Handler.sendMessage()`）向 `Looper` 的消息队列发送消息时，最终会在底层向这个 `eventfd` 写入数据。这个写入操作会触发 `eventfd` 上的可读事件。`epoll` 就会监测到这个事件，从而唤醒正在 `epoll_wait()` 的 Looper 线程。这是 `Looper` 从睡眠中醒来处理新消息的核心机制。

另外还可能有**被监听的其他文件描述符** ，这些可以是其他 I/O 源的文件描述符，例如**管道 (pipes)**，可能用于更复杂的 IPC 场景。 **socket 文件描述符**：如果 Looper 线程也负责网络通信。**各种 Linux 特殊文件**：如 `inotify` (文件系统事件通知)、`timerfd` (定时器事件) 等，如果应用有特殊需求，都可以将其文件描述符添加到 Looper 的 `epoll` 监听中。

因为 `Looper` 不仅仅处理 Java 层面的消息，它 **在原生层也可以监听和处理各种系统事件**。通过将这些文件描述符加入兴趣列表，`Looper` 能够在一个统一的事件循环中同时处理 Java 消息和原生系统事件。

每个被添加到兴趣列表的文件描述符，通常还会附带一个与之关联的 **用户数据（user data）**。在 `epoll` 中，这个用户数据通常是一个 `epoll_data_t` 联合体，它可以是**一个指针或一个整数**。在 Android Looper 中，它通常被用来指向一个内部结构体，该结构体包含了与这个文件描述符相关的回调函数或上下文信息，以便在事件发生时能够正确地对应处理。
#### 就绪列表 (Ready List)
**就绪列表** 也是 `epoll` 实例维护在内核中的一个数据结构。它记录了当前**已经发生事件、可以进行 I/O 操作的文件描述符**。

当 `Looper` 线程调用 `epoll_wait()` 时，如果兴趣列表中的任何文件描述符上发生了它所关注的事件，内核就会将这些“就绪”的文件描述符及其发生的事件类型填充到就绪列表中。`epoll_wait()` 随即返回，`Looper` 线程就可以遍历这个就绪列表，对每个就绪的描述符执行相应的处理逻辑。

就绪列表中的数据通常包含：

1.  **就绪的文件描述符 (File Descriptor)：** 即兴趣列表中发生了事件的那个文件描述符。
2.  **发生的事件类型 (Events)：** 描述该文件描述符上发生了什么类型的事件，例如 `EPOLLIN`（有数据可读）、`EPOLLOUT`（可以写入数据）、`EPOLLERR`（发生错误）等。
3.  **对应的用户数据 (User Data)：** 这是在将文件描述符添加到兴趣列表时一同传入的那个用户数据。Looper 会利用这个用户数据来识别事件源，并执行对应的回调函数来处理事件。例如，如果是 `eventfd` 就绪，它就知道有新消息需要处理；如果是其他文件描述符就绪，它就知道对应的原生 I/O 事件发生了。

总结就是**兴趣列表让 `Looper` 可以声明它关注的所有事件源，而就绪列表则让 `Looper` 能够精确地知道哪些事件已经发生**，从而避免了不必要的轮询，实现了响应式和低功耗的事件驱动模型。
#### epoll四个主要的api
epoll API 由以下 4 个系统调用组成。

`epoll_create()` 创建一个 epoll 实例，返回代表该实例的文件描述符，有一个 size 参数，该参数指定了我们想通过 epoll 实例检查的文件描述符个数。

`epoll_creaet1()` 的作用与 epoll_create() 一样，但是去掉了无用的 size 参数，因为 size 参数在 Linux 2.6.8 后就被忽略了，而 epoll_create1() 把 size 参数换成了 flag 标志，该参数目前只支持 EPOLL_CLOEXEC 一个标志。

`epoll_ctl()` 操作与 epoll 实例相关联的列表，通过 epoll_ctl() ，我们可以增加新的描述符到列表中，把已有的文件描述符从该列表中移除，以及修改代表文件描述符上事件类型的掩码。

`epoll_wait() `用于获取 epoll 实例中处于就绪状态的文件描述符。
## Handler完整链路
Android消息机制流程图：
![](/assets/img/blog/blogs_android_event_cycle.jpg)

### 消息机制初始化
消息机制初始化流程就是 Handler、Looper 和 MessageQueue 三者的初始化流程，Handler 的初始化流程比较简单.

当你直接在 Activity 的 onCreate() 或其他 UI 线程回调中创建一个 Handler 时，通常不需要显式地初始化 Looper，因为系统已经为你做好了。安卓应用的入口点是 `ActivityThread` ，当应用进程启动时，ActivityThread 会被创建。在 ActivityThread 的 main() 方法中，它会调用 `Looper.prepareMainLooper()`。

**`Looper.prepareMainLooper()`**

这个静态方法是主线程 Looper 初始化的关键。它会检查当前线程是否已经有 Looper（避免重复创建）。
* 如果当前线程没有 Looper，则先​创建一个新的 MessageQueue​。再​创建一个 Looper 对象​​，并将 MessageQueue 关联到 Looper 上。
* 将 Looper 存储到当前线程的 ThreadLocal 中（确保每个线程有自己的 Looper）。

当我们通过 Java 层的 `Looper.prepare()` 或 `Looper.prepareMainLooper()` 方法初始化 Looper 时，它会触发 Native 层的对应操作。简而言之，**Native层的初始化过程**主要涉及以下几个关键步骤：

1. 创建 Native MessageQueue 对象：这是消息的实际存储和管理容器。
2. 创建 Native Looper 对象：它将与 MessageQueue 关联，并负责消息的调度和分发。
3. 初始化 epoll 实例：这是 Looper 高效等待消息的关键。
4. 初始化 eventfd：作为唤醒 Looper 的信号机制。

Native层的prepare方法： 

```cpp
// Native层 Looper.cpp (简化版)
void Looper::prepare() {
    // 1. 获取或创建 Looper 的 ThreadLocal 存储
    // Looper::TLS_KEY 是一个线程局部存储键，确保每个线程拥有独立的Looper实例
    // 如果当前线程已经有一个Looper，则会报错，因为一个线程只能有一个Looper。
    // Looper::gLooper 实际上是一个TLS (Thread Local Storage) 变量，
    // 它在每个线程中保存一个 Looper 指针。
    if (gLooper != nullptr) {
        // ... 抛出异常：一个线程只能prepare一次Looper
    }

    // 2. 创建 Native MessageQueue 对象
    // 这个MessageQueue对象是Handler消息的实际存储队列
    sp<MessageQueue> messageQueue = new MessageQueue();

    // 3. 创建 Native Looper 对象
    // Looper::create() 内部会完成 Looper 对象的核心创建和 epoll/eventfd 的初始化
    sp<Looper> looper = Looper::create(messageQueue);

    // 4. 将 Looper 对象存储到当前线程的 ThreadLocal
    // 这样，在当前线程中，任何Handler的构造函数都能通过Looper::getForThread()获取到它。
    gLooper = looper; 
}
```

通过Looper::create方法来创建Looper对象：

```cpp
// Native层 Looper.cpp (简化版)
sp<Looper> Looper::create(sp<MessageQueue> messageQueue) {
    // 1. 创建 Looper 实例
    sp<Looper> looper = new Looper(messageQueue);

    // 2. 初始化 epoll 实例
    // looper->mEpollFd = epoll_create1(EPOLL_CLOEXEC);
    // epoll_create1() 返回一个 epoll 实例的文件描述符 (mEpollFd)。
    // EPOLL_CLOEXEC 标志确保这个文件描述符在执行 execve() 系统调用时会被关闭，防止子进程意外继承。
    looper->mEpollFd = epoll_create1(EPOLL_CLOEXEC); 
    if (looper->mEpollFd < 0) {
        // ... 错误处理
    }

    // 3. 初始化 eventfd
    // looper->mWakeEventFd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
    // eventfd() 创建一个 eventfd 文件描述符 (mWakeEventFd)。
    // 初始值为0。
    // EFD_CLOEXEC 同样确保 execve() 时关闭。
    // EFD_NONBLOCK 表示这是一个非阻塞的eventfd，写入操作不会阻塞。
    looper->mWakeEventFd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK); 
    if (looper->mWakeEventFd < 0) {
        // ... 错误处理
    }

    // 4. 将 eventfd 添加到 epoll 的兴趣列表
    // looper->addFd() 是一个内部方法，用于将文件描述符添加到 epoll 监听列表。
    // 它通过 epoll_ctl(EPOLL_CTL_ADD, ...) 实现。
    // 监听 EPOLLIN 事件，表示 eventfd 有数据可读（即被写入了）。
    // 第四个参数 (LOOPER_ID_WAKE) 是一个标识符，用于在 epoll_wait 返回时识别是哪个文件描述符触发了事件。
    int result = looper->addFd(looper->mWakeEventFd, LOOPER_ID_WAKE, EPOLLIN, nullptr);
    if (result != 0) {
        // ... 错误处理
    }

    return looper;
}
```

Looper 的构造函数中会调用 `epoll_create1()` 创建一个 epoll 实例，然后再调用 epoll_ctl() 给 epoll 实例添加一个唤醒事件文件描述符，把唤醒事件的文件描述符和监控请求的文件描述符添加到 epoll 的兴趣列表中。到这里消息机制的初始化就完成了。
### 消息轮询机制建立
在 `ActivityThread` 的 `main()` 方法的最后，会调用 `Looper.loop()`。这个方法会使当前线程（主线程）进入一个无限循环。

这个循环中会调用 **MessageQueue 的 next()** 方法获取下一条消息，获取到消息后，`loop()` 方法就会调用 `Message` 的 target 的 `dispatchMessage()` 方法，target 其实就是发送 `Message` 的 Handler 。最后就会调用 `Message` 的 `recycleUnchecked()` 方法回收处理完的消息。

如果消息队列为空，Looper 线程会进入休眠状态（这正得益于底层 epoll 和 eventfd 的机制），直到有新消息到来。

这个循环就是安卓 UI 线程保持“存活”和响应用户事件的基础。

在 `MessageQueue` 的 `next()` 方法中，首先会调用 `nativePollOnce()` 这个JNI方法，检查队列中**是否有新的消息要处理**。
* 如果没有，那么当前线程就会在**执行到 `Native` 层的 `epoll_wait()` 时阻塞**。
* 如果有消息，而且**消息是同步屏障**，那就会找出或等待需要优先执行的异步消息。调用完 nativePollOnce() 后，如果没有同步屏障，或者取到到了异步或非异步消息，就会判断消息**是否到了要执行的时间**，是的话则返回消息给 Looper 处理准备分发，不是的话就重新计算消息的执行时间（when）。

在把消息返回给 Looper 后，下一次执行 nativePollOnce() 的 timeout 参数的值是默认的 0 ，所以进入 `next()` 方法时，如果没有消息要处理，`next()` 方法中还可以执行 `IdleHandler` 。在处理完消息后， `next()` 方法最后会遍历 `IdleHandler` 数组，逐个调用 `IdleHandler` 的 `queueIdle()` 方法。

> IdleHandler 可以用来做一些在主线程空闲的时候才做的事情，通过 Looper.myQueue().addIdleHandler() 就能添加一个 IdleHandler 到 MessageQueue 中。

以Java 层的 `Looper.loop()` 和 `MessageQueue` 的 `next()` 方法为入口，这个消息循环的建立，就是 `android::Looper` 利用 `epoll` 高效地等待并处理事件的过程。

在 Native 层，消息循环的建立主要围绕着 `android::Looper` 类的 `loop()` 方法和其内部调用的 `pollOnce()` 方法展开。
#### 1\. `Looper::loop()` - 消息循环的入口
当你在 Java 层调用 `Looper.loop()` 方法时，它会通过 JNI（Java Native Interface）机制，最终调用到 Native 层的 `android::Looper::loop()` 方法。

```cpp
// Native层 Looper.cpp (简化版)
void Looper::loop() {
    // 确保当前线程已经准备好了一个Looper
    sp<Looper> me = Looper::getForThread(); // 获取当前线程的Looper实例
    if (me == nullptr) {
        // 错误处理：当前线程没有调用 Looper.prepare()
        return;
    }

    // 这是一个无限循环，Looper 会一直在这里运行，直到被 quit() 终止
    for (;;) { // 无限循环，除非 Looper 被终止
        // 核心：调用 pollOnce() 等待并处理事件
        int result = me->pollOnce(-1); // -1 表示无限等待，直到有事件发生

        switch (result) {
            case POLL_WAKE: // 由 wake() 方法唤醒，通常表示有新消息或Runnable
                // 如果是Looper被唤醒，但还没消息，则会在这里继续循环
                break;
            case POLL_MESSAGE: // 收到了要处理的消息
                // pollOnce 内部会处理 MessageQueue 中的消息
                break;
            case POLL_TIMEOUT: // 超时 (如果 pollOnce 设置了超时时间)
                break;
            case POLL_ERROR: // 发生错误
                break;
        }

        // 如果 Looper 被终止 (调用了 quit() 或 quitSafely())
        if (me->mExitWhenIdle) { // mExitWhenIdle 是一个标志，表示Looper是否应该退出
            break; // 退出循环
        }
    }
}
```

从代码中可以看出，`Looper::loop()` 的核心就是在一个 **无限 `for` 循环** 中不断地调用 `me->pollOnce(-1)`。这个 `pollOnce` 方法才是 **真正执行阻塞、等待和初步事件处理** 的地方。
#### 2\. `Looper::pollOnce()` - 阻塞、唤醒与事件分发
`pollOnce()` 方法是 `Looper` 消息循环的精髓所在。它利用了 `epoll` 来高效地等待事件，避免了忙等待。

```cpp
// Native层 Looper.cpp (简化版)
int Looper::pollOnce(int timeoutMillis) {
    // 1. 处理待处理的消息 (如果有)
    // 首先检查 MessageQueue 中是否有即将到期或已经到期的消息需要处理。
    // 如果有，它会计算下一个消息的到期时间，并可能调整 epoll_wait 的超时时间。
    // 如果有立即需要处理的消息，直接返回 POLL_MESSAGE，不进入 epoll_wait。
    if (mMessageQueue->mNextBarrierToken != 0 || mMessageQueue->hasMessages(now)) {
        // 如果有消息，并且没到等待时间，会直接处理
        // 或者处理屏障消息，清理掉同步消息
        // ...
        timeoutMillis = 0; // 不阻塞，立即返回
    }

    // 2. 调用 epoll_wait 等待事件
    // mEpollFd 是在 Looper 初始化时创建的 epoll 实例文件描述符
    // events 是一个用于接收就绪事件的数组
    // EPOLL_MAX_EVENTS 是最大事件数
    // timeoutMillis 是等待的超时时间（-1 表示无限等待）
    int result = epoll_wait(mEpollFd, events, EPOLL_MAX_EVENTS, timeoutMillis);

    // 3. 处理 epoll_wait 返回的事件
    if (result < 0) { // epoll_wait 失败
        if (errno == EINTR) { // 被信号中断，继续循环
            return POLL_WAKE; // 被唤醒但没有明确事件
        }
        return POLL_ERROR; // 其他错误
    }

    // 4. 遍历就绪列表，处理事件
    for (int i = 0; i < result; i++) {
        epoll_event& event = events[i];
        int ident = event.data.u32; // 获取事件标识符 (Looper::addFd 时传入的)

        if (ident == LOOPER_ID_WAKE) { // 这是由 eventfd 触发的唤醒事件
            // 清除 eventfd 上的信号，以准备下一次唤醒
            uint64_t counter;
            ssize_t nread = read(mWakeEventFd, &counter, sizeof(uint64_t));
            // 唤醒通常意味着有新消息到达 MessageQueue，但具体消息由后续处理
            result = POLL_WAKE; 
        } else if (ident == LOOPER_ID_MESSAGE) { // 这是由 MessageQueue 自身触发的消息事件（不常用）
            // Android Looper主要通过 LOOPER_ID_WAKE 来感知消息，此分支较少触发
            result = POLL_MESSAGE;
        } else { // 处理其他文件描述符上的自定义事件
            // 如果Looper监听了其他自定义文件描述符（如管道、Socket），会在这里处理
            // 通常会调用与该fd关联的回调函数
            // ...
        }
    }

    // 5. 处理消息队列中的消息
    // 无论是否被 epoll 唤醒，都会再次检查并分发 MessageQueue 中的消息
    // 这是真正将消息从队列中取出并分发给对应 Handler 的地方。
    // 可能会调用 MessageQueue::next() 获取下一个消息，并调用 Handler 的 dispatchMessage()。
    if (mMessageQueue->hasMessages(now)) { // 再次检查是否有消息
         result = POLL_MESSAGE;
    }
    mMessageQueue->dispatchMessages(this, now); // 分发消息

    return result;
}
```

根据上面代码可以看出，调用了 `Looper::loop()` 后，即进入一个无限循环，在循环内部，不断调用 `Looper::pollOnce(-1)`。

`pollOnce()` 函数 首先检查 `MessageQueue` 中是否有立即需要处理的消息。如果有，会立即处理而不阻塞。

**核心逻辑** 为调用 `epoll_wait(mEpollFd, ...)`。此时，Looper 线程会进入 **睡眠状态**，等待 `mEpollFd` 监听的任何文件描述符上发生事件。
* 如果消息队列中有新消息，`MessageQueue` 会向 `mWakeEventFd` 写入一个字节。这会触发 `mWakeEventFd` 上的 `EPOLLIN` 事件。
* `epoll_wait()` 检测到 `mWakeEventFd` 事件，并立即返回。
* `pollOnce()` 遍历 `epoll_wait()` 返回的就绪事件列表。
* 如果检测到 `LOOPER_ID_WAKE` 事件（即 `mWakeEventFd` 被写入），它知道 Looper 被唤醒了，通常意味着有新的消息需要处理。它会读取 `eventfd` 的值来清除信号。
* 最后，`pollOnce()` 调用 `mMessageQueue->dispatchMessages()`，真正地从消息队列中取出消息，并通过 `Handler.dispatchMessage()` 分发给相应的 Handler 进行处理。

通过这种方式，在 Native 层构建了一个高效的事件循环。 **利用 `epoll` 集中监听各种 I/O 事件，并用 `eventfd` 作为轻量级的内部唤醒信号**，确保了在没有任务时线程可以休眠，而在有任务时能够被迅速、精确地唤醒，从而实现安卓系统流畅且低功耗的响应。

### 消息发送
#### 插入：Message对象
Message 的实现。Message 中的 what 是消息的标识符。而 arg1、arg2、obj 和 data 分别是可以放在消息中的整型数据、Object 类型数据和 Bundle 类型数据。when 则是消息的发送时间。

sPool 是全局消息池，最多能存放 50 条消息，一般建议用 Message 的 obtain() 方法复用消息池中的消息，而不是自己创建一个新消息。如果在创建完消息后，消息没有被使用，想回收消息占用的内存，可以调用 recycle() 方法回收消息占用的资源。如果消息在 Looper 的 loop() 方法中处理了的话，Looper 就会调用 recycleUnchecked() 方法回收 Message 。

#### 消息发送
当我们用 Handler 的 **sendMessage() 、 sendEmptyMessage() 和 post()** 等方法发送消息时，首先会创建或者复用一个 `Message` 对象。这个 Message 对象包含了消息类型、数据以及目标 Handler 等信息。 

这几个发送消息的方法最终都会走到 Handler 的 enqueueMessage() 方法。Handler 的 enqueueMessage() 又会调用 `MessageQueue` 的 `enqueueMessage()` 方法。

enqueueMessage() 首先会判断，当**没有更多消息、消息不是延时消息、消息的发送时间早于上一条消息**这三个条件其中一个成立时，就会把当前消息作为链表的头节点，然后如果 IdleHandler 都执行完的话，就会调用 nativeWake() JNI 方法唤醒消息轮询线程。

如果上述三个条件**都不成立**，就会遍历消息链表，当遍历到最后一个节点，或者发现了一条早于当前消息的发送时间的消息，就会结束遍历，然后把遍历结束的最后一个节点插入到链表中。如果在遍历链表的过程中发现了一条异步消息，就不会再调用 nativeWake() JNI 方法唤醒消息轮询线程。

这一步可以确定**当前发送的消息应该放到消息链表的哪个位置**。

### 消息处理
#### **从 MessageQueue 中取出消息**

一旦 pollOnce() 被唤醒并返回，Native Looper 会调用 Native MessageQueue 的相应方法（例如 next()）。MessageQueue 会：

* 加锁保护： 同样，在访问消息队列时会进行加锁操作（例如 pthread_mutex_lock），确保线程安全。
* 遍历链表： 从内部的链表结构中取出最需要处理的那个消息（即 when 值最小且已到期的消息）。
* 移除消息： 将取出的消息从队列中移除。
* 解锁： 释放锁。

#### **将 Native 消息转回 Java Message**
Native Looper 取出 Native Message 对象后，需要将其封装回 Java 层的 Message 对象，以便 Java 层的 Handler 能够理解和处理。这通常通过 JNI 调用来实现，将 Native Message 的数据（如 what, arg1, arg2, obj 指针等）填充到 Java Message 对象中。

#### **派发消息到 Java 层**
一旦 Java Message 对象准备好，Native 层会再次通过 JNI 调用 Java Message 对象的 target (也就是 Handler) 的 dispatchMessage() 方法。

```cpp
// 简化示意，非完整代码
// 在 Native Looper 的 C++ 代码中，执行类似以下的操作
// jniEnv->CallVoidMethod(javaMessageObj, dispatchMessageMethodID);
```

#### **dispatchMessage**
最后，消息回到了 Java 层，由目标 Handler 的 `dispatchMessage()` 方法接收。dispatchMessage() 方法会根据消息的 callback (通过 post() 方法发送的 Runnable) 或 what 值，最终调用 handleMessage() 方法来处理具体的业务逻辑。

此前在 Handler 的 enqueueMessage() 方法中，会设置 Message 的 target 为当前 Handler 对象。

Handler 的 dispatchMessage 方法的优先级顺序：
* 如果 Message.callback（即 post(Runnable) 的 Runnable）不为空，执行 callback.run()。
* 如果 Handler.mCallback（Handler 的构造函数传入的 Callback 对象）不为空，调用 mCallback.handleMessage(msg)。
* 否则调用 Handler.handleMessage(msg)。

## 经典问题点
### postDelay消息是如何实现的？
当你使用 `Handler.postDelayed(Runnable r, long delayMillis)` 或 `Handler.sendMessageDelayed(Message msg, long delayMillis)` 发送延时消息时，Handler 机制会利用底层的一些巧妙设计来确保消息在指定的时间后才被处理。其核心在于 **消息队列的排序** 和 **Looper 的休眠/唤醒机制**。

下面我们来深入了解一下 `postDelayed` 的底层实现原理：

#### **消息的封装与时间戳**
当你调用 `postDelayed` 时，Handler 会创建一个 `Message` 对象（如果是 `postDelayed(Runnable r, ...)`，Runnable 会被封装到 Message 的 `callback` 字段中）。这个 `Message` 对象会被赋予一个关键属性：**`when`**。

`when` 字段表示的是消息应该被处理的 **绝对时间**，计算方式是：

```
when = SystemClock.uptimeMillis() + delayMillis
```

> SystemClock.uptimeMillis()：返回系统开机以来的毫秒数，不包括深度睡眠的时间。这是一个稳定的、适合计算延时的时钟源。delayMillis就是你指定的延时时长。

所以，`when` 字段就存储了这条延时消息的“到期时间”。
#### **消息入队与排序**
`MessageQueue` 并不是一个简单的 FIFO （先进先出）队列，它实际上是一个 **有序队列**，消息会根据它们的 `when` 值进行插入，确保队列中的消息始终按 `when` 值从小到大（即按到期时间从早到晚）的顺序排列。

当一个延时消息被发送并准备入队时，`MessageQueue` 会遍历现有消息，将其**插入到正确的位置**，以保持队列的**有序性**。这意味着，到期时间最早的消息总是在队列的头部。

#### **Looper 的休眠与唤醒**
这是实现延时消息的关键部分。`Looper` 在它的无限循环中，会不断地调用 `MessageQueue.next()` 方法来获取下一个要处理的消息。此方法会检查队列头部的消息。
* 如果队列头部有消息，并且该消息的 `when` 值已经小于或等于当前 `SystemClock.uptimeMillis()`（即消息已到期），那么 `next()` 方法会立即返回该消息，Looper 会立即处理它。
* 如果队列头部有消息，但它的 `when` 值大于当前时间（即消息还未到期），`next()` 方法会计算一个 **`nextPollTimeoutMillis`**。这个超时时间就是当前时间到队列头部消息到期时间之间的时间差。这个超时时间会告诉底层的阻塞机制，Looper 最多可以休眠多久。

```
nextPollTimeoutMillis = 队列头部消息的when - SystemClock.uptimeMillis()
```

#### **Native 层的阻塞**:
计算出 `nextPollTimeoutMillis` 后，`MessageQueue.next()` 会调用到其 Native 层的实现。在 Native 层，Looper 会利用 Linux 内核的 **`epoll_wait`** 机制，传入这个 `nextPollTimeoutMillis` 作为超时参数。
* **如果 `nextPollTimeoutMillis` 大于 0**: Looper 线程会进入阻塞状态，最长休眠 `nextPollTimeoutMillis` 毫秒。这意味着线程会暂停执行，不会消耗 CPU 资源，直到：
    * 指定的时间过去（消息到期）。
    * 有新的消息入队（新的消息可能到期时间更早，需要提前唤醒）。
    * 有其他文件描述符事件发生（例如，用户输入、网络数据到达等）。
* **如果 `nextPollTimeoutMillis` 小于或等于 0**: 说明队列头部的消息已经到期或没有延时，Looper 会立即返回并处理消息，不会阻塞。

#### **唤醒与消息处理**
当 Looper 休眠的时间达到 `nextPollTimeoutMillis` 后，它会自动被系统唤醒(系统直接写eventfd描述符，通过epoll链路通知)。唤醒后，它会再次调用 `MessageQueue.next()`，此时原先的延时消息应该已经到期，于是被取出并分发处理。

**新消息提前唤醒**: 如果在 Looper 休眠期间，有新的消息入队，并且这个新消息的 `when` 值比当前队列头部消息的 `when` 值更早（即新消息需要更早处理），或者 Looper 根本就没有休眠，那么 `MessageQueue` 会通过直接写入数据的方式，立即 **唤醒** 正在休眠的 Looper 线程。Looper 线程被唤醒后，会重新计算下一次休眠时间，或者直接处理更早到期的消息。
