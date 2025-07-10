---
layout: post
description: > 
  本文介绍了WebSocket协议相关内容，背景知识和协议规范
image: 
  path: /assets/img/blog/blogs_websocket_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_websocket_cover.png
    960w:  /assets/img/blog/blogs_websocket_cover.png
    480w:  /assets/img/blog/blogs_websocket_cover.png
accent_image: /assets/img/blog/blogs_websocket_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【网络】WebSocket通信基础
WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议。它在 Web 应用中提供了一种持久连接，允许客户端和服务器之间进行**实时、双向**的数据传输，而**无需像传统的 HTTP 请求那样每次通信都建立新的连接**。这使得 WebSocket 非常适合需要低延迟和高吞吐量的应用，如在线游戏、实时聊天、协作工具、股票行情更新和物联网数据传输等。

### 核心特性
1. **全双工通信**  
   客户端和服务器可以**同时主动发送数据**，无需等待请求-响应模式，适合实时交互场景（如聊天、游戏、股票行情推送）。

2. **低延迟与高效性**  
   - 建立连接后，**数据帧直接传输**，省去 HTTP 请求的头部开销（如 Cookie、User-Agent 等重复字段）。
   - 相比 HTTP 轮询（频繁建立/断开连接），WebSocket 维持长连接，减少延迟和资源消耗。

3. **基于 TCP 的可靠传输**  
   继承 TCP 的可靠性（数据有序、不丢失、自动重传），同时通过应用层协议实现消息边界管理。

### 为什么需要 WebSocket？
在 WebSocket 出现之前，Web 浏览器和服务器之间的实时通信主要依赖于以下几种技术：
* **轮询 (Polling)**: 客户端定时向服务器发送 HTTP 请求，询问是否有新的数据。这种方式效率低下，会产生大量不必要的请求，并增加服务器负载。
* **长轮询 (Long Polling)**: 客户端发送一个 HTTP 请求，服务器保持连接打开，直到有新数据可用或超时。数据发送后，连接关闭，客户端立即发起新的请求。这种方式比轮询有所改进，但仍然是单向的，并且每次数据传输后都需要重新建立连接，依然存在延迟和开销。
* **Comet (Streaming)**: 服务器长时间保持 HTTP 连接打开，并不断向客户端发送数据。客户端收到数据后，连接保持打开。虽然实现了单向实时，但仍然是基于 HTTP 请求-响应模式的变种，而且客户端难以向服务器发送实时数据。
* **Flash Sockets/Java Applets**: 这些技术需要浏览器插件，兼容性差，且逐渐被淘汰。

WebSocket 的出现正是为了解决这些问题，提供一种原生、高效的双向通信机制。

### **二、协议分层与握手过程**
#### **1. 握手阶段（HTTP 升级）**
WebSocket 连接的建立始于一个特殊的 **HTTP 握手**。客户端向服务器发送一个特殊的 HTTP 请求，**请求升级到 WebSocket 协议**。

**客户端请求示例:**

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub24gY2U=
Origin: http://example.com
Sec-WebSocket-Version: 13
```

**服务器响应示例:**

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9GUfnjUMzdzbCYg=
```

**握手过程的关键点:**

  * **`Upgrade: websocket` 和 `Connection: Upgrade`:** 这两个头部字段明确表示客户端请求将协议从 HTTP 升级到 WebSocket。
  * **`Sec-WebSocket-Key`:** 这是一个随机生成的 Base64 编码的字符串。客户端发送此键，服务器会将其与一个固定的 GUID ("258EAFA5-E914-47DA-95CA-C5AB0DC85B11") 拼接，然后计算 SHA-1 散列并进行 Base64 编码，生成 `Sec-WebSocket-Accept`。
  * **`Sec-WebSocket-Version`:** 指定客户端使用的 WebSocket 协议版本。目前最新的标准版本是 13。
  * **`Sec-WebSocket-Accept`:** 服务器的响应，证明服务器理解并同意升级请求。它的值是根据 `Sec-WebSocket-Key` 计算得出的，客户端会验证这个值，确保握手成功。

一旦握手成功，HTTP 连接就会“升级”为 WebSocket 连接，此后所有通信都将使用 WebSocket 协议定义的帧格式，不再是 HTTP 请求/响应模式。

#### 2\. 数据传输阶段 (Data Transfer)

握手成功后，客户端和服务器之间就可以通过这个持久的 TCP 连接进行双向、全双工的数据传输。数据以 **帧 (Frames)** 的形式发送，而不是像 HTTP 那样以完整的消息发送。

##### **数据帧格式**
WebSocket 数据帧每个帧包含以下字段（简化版）：

