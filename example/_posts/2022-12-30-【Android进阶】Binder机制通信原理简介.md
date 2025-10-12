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
# 【Android进阶】Binder机制通信原理简介
Android Binder 机制是 Android 系统中非常核心的 **进程间通信（IPC）机制**，它在 Android 各种组件之间的通信中扮演着至关重要的角色，比如 **App和系统服务之间** 的通信、**两个App之间**的通信等。理解 Binder 机制是深入学习 Android 系统的重要一步。

此前对于Binder的了解仅仅停留在**使用层面**，没有去了解其背后的原理，现对其更加深入地学习一下，本文将介绍Android Binder机制通信流程和背后的原理。
## 进程间通信需求由来
Android系统是基于Linux系统而开发的，也继承了Linux的 **进程隔离机制** 。进程之间是无法直接进行交互的，每个进程内部各自的数据无法直接访问。

操作系统层面上，系统为了防止上层进程直接操作系统资源，造成数据安全和系统安全问题，系统内核空间和用户空间也是分离开来的，保证用户程序进程崩溃时不会影响到整个系统。简单的说就是，分离后，内核空间（Kernel）是系统内核运行的空间，用户空间（UserSpace）是用户程序运行的空间。

两个用户空间的进程，要进行数据交互，也需要通过内核空间来驱动整个过程。进程间的通信，即 **Inter-Process Communication（IPC）**。

在 Linux 系统中，常见的 IPC 方式有**管道、消息队列、共享内存、信号量和Socket**等。然而，Android 并没有直接使用这些机制作为主要的 IPC 方式，而是选择了 Binder。这主要是出于以下几点考虑：

* **性能优化：** Binder 机制在设计上针对移动设备进行了优化，相比 Socket 等方式，其**数据拷贝次数更少，效率更高**。传统的 IPC 方式（如管道、消息队列）通常需要**两次内存复制**，而 **Binder只需要一次复制** （从用户空间写到内核空间，目标用户空间可以直接通过内核缓冲区读取）。
* **安全性：** Binder 机制从底层提供 UID/PID 认证，可以方便地进行权限控制，确保通信的安全性。这对于 Android 这种多应用、多用户环境非常重要。
* **架构优势：** Binder 机制基于 C/S (Client/Server) 架构，使得服务提供者和使用者之间解耦，更容易进行系统设计和扩展。
* **内存管理：** Binder 机制在内核层实现了内存的映射和管理，能够更好地处理大块数据的传输。

## 内存映射
虽然用户地址空间是不能互相访问的，但是**不同进程的内核地址空间是映射到相同物理地址**的，它们是相同和共享的，我们可以借助内核地址空间作为中转站来实现进程间数据的传输。

![](/assets/img/blog/blogs_binder_mem_share.png)

进程 B 通过 `copy_from_user` 往内核地址空间里写值，进程 A 通过 `copy_to_user` 来获取这个值。可以看出为了共享这个数据，需要两个进程拷贝两次数据。

我们可以通过 mmap 将 **进程 A 的用户地址空间与内核地址空间进行映射** ，让他们指向相同的物理地址空间，这样就只需要进行一次拷贝。

![](/assets/img/blog/blogs_binder_mem_share_mmap.png)

当进程 B 调用一次 `copy_from_user` 时，会将数据从用户空间拷贝到内核空间，而进程 A 和这个内核空间映射了同一个物理地址，故进程 A 可以直接使用这个数据，而无需再次拷贝。

### Binder通信的内存映射机制
每个使用 `Binder` 的进程在启动时，会通过 `mmap()` 系统调用，向 `Binder` 驱动申请一块内存映射区域。驱动响应这个调用， **在内核空间分配一块物理页面，同时将其映射到该进程的用户空间和内核空间** 。这样，每个进程都有自己对应的 `Binder` 映射缓冲区。

这块内核缓冲区会被 `mmap()` 映射到该进程的用户空间地址。

通信开始时，客户端进程将要发送的数据（Parcel）写入其用户空间中映射的这块共享内存区域（实际是写入到 Binder 驱动分配的内核缓冲区）。数据从客户端的用户空间拷贝到Binder 驱动的内核缓冲区，这是第一次拷贝。

由于数据已经在内核缓冲区中。Binder 驱动并不需要将数据从内核缓冲区再次拷贝到目标进程的用户空间。驱动程序通过 **页表机制（Page Table）** 的调整，将内核缓冲区中的数据直接映射到服务端进程的用户空间。

