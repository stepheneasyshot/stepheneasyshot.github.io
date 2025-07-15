---
layout: post
description: > 
  本文介绍了Android Binder机制通信流程和背后的原理
image: 
  path: /assets/img/blog/blogs_android_binder_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_binder_cover.png
    960w:  /assets/img/blog/blogs_android_binder_cover.png
    480w:  /assets/img/blog/blogs_android_binder_cover.png
accent_image: /assets/img/blog/blogs_android_binder_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Android Binder机制简要通信原理
Android Binder 机制是 Android 系统中非常核心的 **进程间通信（IPC）机制**，它在 Android 各种组件之间的通信中扮演着至关重要的角色，比如 **App和系统服务之间** 的通信、**两个App之间**的通信等。理解 Binder 机制是深入学习 Android 系统的重要一步。

此前对于Binder的了解仅仅停留在**使用层面**，没有去了解其背后的原理，现对其更加深入地学习一下，本文将介绍Android Binder机制通信流程和背后的原理。
## 进程间通信需求由来
Android系统是基于Linux系统而开发的，也继承了Linux的 **进程隔离机制** 。进程之间是无法直接进行交互的，每个进程内部各自的数据无法直接访问。

操作系统层面上，系统为了防止上层进程直接操作系统资源，造成数据安全和系统安全问题，系统内核空间和用户空间也是分离开来的，保证用户程序进程崩溃时不会影响到整个系统。简单的说就是，分离后，内核空间（Kernel）是系统内核运行的空间，用户空间（UserSpace）是用户程序运行的空间。

两个用户空间的进程，要进行数据交互，也需要通过内核空间来驱动整个过程。进程间的通信，即 **Inter-Process Communication（IPC）**。

在 Linux 系统中，常见的 IPC 方式有**管道、消息队列、共享内存、信号量和Socket**等。然而，Android 并没有直接使用这些机制作为主要的 IPC 方式，而是选择了 Binder。这主要是出于以下几点考虑：

* **性能优化：** Binder 机制在设计上针对移动设备进行了优化，相比 Socket 等方式，其**数据拷贝次数更少，效率更高**。传统的 IPC 方式（如管道、消息队列）通常需要**两次内存复制**，而 **Binder只需要一次复制** （从用户空间写到内核空间，再复制到目标用户空间）。
* **安全性：** Binder 机制从底层提供 UID/PID 认证，可以方便地进行权限控制，确保通信的安全性。这对于 Android 这种多应用、多用户环境非常重要。
* **架构优势：** Binder 机制基于 C/S (Client/Server) 架构，使得服务提供者和使用者之间解耦，更容易进行系统设计和扩展。
* **内存管理：** Binder 机制在内核层实现了内存的映射和管理，能够更好地处理大块数据的传输。

## 内存映射
虽然用户地址空间是不能互相访问的，但是不同进程的内核地址空间是映射到相同物理地址的，它们是相同和共享的，我们可以借助内核地址空间作为中转站来实现进程间数据的传输。

![](/assets/img/blog/blogs_binder_mem_share.png)

进程B通过 `copy_from_user` 往内核地址空间里写值，进程A通过 `copy_to_user` 来获取这个值。可以看出为了共享这个数据，需要两个进程拷贝两次数据。

我们可以通过 mmap 将 **进程 A 的用户地址空间与内核地址空间进行映射** ，让他们指向相同的物理地址空间，这样就只需要进行一次拷贝。

![](/assets/img/blog/blogs_binder_mem_share_mmap.png)

当B进程调用一次 `copy_from_user` 时，会将数据从用户空间拷贝到内核空间，而进程A和这个内核空间映射了同一个物理地址，故进程A可以直接使用这个数据，而无需再次拷贝。

## Binder通信过程的四个主要角色
Server，服务的提供者，实现具体的业务逻辑。

Client，客户端， 服务的使用者，通过 `Binder` 代理对象与服务端进行通信。比如当我们获取WindowManager服务时，我们的APP就是客户端，当我们暴露通过AIDL暴露接口给其他应用去使用时，他们就是客户端，我们是提供服务的服务端。

`Service Manager` ，负责注册和查找服务，类似于 DNS 服务器。当**服务端启动**时，会向 `Service Manager` **注册**自己的 `Binder` 对象。在客户端需要查找服务时，则通过 `Service Manager` 获取对应服务的 Binder 代理对象。