| 字段 | 作用 |
|------|------|
| **FIN** | 标识是否为消息的最后一帧（1 bit） |
| **Opcode** | 帧类型（4 bits）：如 `0x1` 文本、`0x2` 二进制、`0x8` 关闭连接等 |
| **Mask** | 是否对负载数据掩码（1 bit，客户端必须设为 1） |
| **Payload Length** | 负载长度（7/16/64 bits，动态扩展） |
| **Masking Key** | 掩码密钥（4 bytes，客户端生成） |
| **Payload Data** | 实际数据（可能被掩码处理） |

**掩码机制**：客户端发送的数据必须掩码，服务器发送的数据不能掩码，防止缓存代理干扰。

**WebSocket 帧的特点:**

  * **轻量级:** WebSocket 帧头非常小，通常只有几字节，这减少了协议开销。
  * **支持多种数据类型:** 可以传输文本数据 (UTF-8 编码) 和二进制数据。
  * **分片 (Fragmentation):** 大的数据可以被分成多个帧进行传输，接收方再将其重组。这对于处理大型文件或需要分批发送的数据非常有用。
  * **掩码 (Masking):** 从客户端发送到服务器的数据帧必须进行掩码处理，以防止中间代理服务器缓存响应（因为 WebSocket 帧可能看起来像有效的 HTTP 请求）。服务器发送给客户端的数据不需要掩码。
  * **心跳机制 (Ping/Pong):** WebSocket 内置了 `PING` 和 `PONG` 帧，用于客户端和服务器之间发送心跳包，检测连接是否仍然活跃，类似于 MQTT 的 Keep Alive 机制。如果一端发送 `PING` 帧，另一端必须回复 `PONG` 帧。

#### **3\. 连接生命周期管理**
1. **关闭连接**  
   任一方发送 `Opcode=0x8` 的帧，携带 2 字节的状态码（如 `1000` 正常关闭）。对方需回复相同帧确认关闭。

2. **心跳机制（Ping/Pong）**  
   - 客户端或服务器可发送 `Opcode=0x9` 的 Ping 帧探测连接活性。
   - 对方需回复 `Opcode=0xA` 的 Pong 帧，超时未响应则视为连接断开。

### 安全性
WebSocket 协议支持加密通信，即 **WebSocket Secure (WSS)**。WSS 连接通过 TLS/SSL 进行加密，类似于 HTTPS。这意味着数据在传输过程中是加密的，可以防止窃听和篡改。

  * **`ws://`:** 非加密的 WebSocket 连接。
  * **`wss://`:** 加密的 WebSocket 连接，基于 TLS/SSL。

### WebSocket 的应用场景
基于WebSocket上述工作特点，常见的应用领域有下面这些：

  * **实时聊天应用:** 即时消息的发送和接收。
  * **在线游戏:** 多人游戏的实时玩家状态同步和交互。
  * **股票行情、金融数据推送:** 实时股价、市场数据更新。
  * **协作工具:** 多个用户同时编辑文档、共享屏幕等。
  * **物联网 (IoT):** 设备与服务器之间的小型、频繁数据交换。
  * **实时通知和报警:** 服务器向客户端推送即时通知。
  * **在线白板/绘图应用:** 实时共享用户绘图操作。

### Android平台进行WebSocket连接开发
在 Android 端进行 WebSocket 开发非常常见，特别是在需要实时数据传输的应用中，比如聊天、实时通知或物联网仪表盘。我们可以基于okhttp库的功能来实现。

**OkHttp** 是 Square 公司开发的一个流行的 HTTP 和 WebSocket 客户端。它功能强大、易于使用，并且是 Android 开发中最推荐的网络库之一。

```gradle
dependencies {
    implementation 'com.squareup.okhttp3:okhttp:4.12.0' 
}
```

#### 基本开发步骤

1.  **创建 OkHttpClient 实例:**
    通常，你会在应用中创建一个单例 `OkHttpClient` 实例。

    ```java
    OkHttpClient client = new OkHttpClient();
    ```

2.  **创建 WebSocketRequest:**
    构建一个 `Request` 对象，指定 WebSocket 服务器的 URL。

    ```java
    String webSocketUrl = "ws://echo.websocket.org"; // 替换为你的 WebSocket 服务器地址
    Request request = new Request.Builder().url(webSocketUrl).build();
    ```