服务端返回数据时同理。
## Binder通信过程的四个主要角色
* Server，服务的提供者，实现具体的业务逻辑。
* Client，客户端， 服务的使用者。
* `Service Manager` ，负责注册和查找服务。当**服务端启动**时，会向 `Service Manager` **注册**自己的 `Binder` 对象。在客户端需要查找服务时，则通过 `Service Manager` 获取对应服务的 Binder 代理对象。
* `Binder` 驱动，这是整个 Binder 机制的核心，它是 Linux 内核中的一个字符设备驱动程序。它负责完成进程间的数据传输和进程线程的调度。所有 Binder 通信都必须通过 Binder 驱动。

### Service Manager介绍
**ServiceManager** 是 Android Binder 进程间通信（IPC）机制的核心组成部分之一。简单来说，它就像一个**服务注册中心**或**黄页**。当系统中的各种服务（例如 ActivityManagerService、PackageManagerService 等）启动时，它们会**将自己注册到 ServiceManager 中**。其他进程如果需要使用这些服务，就可以通过 ServiceManager **查询并获取** 到对应的 Binder 代理对象，进而与服务进行通信。

在 Android 早期版本中，ServiceManager 是一个独立的进程，但在现代 Android 版本中，它通常被集成在 `init` 进程中，作为 `servicemanager` 可执行文件运行。它监听一个固定的 Binder 端口（通常是 0），作为所有其他 Binder 通信的入口。

它为所有系统服务提供了一个统一的查找和访问入口，简化了服务的管理和调用。在 Android 系统的启动过程中， `ServiceManager` 是最先启动的核心服务之一，因为它为后续其他关键服务的启动和交互提供了基础。
### Binder驱动介绍
Binder 驱动在通信过程中主要扮演了以下几个关键角色：

第一点就是刚刚提到的 **分配共享缓冲区** 。当任何一个用户进程（客户端或服务端）第一次打开 Binder 设备时，Binder 驱动会通过 `mmap()` 系统调用，为这个进程在内核中分配一块**连续的物理内存作为共享缓冲区**。

驱动程序会将这块内核缓冲区分别映射到该进程的**用户空间**。这样，数据在客户端和驱动之间、以及驱动和服务端之间传输时，就可以通过**共享内存**机制实现高效的传输。

第二点为**路由和转发**，驱动程序维护着所有使用 Binder 的进程、这些进程中的 Binder 线程池、以及所有 Binder 实体（服务对象）的**内部映射表**。

当客户端通过 `ioctl()` 系统调用向驱动发送请求时，它会携带一个表示目标服务对象的**句柄 (Handle)**。驱动程序根据这个句柄，查询内部映射表，精确地找到对应的**目标服务端进程**和目标 **Binder 实体（Stub）**，然后将请求数据转发给该进程。

第三是**线程管理和调度**，每个服务端进程都会向 Binder 驱动“注册”一组专用于处理 IPC 请求的线程（即 Binder 线程池）。当驱动程序接收到发往某个服务端的请求时，它会唤醒服务端进程中**空闲的** Binder 线程，让它去处理这个请求。如果所有 Binder 线程都在忙碌，驱动可能会指示服务端**创建**一个新的 Binder 线程来处理请求（直到达到最大线程数限制）。驱动确保每个传入的请求都能被服务端的某个 Binder 线程**排队和处理**。

最后一点为**安全管理**，驱动程序在处理事务时，会记录并传递**调用进程的 UID/PID** 等身份信息。这为上层（如 Android Framework）进行权限检查（例如，判断调用者是否有权限调用某个服务）提供了底层信任的基础数据。
### 两个关键自动生成类 Stub 和 Proxy
aidl接口声明示例文件：

```java
package com.stephen.commondebugdemo;

interface ICalculateTest {
    void setRemoteValue(int value);
    int getRemoteValue();
}
```

在定义完了AIDL文件，确定好方法之后，编译器会根据AIDL文件自动生成 Stub 和 Proxy 类。
#### Stub类
`Stub` 类是 AIDL 封装机制中的**服务端核心**，它承担了将底层 IPC 机制与上层业务逻辑连接起来的全部工作。它是一个**抽象的内部类**，其完整声明通常是：

```java
public static abstract class Stub extends android.os.Binder implements com.stephen.commondebugdemo.ICalculateTest
```

可以看到，`Stub` 类继承自 `android.os.Binder`，这赋予了它作为 **Binder 实体对象** 的能力。当服务进程将 `Stub` 的实现对象注册给 `ServiceManager` 后，它就代表了服务端的具体功能。当客户端发起 IPC 调用时，请求会直接被路由到这个 `Binder` 对象上。

实现预定义的 `CalculateTest` 接口，要求所有继承 `Stub` 的具体服务类必须实现 AIDL 接口中定义的所有业务方法。