`Binder` 驱动，这是整个 Binder 机制的核心，是 Linux 内核中的一个字符设备驱动程序。它负责完成进程间的数据传输和进程线程的调度。所有 Binder 通信都必须通过 Binder 驱动。

除了上述四个通信过程中的主要角色，还需要了解应用层建立Binder通信几个重要概念。


> **IBinder 接口：** 定义了 Binder 通信的基本接口，是 Binder 进程间通信的基石。所有可进行 Binder 通信的对象都必须实现 IBinder 接口。

> **Stub (本地 Binder 对象)：** 服务端实际业务逻辑的实现。它继承自 Binder 本地对象，并实现了 Stub 接口（通常是 AIDL 生成的接口）。当 Binder 驱动收到 Client 的请求时，会唤醒 Server 进程，并调用 Stub 对象的方法来处理请求。

> **Proxy (远程 Binder 代理对象)：** 客户端持有的服务端的 Binder 代理对象。它实现了 Stub 接口，但其内部并没有实际的业务逻辑。当客户端调用 Proxy 对象的方法时，Proxy 会将方法调用的信息（方法编号、参数等）打包，然后通过 Binder 驱动发送给服务端。

> **AIDL (Android Interface Definition Language)：** Android 接口定义语言。它用于定义进程间通信的接口，通过 AIDL 文件，编译器可以自动生成 Stub 和 Proxy 类，大大简化了 Binder 的使用。

## Service Manager介绍
**ServiceManager** 是 Android Binder 进程间通信（IPC）机制的核心组成部分之一。简单来说，它就像一个**服务注册中心**或**黄页**。当系统中的各种服务（例如 ActivityManagerService、PackageManagerService 等）启动时，它们会**将自己注册到 ServiceManager 中**。其他进程如果需要使用这些服务，就可以通过 ServiceManager **查询并获取** 到对应的 Binder 代理对象，进而与服务进行通信。

在 Android 早期版本中，ServiceManager 是一个独立的进程，但在现代 Android 版本中，它通常被集成在 `init` 进程中，作为 `servicemanager` 可执行文件运行。它监听一个固定的 Binder 端口（通常是 0），作为所有其他 Binder 通信的入口。
### Service Manager在系统中的角色
1.  **服务注册（Add Service）**:
      * 各种系统服务（通常是 C++ 或 Java 实现）在启动时，会通过 Binder 机制将自己的 Binder 对象注册到 ServiceManager 中，并给自己的服务起一个唯一的名称（例如 "activity" 对应 ActivityManagerService）。
      * 这个注册过程就是调用 ServiceManager 的 `addService()` 方法。

2.  **服务查询（Get Service）**:
      * 当客户端进程需要使用某个服务时，它不会直接知道服务的地址或句柄。
      * 客户端会向 ServiceManager 发送请求，通过服务的名称来查询对应的 Binder 对象。
      * ServiceManager 收到请求后，会返回该服务在 Binder 驱动中的句柄（handle），客户端就可以通过这个句柄与服务建立 Binder 通信。这个过程就是调用 ServiceManager 的 `getService()` 方法。

它为所有系统服务提供了一个统一的查找和访问入口，简化了服务的管理和调用。

在 Android 系统的启动过程中， `ServiceManager` 是最先启动的核心服务之一，因为它为后续其他关键服务的启动和交互提供了基础。
## Binder驱动介绍
**Binder 驱动** 是 Linux 内核中的一个字符设备驱动程序。它的主要作用是**实现并管理 Android 系统中的进程间通信（IPC）机制**。

Binder 驱动就像一个中间人或者一个**高性能的快递员**，负责在不同的进程之间安全、高效地传递数据和调用请求。所有基于 Binder 的 IPC 都是通过 Binder 驱动来完成的。

Binder 驱动在数据传输过程中会进行**权限验证**。它能够识别通信的双方进程 ID (PID) 和用户 ID (UID)，从而对通信双方进行身份校验，保证了 IPC 的安全性。

另外，Binder 机制是**面向对象**的。它允许进程间像调用本地对象方法一样调用远程进程中的服务方法，这使得 Android 服务的开发和使用更加符合面向对象的编程范式。客户端拿到的是一个远程服务的“代理对象”，通过这个代理对象调用方法时，Binder 会负责将请求转发到实际的服务进程。

在稳定性方面， Binder 驱动通过统一管理进程和线程，以及 Binder 内存池，提供了一个更加健壮和稳定的 IPC 框架，减少了死锁和资源泄露的风险。

