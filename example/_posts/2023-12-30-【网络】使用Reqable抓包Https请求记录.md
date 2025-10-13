---
layout: post
description: > 
  本文记录了使用Reqable抓包Https请求的过程。
image: 
  path: /assets/img/blog/blogs_reqable_site_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_reqable_site_cover.png
    960w:  /assets/img/blog/blogs_reqable_site_cover.png
    480w:  /assets/img/blog/blogs_reqable_site_cover.png
accent_image: /assets/img/blog/blogs_reqable_site_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【网络】使用Reqable抓包Https请求记录
在集成DeepSeek的Chat对话API时，我遇到了一个典型的用户体验问题：**启用流式输出（streaming response）后，服务器返回的数据过于密集且速度快**，导致前端页面渲染出现卡顿，文字滚动不连贯，严重影响交互流畅度。

为了精准定位问题根源，我需要分析实际网络传输过程中的数据包特征——包括响应头、数据分块频率、SSE协议实现细节等。此时，专业的抓包工具成为不可或缺的助手。经过对比，我选择了 **Reqable** 这款支持跨平台、全协议（尤其是HTTPS解密）的抓包工具，它不仅能捕获明文和加密流量，还提供直观的过滤和可视化分析功能。

本文将详细记录从环境搭建到HTTPS抓包配置的全过程，并解析DeepSeek流式接口返回的 `text/event-stream` 数据特征。
## 抓包工具工作原理
我们都知道HTTPS协议是加密传输，是安全的，抓包工具是如何插到客户端和服务器中间，来明文地查看通信报文的呢？

本文介绍的Reqable，或者其他工具如 Fiddler、Charles 等抓包工具能够拦截并显示 HTTPS 报文的通信信息的原理，核心在于利用了 **“中间人攻击”（Man-in-the-Middle, MITM）** 技术，但它是一种 **受控的、合法的、且需要用户授权的操作** 。

本质上，抓包工具扮演了一个 **“代理服务器”（Proxy Server）** 的角色，将自己插入到客户端（浏览器、App）和目标服务器之间。

以下是抓包工具拦截和解密 HTTPS 报文的详细步骤：
### 1. 代理和证书的安装
这是实现 HTTPS 抓包的前提条件。必须让APP信任抓包工具提供的证书，才可能往下推进。

1.  **设置代理：** 您需要将客户端设备（如手机、电脑）的网络配置指向抓包工具运行的机器的 IP 和端口，让所有的网络流量都流经抓包工具。
2.  **安装根证书（Root Certificate）：** 抓包工具会生成一个自己的**根证书**。您必须将这个证书安装到客户端设备（手机或电脑）的**系统信任证书列表**中。这是最关键的一步，它让客户端设备“相信”抓包工具是合法的证书颁发机构（CA）。

#### 1.1 需要注意的情况
从 Android 7 开始，App 默认不再信任用户安装的 CA 证书（即抓包工具的根证书），除非 App 的 `network_security_config.xml` 文件中显式配置允许，Reqable就是采用了这个方案。如果缺少证书的验证，只能看到客户端的所有网络请求地址，在校验初期就会失败而中断连接。

另外，有些高度安全的 App 会将目标服务器的**真实证书**或**公钥**硬编码到 App 代码中。在这种情况下，即使您安装了抓包工具的根证书，App 也会发现抓包工具发送的**假证书**与硬编码的真实证书不匹配，从而立即终止连接，防止抓包。要绕过证书固定，通常需要使用 Hook 工具在 App 验证证书的代码执行前将其拦截和修改，这也是移动安全研究的一个重要领域。
### 2. TLS 握手时的“中间人”行为
当客户端（App 或浏览器）尝试通过 HTTPS 连接目标服务器时，抓包工具开始扮演“中间人”的角色。

**1. 客户端问候** ，客户端向抓包工具发送 `Client Hello`。抓包工具将 `Client Hello` 转发给目标服务器。
**2. 服务器响应** ，抓包工具收到目标服务器的 `Server Hello` 和**真实证书**。目标服务器发送**真实证书**。
**3. 伪造证书** ，抓包工具**动态生成一个“假证书”**，其域名与目标服务器的域名（如 `www.google.com`）一致。然后，**抓包工具使用自己安装在客户端上的“根证书”**来签名这个假证书。
**4. 抓包工具问候**，抓包工具将这个**伪造的、已签名的假证书**发送给客户端。
**5. 客户端信任** ，客户端收到证书后，发现这个假证书的签发者（抓包工具）的根证书在自己的信任列表里（因为您手动安装了），因此**客户端认为这个假证书是合法的**。

### 3. 双重加密和解密
握手完成后，抓包工具实际上建立了两个独立的 TLS 连接：

* 客户端 $\leftrightarrow$ 抓包工具** ，抓包工具是服务器的角色，使用**假证书**协商出的**对称密钥 $K_A$** ，报文**被加密**（用 $K_A$）
* 抓包工具 $\leftrightarrow$ 目标服务器，抓包工具是客户端的角色，使用**真实证书**协商出的**对称密钥 $K_B$** ，报文**被加密**（用 $K_B$）

**数据流转过程：**

1.  **客户端发送数据：** 客户端用 $K_A$ 加密数据 $\rightarrow$ 发给抓包工具。
2.  **抓包工具解密（第一次）：** 抓包工具收到数据后，用自己的 $K_A$ 将报文**解密成明文**。
3.  **显示与修改：** 此时，**Reqable 等工具就能以明文形式显示通信信息**，甚至允许您在转发前修改报文内容。
4.  **抓包工具加密（第二次）：** 抓包工具用 $K_B$ 重新加密报文 $\rightarrow$ 发给目标服务器。
5.  **目标服务器解密：** 目标服务器用自己的 $K_B$ 将报文解密。