##### **Stub 核心工作：处理客户端请求（`onTransact` 方法）**

`Stub` 类最重要、最复杂的工作就是**重写了 `Binder` 类的 `onTransact()` 方法**。这是所有远程调用请求进入服务端的唯一入口。

```java
/**
 * 处理客户端请求的核心方法。
 * @Param code 客户端请求的方法标识符
 * @Param data 包含客户端传递过来的方法参数的 `Parcel` 对象。
 * @Param reply 用于将服务端方法返回值或异常信息打包回客户端的 `Parcel` 对象。
 * @Param flags 用于控制 IPC 调用行为的标志位（例如 `FLAG_ONEWAY` 表示异步调用）。
 * 
*/
@Override public boolean onTransact(
    int code,
    android.os.Parcel data, 
    android.os.Parcel reply,
    int flags) throws android.os.RemoteException
```

详细代码如下：

```java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    java.lang.String descriptor = DESCRIPTOR;
    // 检查调用是否来自预期的接口
    if (code >= android.os.IBinder.FIRST_CALL_TRANSACTION && code <= android.os.IBinder.LAST_CALL_TRANSACTION) {
        data.enforceInterface(descriptor);
    }
    // 处理特殊的接口查询事务
    if (code == INTERFACE_TRANSACTION) {
        reply.writeString(descriptor);
        return true;
    }
    // 处理具体的业务方法调用
    switch (code) {
        case TRANSACTION_setRemoteValue: {
            int _arg0;
            // 从 data 中读取客户端传递的参数 _arg0
            _arg0 = data.readInt();
            // 调用服务端内部的 setRemoteValue 方法，将客户端传递的参数 _arg0 设置为服务端的状态
            this.setRemoteValue(_arg0);
            // 写入无异常标志位，通知客户端调用成功
            reply.writeNoException();
            break;
        }
        case TRANSACTION_getRemoteValue: {
            // 获取数据写入到 reply 中
            int _result = this.getRemoteValue();
            reply.writeNoException();
            reply.writeInt(_result);
            break;
        }
        default: {
            return super.onTransact(code, data, reply, flags);
        }
    }
    return true;
}
```

`onTransact` 方法的工作流如下：

1.  **方法识别 (`code`)：** 根据传入的 `code`（一个整数标识符，对应 AIDL 中的方法），判断客户端想调用哪个方法。
2.  **数据解包 (`data`)：** 从输入的 `Parcel` 对象 (`data`) 中，按照约定的顺序和类型，逐一读取客户端传递过来的方法参数。
3.  **调用业务逻辑：** 调用开发者在 `Stub` 子类中实现的对应方法（例如 `this.methodA(...)`）。
4.  **结果打包 (`reply`)：** 将业务方法返回的结果（以及可能的异常信息）写入到输出的 `Parcel` 对象 (`reply`) 中。
5.  **返回结果：** `reply` 对象通过 Binder 驱动回传给客户端进程。

> Parcel 是 Android 工程师为了解决 IPC 效率问题而专门设计的一种**序列化容器**。它牺牲了通用性，换来了极高的性能，成为 Android Binder 机制中不可或缺的基石。在 AIDL 中，Proxy 和 Stub 所做的大量工作，就是围绕 Parcel 的读写来进行的。

##### **`asInterface()` 连接客户端与服务（`asInterface` 静态方法）**

```java
/**
 * Cast an IBinder object into an com.stephen.commondebugdemo.ICalculateTest interface,
 * generating a proxy if needed.
 */
public static com.stephen.commondebugdemo.ICalculateTest asInterface(android.os.IBinder obj)
{
    if ((obj==null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin!=null)&&(iin instanceof com.stephen.commondebugdemo.ICalculateTest))) {
        return ((com.stephen.commondebugdemo.ICalculateTest)iin);
    }
    return new com.stephen.commondebugdemo.ICalculateTest.Stub.Proxy(obj);
}
```

这个方法是客户端获取服务端代理对象的**桥梁**。客户端通过 `ServiceConnection` 的 `onServiceConnected()` 拿到一个原始的 `IBinder` 对象。

如果传入的 `obj` 是一个 `Stub` 实例（即客户端和服务端在同一个进程），则直接返回这个 `Stub` 实例本身。

远程调用 (跨进程)，如果传入的 `obj` 来自另一个进程，它会**创建并返回一个 `Proxy` 代理对象**，将这个远程 `IBinder` 封装起来。

这样，客户端调用者无需关心目标对象是否在本地，只需调用 `asInterface()` 即可得到一个统一的 `IMyAidlInterface` 接口对象。

##### 服务端使用Stub的示例代码