Binder 驱动维护着一个内部的数据结构，来管理所有已注册的 Binder 对象、它们的句柄以及与进程和线程的对应关系。它提供了一个高效、安全且面向对象的通信机制。它的存在使得 Android 上的各个系统服务和应用程序组件能够无缝地进行协作。

Binder 是一个字符驱动，对应的设备文件是 /dev/binder，和其他驱动一样，是通过 Linux 的文件访问系统调用对外提供具体功能的：
* open()，用于打开 Binder 驱动，返回 Binder 驱动的文件描述符
* mmap()，用于在内核中申请一块内存，并完成应用层与内核层的虚拟地址映射
* ioctl，在应用层调用 ioctl ，从应用层向内核层发送数据或者读取内核层发送到应用层的数据


## Binder通信简化流程
通信架构示意图：

![](/assets/img/blog/blogs_binder_arch.png)

Binder 通信的流程可以分为以下几步：
1.  **注册服务：**
    * **Server 服务端**在启动时，会向 **Service Manager** 注册自己提供的服务及其对应的 Binder 对象。
    * 这个注册过程其实也是一次 Binder 通信，Server 作为 Client 向 Service Manager 发送注册请求。
2.  **获取服务：**
    * **Client 端**（例如：某个 App）需要使用某个服务时，会通过 **Service Manager** 查询对应的服务，获取其 Binder 代理对象。
    * 这个获取过程也是一次 Binder 通信，Client 作为 Client 向 Service Manager 发送查询请求。
3.  **Client 调用服务：**
    * Client 获取到服务的 Binder 代理对象后，就可以通过这个代理对象调用服务端的具体方法。
    * 当 Client 调用代理对象的方法时，实际是把请求数据打包（**marshalling**）并通过 **Binder 驱动** 发送给 Server 端。
    * Binder 驱动将数据从 Client 进程的用户空间拷贝到内核空间，然后根据目标进程的 Binder 对象，再将数据从内核空间映射到 Server 进程的用户空间。
    * Server 端收到请求后，会解包（**unmarshalling**）数据，然后执行对应的服务方法，并将结果返回给 Client。这个返回过程也是通过 Binder 驱动进行的。

一般来说，Client 进程访问 Server 进程函数，我们需要在 Client 进程中 **按照固定的规则打包数据** ，这些数据包含了：
* 数据发给哪个进程，Binder 中是一个整型变量 `Handle`
* 要调用目标进程中的那个函数，Binder 中用一个整型变量 `Code` 表示目标函数的参数
* 要执行具体什么操作，也就是 Binder 协议

Client 进程通过 IPC 机制将数据传输给 Server 进程。当 Server 进程收到数据，按照固定的格式解析出数据，调用函数，并使用相同的格式将函数的返回值传递给 Client 进程。

通信关系图：

![](/assets/img/blog/blogs_binder_relationship.jpg)

### binder_node
binder_node 是应用层的 **service 在内核中的存在形式** ，是内核中对应用层 service 的描述，在内核中具体表现为 binder_node 结构体。

在上图中，ServiceManager 在 Binder 驱动中有对应的的一个 binder_node(Binder 实体)。每个 Server 在 Binder 驱动中也有对应的的一个 binder_node(Binder 实体)。这里假设每个 Server 内部仅有一个 service，内核中就只有一个对应的 binder_node(Binder 实体)，实际可能存在多个。

binder_node 结构体中存在一个指针 struct binder_proc *proc;，指向 binder_node 对应的 binder_proc 结构体。

```cpp
// binder_node 是内核中对应用层 binder 服务的描述
struct binder_node {
	//......
	struct binder_proc *proc;
	//......
}
```

### binder_proc
binder_proc 是内核中对应用层进程的描述，内部有众多重要数据结构。

```cpp
// binder_proc 是内核中对应用层进程的描述
struct binder_proc {
	//......
	struct binder_context *context;
	//......
}
```

### binder_ref（Binder 引用）
所谓 binder_ref（Binder 引用），实际上是内核中 binder_ref 结构体的对象，它的作用是在表示 binder_node(Binder 实体) 的引用。

换句话说，每一个 binder_ref（Binder 引用）都是某一个 binder_node (Binder实体)的引用，通过 binder_ref（Binder 引用） 可以在内核中找到它对应的 binder_node(Binder 实体)。

