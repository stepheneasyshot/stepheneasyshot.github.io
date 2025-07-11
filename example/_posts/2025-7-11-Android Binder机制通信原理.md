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
# Android Binder机制通信原理
Android Binder 机制是 Android 系统中非常核心的 **进程间通信（IPC）机制**，它在 Android 各种组件之间的通信中扮演着至关重要的角色，比如 **App和系统服务之间** 的通信、**两个App之间**的通信等。理解 Binder 机制是深入学习 Android 系统的重要一步。

此前对于Binder的了解仅仅停留在**使用层面**，没有去了解其背后的原理，只知道在底层是通过内存映射实现的。现对其更加深入地学习一下，本文将介绍Android Binder机制通信流程和背后的原理，并对比下其他IPC通信之间的区别。

## 进程间通信需求由来
Android系统是基于Linux系统而开发的，也继承了Linux的 **进程隔离机制** 。进程之间是无法直接进行交互的，每个进程内部各自的数据无法直接访问。

操作系统层面上，系统为了防止上层进程直接操作系统资源，造成数据安全和系统安全问题，系统内核空间和用户空间也是分离开来的，保证用户程序进程崩溃时不会影响到整个系统。简单的说就是，分离后，内核空间（Kernel）是系统内核运行的空间，用户空间（UserSpace）是用户程序运行的空间。

两个用户空间的进程，要进行数据交互，也需要通过内核空间来驱动整个过程。进程间的通信，即 **Inter-Process Communication（IPC）**。

在 Linux 系统中，常见的 IPC 方式有**管道、消息队列、共享内存、信号量和Socket**等。然而，Android 并没有直接使用这些机制作为主要的 IPC 方式，而是选择了 Binder。这主要是出于以下几点考虑：

* **性能优越：** Binder 机制在设计上针对移动设备进行了优化，相比 Socket 等方式，其**数据拷贝次数更少，效率更高**。传统的 IPC 方式（如管道、消息队列）通常需要**两次内存复制**，而 **Binder只需要一次复制** （从用户空间写到内核空间，再复制到目标用户空间）。
* **安全性：** Binder 机制从底层提供 UID/PID 认证，可以方便地进行权限控制，确保通信的安全性。这对于 Android 这种多应用、多用户环境非常重要。
* **架构优势：** Binder 机制基于 C/S (Client/Server) 架构，使得服务提供者和使用者之间解耦，更容易进行系统设计和扩展。
* **内存管理：** Binder 机制在内核层实现了内存的映射和管理，能够更好地处理大块数据的传输。

## Binder通信过程的四个主要角色
1.  **Client (客户端)：** 服务的使用者，通过 Binder 代理对象与服务端进行通信。
2.  **Server (服务端)：** 服务的提供者，实现具体的业务逻辑。
3.  **Service Manager (服务管理者)：** 负责注册和查找服务，类似于 DNS 服务器。当服务端启动时，会向 Service Manager 注册自己的 Binder 对象。客户端需要查找服务时，则通过 Service Manager 获取对应服务的 Binder 代理对象。
4.  **Binder 驱动：** Binder 机制的核心，是 Linux 内核中的一个字符设备驱动程序。它负责完成进程间的数据传输和进程线程的调度。所有 Binder 通信都必须通过 Binder 驱动。

除了上述四个主要角色，还需要了解应用层建立Binder通信几个重要概念：
* **IBinder 接口：** 定义了 Binder 通信的基本接口，是 Binder 进程间通信的基石。所有可进行 Binder 通信的对象都必须实现 IBinder 接口。
* **Stub (本地 Binder 对象)：** 服务端实际业务逻辑的实现。它继承自 Binder 本地对象，并实现了 Stub 接口（通常是 AIDL 生成的接口）。当 Binder 驱动收到 Client 的请求时，会唤醒 Server 进程，并调用 Stub 对象的方法来处理请求。
* **Proxy (远程 Binder 代理对象)：** 客户端持有的服务端的 Binder 代理对象。它实现了 Stub 接口，但其内部并没有实际的业务逻辑。当客户端调用 Proxy 对象的方法时，Proxy 会将方法调用的信息（方法编号、参数等）打包，然后通过 Binder 驱动发送给服务端。
* **AIDL (Android Interface Definition Language)：** Android 接口定义语言。它用于定义进程间通信的接口，通过 AIDL 文件，编译器可以自动生成 Stub 和 Proxy 类，大大简化了 Binder 的使用。

## Binder通信流程
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

### Binder机制的实现原理
从更深层次看，Binder 机制主要依赖以下几点实现：
1.  **内存映射 (mmap)：** Binder 驱动在内核空间和每个使用 Binder 的进程的用户空间之间建立了一块共享内存区域。当 Client 发送数据时，数据首先从 Client 进程的用户空间拷贝到这块共享内存的内核缓冲区。然后，Binder 驱动会修改 Server 进程的页表，将这块内核缓冲区映射到 Server 进程的用户空间，从而实现了数据的一次拷贝。
2.  **进程通信协议：** Binder 驱动定义了一套自己的通信协议，Client 和 Server 之间的数据传输都是按照这个协议进行的。这个协议会包含事务码 (transaction code)、数据大小、数据内容等信息。
3.  **线程管理：** Binder 驱动在内部维护了一个线程池，用于处理来自不同进程的 Binder 请求。当一个 Binder 请求到达时，如果当前没有空闲线程，Binder 驱动会创建新的线程来处理请求，从而保证并发性。
4.  **引用计数：** Binder 驱动为每个 Binder 对象维护了引用计数。当一个 Binder 对象被创建或被 Client 引用时，引用计数会增加；当 Client 释放对 Binder 对象的引用时，引用计数会减少。当引用计数为零时，Binder 驱动会自动回收该 Binder 对象。
5.  **驱动调度：** 当 Client 发送 Binder 请求时，Binder 驱动会将请求加入到等待队列中。当 Server 有空闲线程时，Binder 驱动会从等待队列中取出请求并唤醒 Server 线程进行处理。