```kotlin
class CalculateTestService : Service() {

    private var internalTestValue = 0

    private val binder = object : CalculateTest.Stub() {
        override fun setRemoteValue(value: Int) {
            internalTestValue = value
        }

        override fun getRemoteValue(): Int {
            return internalTestValue
        }
    }

    override fun onBind(intent: Intent) = binder
}
```

#### Proxy类
`Proxy` 类是 AIDL 封装机制中的**客户端核心**，它负责将客户端对接口方法的调用，转化为底层 Binder 机制所需的 IPC 请求。

它通常是 `Stub` 类内部的一个静态私有子类，其结构通常是下面这样，实现了所定义的AIDL服务接口。

```java
private static class Proxy implements com.stephen.commondebugdemo.ICalculateTest {
    private android.os.IBinder mRemote;

    Proxy(android.os.IBinder remote) {
        mRemote = remote;
    }

    @Override
    public android.os.IBinder asBinder() {
        return mRemote;
    }

    public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
    }

    ...
}
```

它的核心工作可以分为以下几个方面：

##### 结构与初始化：持有远程引用
实现 `ICalculateTest` 接口，这使得 `Proxy` 实例可以被客户端代码当作真正的 AIDL 接口对象来调用，从而实现了**透明代理**。

`Proxy` 持有 `IBinder` 对象 (`mRemote`字段)。这个 `IBinder` 对象是客户端通过 `ServiceConnection` 类，在服务连接成功后，调用 `asInterface()` 传入构造函数的。它代表了服务端的那个真正的 `Stub` 实体在客户端进程中的**引用（Binder Token）**。所有的 IPC 通信都是通过这个 `mRemote` 对象进行的。

#### 核心工作：发起远程调用（`transact` 方法）
`Proxy` 类实现了 AIDL 接口中定义的所有方法，但这些实现方法的内部是**统一的 IPC 封装逻辑**，直接调用 `transact()` 方法。

例如 `setRemoteValue()` ：

```java
@Override
public void setRemoteValue(int value) throws android.os.RemoteException {
    // 后面将方法参数写入 _data 中
    android.os.Parcel _data = android.os.Parcel.obtain();
    // 从reply中读取异常信息和返回值
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeInt(value);
        boolean _status = mRemote.transact(Stub.TRANSACTION_setRemoteValue, _data, _reply, 0);
        _reply.readException();
    } finally {
        // 回收 Parcel 对象
        _reply.recycle();
        _data.recycle();
    }
}
```

各实现方法简单来说就是数据的打包和发送。等待服务端处理完毕后，从返回的 `Parcel` 中读取服务端的执行结果（返回值或异常信息）。
##### 辅助验证方法 getInterfaceDescriptor
**`getInterfaceDescriptor()`** 提供了 AIDL 接口的唯一标识字符串，用于在通信双方进行接口校验。
##### 客户端获取AIDL服务实例代码

```kotlin
object ClientProxy {

    private lateinit var calculateTest: ICalculateTest

    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(
            name: ComponentName?,
            service: IBinder?
        ) {
            calculateTest = ICalculateTest.Stub.asInterface(service)
        }

        override fun onServiceDisconnected(name: ComponentName?) {
            infoLog("onServiceDisconnected")
        }
    }

    ...

    // connect and call methods
}
```

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
`binder_node` 是应用层的 **service 在内核中的存在形式** ，是内核中对应用层 service 的描述，在内核中具体表现为 `binder_node` 结构体。

在上图中，ServiceManager 在 Binder 驱动中有对应的的一个 **binder_node(Binder 实体)**。

每个 `Server` 在 Binder 驱动中也有对应的的一个 **binder_node(Binder 实体)**。

这里假设每个 Server 内部仅有一个 service，内核中就只有一个对应的 binder_node(Binder 实体)，实际可能存在多个。

`binder_node` 结构体中存在一个指针 `struct binder_proc *proc;` ，指向 `binder_node` 对应的 `binder_proc` 结构体。

```cpp
// binder_node 是内核中对应用层 binder 服务的描述
struct binder_node {
	//......
	struct binder_proc *proc;
	//......
}
```

### binder_proc
`binder_proc` 是内核中对**应用层进程**的描述，内部有众多重要数据结构。

```cpp
// binder_proc 是内核中对应用层进程的描述
struct binder_proc {
	//......
	struct binder_context *context;
	//......
}
```

### binder_ref（Binder 引用）
所谓 **binder_ref（Binder 引用）**，实际上是内核中 `binder_ref` 结构体的对象，它的作用是在表示 binder_node(Binder 实体) 的引用。