### 寻址
Binder 是一个 RPC 框架，最少会涉及两个进程。那么就涉及到寻址问题，所谓寻址就是当 A 进程需要调用 B 进程中的函数时，怎么找到 B 进程。

Binder 中寻址分为两种情况：
* ServiceManager 寻址，即 Client/Server 怎么找到 ServiceManager，对应于内核，就是找到 ServiceManager 对应的 binder_proc 结构体实例
* Server 寻址，即 Client 怎么找到 Server，对应于内核，就是找到 Server 对应的 binder_proc 结构体实例

#### 如何找到ServiceManager
Service Manager 启动并注册自身
* 在 Android 系统启动初期，servicemanager 进程（一个守护进程）会首先启动。
* 它会打开 `/dev/binder` 设备文件，获取一个文件描述符 (fd)。
*然后，它会通过一个特殊的 ioctl 系统调用：BINDER_SET_CONTEXT_MGR，将自己注册为 Binder 驱动的“上下文管理器”。这个操作会告诉 Binder 驱动，Service Manager 是那个负责管理所有其他 Binder 服务的特殊实体。
* 在这个注册过程中，Binder 驱动会为 Service Manager 分配句柄 0。从此以后，**Service Manager 就成为了 Binder 驱动中唯一拥有句柄 0 的实体**。

每个使用 binder 的进程，在初始化时，会在内核中将 `binder_device` 的 context 成员赋值给 `binder_proc->context` 。

```cpp
binder_proc->context = binder_device->context;
```

`binder_device` 指的是 Binder 驱动程序在 Linux 内核中提供的设备文件，而 binder_device是全局唯一变量，这样的话，所有进程的 `binder_proc->context` 都指向同一个结构体实例。

当 `ServiceManager` 调用 `binder_become_context_manager` 后，会陷入内核，在内核中会构建一个 `binder_node` 结构体实例，构建好以后，会将他保存到 `binder_proc->context->binder_context_mgr_node` 中。

也就是说，任何时候我们都可以通过 `binder_proc->context->binder_context_mgr_node` 获得 `ServiceManager` 对应的 `binder_node` 结构体实例。 `binder_node` 结构体中有一个成员 `struct binder_proc *proc;`，通过这个成员我们就能找到 `ServiceManager` 对应的 `binder_proc`.

当任何其他 Binder 进程（Client 或 Server，在向 Service Manager 注册服务时，它也充当 Service Manager 的 Client）需要与 Service Manager 通信时，它们并不需要“查找” Service Manager。

它们可以直接通过 Binder 框架提供的 API **获取一个代表 Service Manager 的 IBinder 代理对象**。 这个代理对象内部封装的 Binder 句柄值就是 0。
#### 如何找到Server
服务注册阶段
* Server 端向 ServiceManager 发起注册服务请求时(svcmgr_publish)，会陷入内核，首先通过 ServiceManager 寻址方式找到 ServiceManager 对应的 binder_proc 结构体，然后在内核中**构建一个**代表待注册服务的 binder_node 结构体**实例**，并**插入服务端对应的 binder_proc->nodes 红黑树中**。
* 接着会构建一个 binder_ref 结构体，binder_ref 会**引用**到上一阶段构建的 **binder_node** ，并插入到 ServiceManager 对应的 binder_proc->refs_by_desc 红黑树中，同时会计算出一个 desc 值（1，2，3 ....依次赋值）保存在 binder_ref 中。
* 最后服务相关的信息（主要是名字和 desc 值）会传递给 ServiceManager 应用层，应用层通过一个链表将这些信息保存起来