通过建立并控制这两个独立的 TLS 通道，抓包工具成功地看到了 **中间的明文数据。**
## Reqable 的安装
Reqable是一款集API抓包调试与测试于一体的高效工具，支持以下核心能力：
- **多平台覆盖**：Windows/macOS/Linux/Android/iOS全主流操作系统
- **HTTPS解密**：通过安装CA证书实现加密流量的明文查看
- **协议支持全面**：HTTP/1.1、HTTP/2、WebSocket、gRPC、SSE等
- **过滤与分析强大**：支持按域名、状态码、内容类型等条件快速筛选目标请求

在下载页面下载并安装Reqable的MACOS和Android的安装包，地址为：

[Reqable下载页面](https://reqable.com/zh-CN/download/)


要抓取HTTPS请求，必须解决**中间人攻击防护机制**导致的加密流量不可见问题。Reqable采用标准CA证书信任机制，需要在客户端（Mac和Android）分别安装其根证书并信任。

Android平台需要手动下载并配置Android端的证书，下载Reqable的证书后，在设置的安全页面，点击安装证书。
### Android证书安装
关键步骤为手动配置证书信任：

1. 手机与电脑处于同一局域网，连接Wi-Fi后进入 **Wi-Fi设置 → 选择当前网络 → 修改网络 → 高级 → 代理**，设置为 **手动代理**。
2. 输入PC的本地IP（如 `192.168.1.5`）和Reqable默认监听端口 `9000`（可在Reqable的「设置→代理」中确认）。
3. 在Reqable中导出CA证书（设置 → 证书 → 导出），或直接通过手机浏览器访问证书下载链接（若失败则手动导出并传输到手机）：链接2。
4. 在手机 **设置 → 安全 → 加密与凭据 → 安装证书 → CA证书**，选择下载的证书文件完成安装（需输入锁屏密码授权）。
5. 最后进入 **设置 → 安全 → 加密与凭据 → 信任的凭据 → 用户**，确认证书已成功安装。

> 📌 注意：若你的Android项目未自动信任用户证书（例如非Debug包或Flutter项目），需额外配置网络安全策略（详见下文）。

添加网络安全配置文件开发者在项目源码中配置网络安全文件用户证书，并重新打包，选择下面任意一种方式即可。注意，网络安全配置不适用于Flutter项目。

**build.gradle中配置依赖（推荐）**

```
dependencies {
    debugImplementation 'com.reqable:reqable-android:1.0.0'
}
```

> Debug包将自动集成网络安全配置文件，如果无法连接到Maven中央仓库，请按照下方的指引手动创建并配置网络安全文件。

**手动创建网络安全文件新建文件 `res/xml/network_security_config.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
  <base-config cleartextTrafficPermitted="true">
    <trust-anchors>
      <certificates src="system" />
      <certificates src="user" />
    </trust-anchors>
  </base-config>
</network-security-config>
```

**配置到 AndroidManifest.xml**

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
    ...
</application>
```

完成以上步骤后，重新打包应用，进行双端互联，即可开始抓包。
## 抓包配置
完成双端配置后，启动Reqable的抓包功能（默认开启代理监听）。界面会实时显示所有经过代理的网络请求，包括请求方法、URL、状态码、响应时间等基础信息。

初始状态下，列表可能包含大量无关请求（如广告、资源加载）。为聚焦目标，我们通过 **域名过滤** 快速定位DeepSeek相关流量。

点击启动按钮后，在列表中可以看到所有的网络请求的情况。

![](/assets/img/blog/blogs_reqable_all_packages.png)

对于Deepseek的调试，我将服务器地址这里的 `https://api.deepseek.cn` 添加到Reqable的筛选中。

在AI助手页面内，输入对应的功能，触发网络请求之后，即可看到服务器返回的情况。

![](/assets/img/blog/blogs_reqable_stream_chat.png)

根据原始数据可以看出，服务器返回的类型确实是一个流式的数据 `text/event-stream` 。选中某一条目标请求，展开详情面板。从响应头中可以看到关键信息：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
...
```

重点字段解析：
- **Content-Type: text/event-stream**：表明服务器返回的是Server-Sent Events（SSE）格式的流式数据，专为实时推送设计。
- **Transfer-Encoding: chunked**：数据以分块（chunked）形式传输，适合不确定长度的流式内容。

进一步切换到 **SSE** 标签页（Reqable针对SSE协议提供的专用解析视图），可以清晰看到服务端推送的数据块结构。根据DeepSeek官方文档说明，当参数 `stream=true` 时，API会以SSE形式持续返回增量内容，每条数据以 `data: {...}` 格式发送，最终以 `data: [DONE]` 标志结束。

通过观察原始数据流，我们发现服务端推送频率较高（例如每秒多次），且单次数据块较小——这正是前端渲染卡顿的直接原因。

> 根据Deepseek官方文档，如果设置为 True，将会以 SSE（server-sent events）的形式以流式发送消息增量。消息流以 data: `[DONE]` 结尾。SSE（Server-Sent Events）返回类型是一种用于实现服务器向客户端单向实时推送数据的技术。它基于 HTTP 协议，允许服务器通过一个长连接持续向客户端发送数据流。

## 优化思路
基于抓包分析结果，我们可以针对性地优化用户体验。对接收到的SSE数据块进行缓冲，合并短时间内的多次推送后再更新UI，减少频繁重绘。或者在每一个数据包推送过来之后，手动加一段100-200ms的延时，让界面有一定的响应时间，避免用户体验上的卡顿。或者与DeepSeek团队沟通，探讨是否支持调整推送频率或合并小数据块（例如按语义段落聚合）。