换句话说，每一个 **binder_ref（Binder 引用）** 都是某一个 **binder_node (Binder实体)的引用** ，通过 binder_ref（Binder 引用） 可以在内核中找到它对应的 binder_node(Binder 实体)。

### 寻址
Binder 是一个 RPC 框架，最少会涉及两个进程。那么就涉及到寻址问题，所谓寻址就是当 A 进程需要调用 B 进程中的函数时，怎么找到 B 进程。

Binder 中寻址分为两种情况：
* ServiceManager 寻址，即 Client/Server 怎么找到 ServiceManager，对应于内核，就是找到 ServiceManager 对应的 `binder_proc` 结构体实例
* Server 寻址，即 Client 怎么找到 Server，对应于内核，就是找到 Server 对应的 `binder_proc` 结构体实例

#### 如何找到ServiceManager
系统启动时，Service Manager 启动并注册自身:

* 在 Android 系统启动初期，servicemanager 进程（一个守护进程）会首先启动。
* 它会打开 `/dev/binder` 设备文件，获取一个文件描述符 (fd)。
*然后，它会通过一个特殊的 ioctl 系统调用 `BINDER_SET_CONTEXT_MGR` 将自己注册为 Binder 驱动的 **“上下文管理器”** 。这个操作会告诉 Binder 驱动，Service Manager 是那个负责管理所有其他 Binder 服务的特殊实体。
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
##### **服务注册阶段**
Server 端向 ServiceManager 发起**注册服务请求**时(svcmgr_publish)，会陷入内核，首先通过 ServiceManager 寻址方式找到 ServiceManager 对应的 binder_proc 结构体，然后在内核中构建一个代表待注册服务的 `binder_node` 结构体实例，并**插入服务端对应的 binder_proc->nodes 红黑树中**。

接着会**构建一个 binder_ref 结构体** ， `binder_ref` 会引用到上一阶段构建的 **binder_node** ，并插入到 `ServiceManager` 对应的 **binder_proc->refs_by_desc 红黑树** 中，同时会计算出一个 desc 值（1，2，3 ....依次赋值）保存在 `binder_ref` 中。

最后服务相关的信息（主要是名字和 desc 值）会传递给 ServiceManager 应用层，应用层通过一个链表将这些信息保存起来。
##### **服务获取阶段**
`Client` 端向 `ServiceManager` 发起获取服务请求时(svcmgr_lookup，请求的数据中包含服务的名字)，会陷入内核， 通过 `binder_proc->context->binder_context_mgr_node` 寻址到 ServiceManager，接着通过分配映射内存，拷贝数据后，将 **"获取服务请求"** 的数据发送给 `ServiceManager` 。

**ServiceManager** 应用层收到数据后，会**遍历内部的链表**，通过传递过来的 name 参数，找到对应的 handle，然后将数据返回给 Client 端，接着陷入内核，通过 handle 值在 `ServiceManager` 对应的 `binder_proc->refs_by_desc` 红黑树中**查找到服务对应 binder_ref**，接着通过 `binder_ref` 内部指针找到服务对应的 `binder_node` 结构。

接着 **创建出一个新的 `binder_ref` 结构体**实例，内部 node 指针指向刚刚找到的服务端的 binder_node，接着再 **将 binder_ref 插入到 Client 端的 `binder_proc->refs_by_desc`** ，并计算出一个 desc 值（1，2，3 ....依次赋值），保存到 binder_ref 中。desc 值也会返回给 Client 的应用层。

Client 应用层收到内核返回的这个 desc 值，改名为 `handle` ，接着向 Server 发起远程调用，远程调用的数据包中包含有 handle 值，接着陷入内核，在内核中首先根据 `handle` 值在 Client 的 `binder_proc->refs_by_desc` 获取到 `binder_ref` ，通过 `binder_ref` 内部 node 指针找到目标服务对应的 binder_node，然后通过 `binder_node` 内部的 proc 指针找到目标进程的 `binder_proc` ，这样就完成了整个寻址过程。

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

Android系统 中大部分IPC场景是使用 `Binder` 作为通信方式，在进程创建的时候会为 `Binder` 创建一个1M 左右的缓冲区用于跨进程通信时的数据传输，如果超过这个上限就就会抛出这个异常，而且这个缓存区时当前进程内的所有线程共享的，线程最大数量为16（线程池 15 个子线程 + 1 个主线程）个，如同时间内总传输大小超过了1M，也会抛异常。

另外在 `Activity` 启动的场景比较特殊，因为 `Binder` 的通信方式为两种，一种是异步通信，一种是同步通信，异步通信时数据缓冲区大小被设置为了原本的1半。