服务获取阶段
* Client 端向 `ServiceManager` 发起获取服务请求时(svcmgr_lookup，请求的数据中包含服务的名字)，会陷入内核， 通过 `binder_proc->context->binder_context_mgr_node` 寻址到 ServiceManager，接着通过分配映射内存，拷贝数据后，将"获取服务请求"的数据发送给 ServiceManager， **ServiceManager** 应用层收到数据后，会**遍历内部的链表**，通过传递过来的 name 参数，找到对应的 handle，然后将数据返回给 Client 端，接着陷入内核，通过 handle 值在 ServiceManager 对应的 `binder_proc->refs_by_desc` 红黑树中**查找到服务对应 binder_ref**，接着通过 binder_ref 内部指针找到服务对应的 binder_node 结构。
* 接着会**创建出一个新的 binder_ref 结构体**实例，内部 node 指针指向刚刚找到的服务端的 binder_node，接着再将 binder_ref 插入到 Client 端的 `binder_proc->refs_by_desc`，并计算出一个 desc 值（1，2，3 ....依次赋值），保存到 binder_ref 中。desc 值也会返回给 Client 的应用层。
* Client 应用层收到内核返回的这个 desc 值，改名为 `handle` ，接着向 Server 发起远程调用，远程调用的数据包中包含有 handle 值，接着陷入内核，在内核中首先根据 handle 值在 Client 的 `binder_proc->refs_by_desc` 获取到 binder_ref，通过 binder_ref 内部 node 指针找到目标服务对应的 binder_node，然后通过 binder_node 内部的 proc 指针找到目标进程的 binder_proc，这样就完成了整个寻址过程。
### Binder机制的实现原理
从更深层次看，Binder 机制主要依赖以下几点实现：
1.  **内存映射 (mmap)：** Binder 驱动在内核空间和每个使用 Binder 的进程的用户空间之间建立了一块共享内存区域。当 Client 发送数据时，数据首先从 Client 进程的用户空间拷贝到这块共享内存的内核缓冲区。然后，Binder 驱动会修改 Server 进程的页表，将这块内核缓冲区映射到 Server 进程的用户空间，从而实现了数据的一次拷贝。
2.  **进程通信协议：** Binder 驱动定义了一套自己的通信协议，Client 和 Server 之间的数据传输都是按照这个协议进行的。这个协议会包含事务码 (transaction code)、数据大小、数据内容等信息。
3.  **线程管理：** Binder 驱动在内部维护了一个线程池，用于处理来自不同进程的 Binder 请求。当一个 Binder 请求到达时，如果当前没有空闲线程，Binder 驱动会创建新的线程来处理请求，从而保证并发性。
4.  **引用计数：** Binder 驱动为每个 Binder 对象维护了引用计数。当一个 Binder 对象被创建或被 Client 引用时，引用计数会增加；当 Client 释放对 Binder 对象的引用时，引用计数会减少。当引用计数为零时，Binder 驱动会自动回收该 Binder 对象。
5.  **驱动调度：** 当 Client 发送 Binder 请求时，Binder 驱动会将请求加入到等待队列中。当 Server 有空闲线程时，Binder 驱动会从等待队列中取出请求并唤醒 Server 线程进行处理。

## oneway机制
简单来说，oneway 是一种异步调用机制。当你通过Binder调用远程进程中的方法时，如果该方法被标记为 oneway，那么：
* 调用者（Client）会立即返回，而不会等待被调用者（Server）执行完方法并返回结果。 调用请求会被发送到远程进程，但调用者线程不会被阻塞。
* 被调用者（Server）会在一个单独的Binder线程池中处理这个 oneway 请求。 这意味着 oneway 调用不会阻塞服务端的UI线程或其他重要线程。

oneway 方法不能有返回值。 因为调用者不会等待结果，所以也就没有返回值的意义。如果需要执行结果，可以设置一个callback。

在声明AIDL接口时，将返回值类型设置为oneway既可。

```java
// IMyService.aidl
package com.example.myapp;

interface IMyService {
    // 同步方法，会等待返回值
    int add(int a, int b);

    // oneway 方法，客户端不会等待其执行完成
    oneway void doSomethingAsync(String message);

    // 如果整个接口都是oneway的，也可以直接在接口前面声明
    // oneway interface IAnotherService {
    //     void notifyEvent(int eventCode);
    // }
}
```

## Binder 通信大小限制
Binder 调用中同步调用优先级大于 oneway（异步）的调用，为了充分满足同步调用的内存需要，所以将 oneway 调用的内存限制到申请内存上限的一半。

![](/assets/img/blog/blogs_binder_max_memory.png)

Android系统 中大部分IPC场景是使用 Binder 作为通信方式，在进程创建的时候会为 Binder 创建一个1M 左右的缓冲区用于跨进程通信时的数据传输，如果超过这个上限就就会抛出这个异常，而且这个缓存区时当前进程内的所有线程共享的，线程最大数量为16（线程池 15 个子线程 + 1 个主线程）个，如同时间内总传输大小超过了1M，也会抛异常。

另外在Activity启动的场景比较特殊，因为Binder 的通信方式为两种，一种是异步通信，一种是同步通信，异步通信时数据缓冲区大小被设置为了原本的1半。