3.  **实现 WebSocketListener:**
    创建一个实现 `WebSocketListener` 接口的类，用于处理 WebSocket 事件（打开、关闭、接收消息、失败等）。

    ```java
    public class MyWebSocketListener extends WebSocketListener {

        private static final int NORMAL_CLOSURE_STATUS = 1000;

        @Override
        public void onOpen(@NonNull WebSocket webSocket, @NonNull Response response) {
            super.onOpen(webSocket, response);
            // 连接成功建立时调用
            System.out.println("WebSocket Connected!");
            // 连接成功后可以发送消息
            webSocket.send("Hello from Android!");
            // 也可以发送二进制数据
            // webSocket.send(ByteString.decodeHex("deadbeef"));
        }

        @Override
        public void onMessage(@NonNull WebSocket webSocket, @NonNull String text) {
            super.onMessage(webSocket, text);
            // 接收到文本消息时调用
            System.out.println("Receiving Text: " + text);
        }

        @Override
        public void onMessage(@NonNull WebSocket webSocket, @NonNull ByteString bytes) {
            super.onMessage(webSocket, bytes);
            // 接收到二进制消息时调用
            System.out.println("Receiving Bytes: " + bytes.hex());
        }

        @Override
        public void onClosing(@NonNull WebSocket webSocket, int code, @NonNull String reason) {
            super.onClosing(webSocket, code, reason);
            // WebSocket 即将关闭时调用
            System.out.println("Closing: " + code + " / " + reason);
            webSocket.close(NORMAL_CLOSURE_STATUS, null); // 告知服务器正常关闭
        }

        @Override
        public void onFailure(@NonNull WebSocket webSocket, @NonNull Throwable t, @Nullable Response response) {
            super.onFailure(webSocket, t, response);
            // 连接失败时调用，例如网络问题、握手失败等
            System.err.println("Error: " + t.getMessage());
            if (response != null) {
                System.err.println("Error Response: " + response.code() + " " + response.message());
            }
        }

        @Override
        public void onClosed(@NonNull WebSocket webSocket, int code, @NonNull String reason) {
            super.onClosed(webSocket, code, reason);
            // WebSocket 完全关闭时调用
            System.out.println("WebSocket Closed: " + code + " / " + reason);
        }
    }
    ```

4.  **建立 WebSocket 连接:**
    在你的 Activity 或 Service 中，创建 `MyWebSocketListener` 实例，并通过 `OkHttpClient` 发起连接。

    ```java
    public class MyActivity extends AppCompatActivity {

        private WebSocket webSocket;
        private OkHttpClient client;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            client = new OkHttpClient(); // 初始化一次，最好作为单例

            String webSocketUrl = "ws://echo.websocket.org"; // 公共测试 WebSocket 服务器
            Request request = new Request.Builder().url(webSocketUrl).build();
            MyWebSocketListener listener = new MyWebSocketListener();

            webSocket = client.newWebSocket(request, listener); // 发起连接

            // 示例：可以在连接建立后发送消息，但在实际应用中通常在 onOpen 回调中发送
            // webSocket.send("Hello World!");
        }

        @Override
        protected void onDestroy() {
            super.onDestroy();
            if (webSocket != null) {
                webSocket.close(MyWebSocketListener.NORMAL_CLOSURE_STATUS, "Goodbye!"); // 在 Activity 销毁时关闭连接
            }
            if (client != null) {
                client.dispatcher().executorService().shutdown(); // 关闭 OkHttpClient 的线程池
            }
        }

        // 可以添加一个方法来发送消息
        public void sendMessage(String message) {
            if (webSocket != null) {
                webSocket.send(message);
            }
        }
    }
    ```

#### 注意事项

  * **权限:** 确保在 `AndroidManifest.xml` 中添加网络权限：
    ```xml
    <uses-permission android:name="android.permission.INTERNET" />
    ```
  * **线程:** OkHttp 的 WebSocket 回调是在其内部的线程池中执行的。如果你需要在这些回调中更新 UI，请确保切换到主线程（例如使用 `runOnUiThread()` 或 Handler）。
  * **生命周期管理:** 妥善管理 WebSocket 连接的生命周期。在 Activity/Fragment 销毁时关闭连接，避免内存泄漏或不必要的网络活动。对于需要后台持久连接的应用，应该考虑使用 Android **Service** 来管理 WebSocket 连接。
  * **重连机制:** 生产环境中，你需要实现一套健壮的重连机制。当 `onFailure` 或 `onClosed` 被调用时，根据错误类型和网络状态进行指数退避或其他策略的重连尝试。
  * **心跳/Keep Alive:** WebSocket 协议自带 `PING`/`PONG` 帧，OkHttp 会自动处理这部分。但如果你服务器有特殊的超时设置，可能需要调整 OkHttp 的 `readTimeout` 或自己实现更精细的心跳逻辑（通常不需要）。
  * **安全性 (WSS):** 如果你的服务器使用 `wss://` (WebSocket Secure)，OkHttp 会自动处理 SSL/TLS 加密。确保你的服务器证书是可信的，或者配置 `OkHttpClient` 来信任自签名证书（生产环境不推荐）。
