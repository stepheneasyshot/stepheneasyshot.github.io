---
layout: post
description: > 
  本文介绍了流行框架retrofit的设计理念和实现原理
image: 
  path: /assets/img/blog/blogs_retrofit_library.png
  srcset: 
    1920w: /assets/img/blog/blogs_retrofit_library.png
    960w:  /assets/img/blog/blogs_retrofit_library.png
    480w:  /assets/img/blog/blogs_retrofit_library.png
accent_image: /assets/img/blog/blogs_retrofit_library.png
excerpt_separator: <!--more-->
sitemap: false
---
# Retrofit设计模式解析
Retrofit 是一个由 Square 公司（现在叫 Block）开发的 **类型安全 (Type-safe) 的 HTTP 客户端**，专门用于 Android 应用开发。

它的核心思想非常巧妙：**将一个 HTTP API 抽象成一个 Java 接口**。你只需要通过 **注解（Annotations）** 来描述网络请求的各个部分（如请求方法、URL、请求头、请求体等），Retrofit 就会在运行时自动为你生成实现这个接口的 **动态代理对象** ，然后你就可以像调用一个普通的 Java 方法一样来发起网络请求了。
## Retrofit解决的痛点
在 Retrofit 出现之前，开发者通常使用 `HttpURLConnection` 或 Apache `HttpClient`，代码冗长、易错且难以维护。后来出现的 OkHttp 极大地改善了底层通信，但直接使用 OkHttp 仍然需要手动构建 Request 和处理 Response，不够直观。

### **声明式 API**
接口的定义设计为了声明式 API，代码简洁优雅，你只需定义一个接口和一些方法，用注解来描述 API，完全无需关心如何构建 `Request` 对象、如何解析 `Response` 等繁琐细节。

传统方式需要手动拼接 URL，设置请求头，创建请求体，执行请求，检查状态码，读取输入流，然后用 Gson/Jackson 手动解析 JSON 字符串。整个过程可能需要几十行代码。使用Retrofit只需要定义一个接口方法，一行代码调用即可。

### **类型安全**
类型安全也是 Retrofit 最强大的特性之一。网络请求的返回数据会通过转换器（Converter）自动反序列化成你指定的 Java 或 Kotlin 对象（如一个 `User` 或 `List<Repo>`）。

如果在接口中定义的返回类型与服务器实际返回的 JSON 结构不匹配，会在运行时通过转换器抛出异常，而不是得到一个 `null` 或导致 `ClassCastException`，这使得错误能够被快速定位和处理。请求参数的类型也是确定的，减少了手动拼写等错误的概率。

### **高度集成与可扩展性**
Retrofit 底层默认且推荐使用 OkHttp 来执行实际的网络请求。因此，它自动继承了 OkHttp 的所有优点，如连接池、Gzip 压缩、请求重试和响应缓存等。你还可以自定义 `OkHttpClient` 来配置拦截器（Interceptor）、超时时间等。

通过可插拔的 `Converter`，Retrofit 不仅支持 JSON (通过 Gson, Moshi, Jackson)，还支持 XML, Protobuf 等多种数据格式。

除了直接使用默认的 `Call<T>` 回调方式。通过 `CallAdapter`，Retrofit 可以将请求的返回类型适配成不同的异步工具，完美支持 Kotlin 协程 (Coroutines)、RxJava、Flow、Guava 等。

### **性能优异**
由于底层依赖于高性能的 OkHttp，Retrofit 本身的开销非常小。它主要是在初始创建时通过反射生成接口实现，一旦创建完成，后续调用的性能非常高。

## Retrofit 的核心组件
下面这几个组件是掌握 Retrofit 的关键。
### **API 接口 (The API Interface)**
定义一个 `interface` 接口类，在接口里定义网络请求方法，然后使用注解来描述每个请求方法。

**常用注解**：
* **请求方法**: `@GET`, `@POST`, `@PUT`, `@DELETE`, `@HEAD`, `@PATCH`。
* **URL 处理**:
    * `@Path`: 替换 URL 中的路径段，如 `/users/{id}` 中的 `id`。
    * `@Query`: 添加查询参数，如 `/users?sort=desc` 中的 `sort`。
    * `@Url`: 当需要动态指定完整 URL 时使用。
* **请求体**:
    * `@Body`: 指定一个对象作为请求体，通常用于 `POST` 或 `PUT`，会被 Converter 序列化。
    * `@Field`: 用于表单提交 (`application/x-www-form-urlencoded`)，需要与 `@FormUrlEncoded` 配合使用。
* **请求头**: `@Header`, `@Headers`。

```kotlin
// api declaration GithubApiService.kt
interface GithubApiService {
  @GET("users/{username}")
  fun getUser(@Path("username") username: String): Call<User>
}
```

### **Retrofit 类 (The Retrofit Class)**
这是整个框架的入口，通过建造者模式 `Retrofit.Builder` 来配置和构建一个 Retrofit 实例。

主要配置项一般为：
* `baseUrl()`: API 的基础 URL，必须以 `/` 结尾。
* `addConverterFactory()`: 添加数据转换器工厂，用于序列化和反序列化。
* `addCallAdapterFactory()`: 添加调用适配器工厂，用于支持不同的异步库。
* `client()`: 传入一个自定义的 `OkHttpClient` 实例。

```kotlin
// create retrofit instance
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .addCallAdapterFactory(CoroutineCallAdapterFactory())
    .build()

// create api service instance
val githubApiService = retrofit.create(GithubApiService::class.java)
```

### **转换器 (Converter)**
负责将 Java 对象（POJO）与网络请求的 Body（如 JSON, XML）进行相互转换。
    * **请求时**：将 `@Body` 注解的对象序列化成网络请求体。
    * **响应时**：将网络响应体反序列化成接口方法定义的返回类型对象。
    * **常用转换器**：
        * `converter-gson`: 使用 Google 的 Gson 库。
        * `converter-moshi`: 使用 Square 的 Moshi 库，性能更高，对 Kotlin 更友好。
        * `converter-jackson`: 使用流行的 Jackson 库。

###  **调用适配器 (Call Adapter)**
这个配置决定了你接口方法的返回类型。默认情况下，方法返回 `Call<T>` 类型，通过添加适配器，你可以让方法直接返回其他类型，从而与现代异步编程范式结合。

**示例**：
* 不加适配器：`fun getUser(): Call<User>`
* 使用协程：`suspend fun getUser(): User` (需要 `CallAdapter` 在幕后处理)
* 使用 RxJava：`fun getUser(): Single<User>`

## 一个完整的 Kotlin + 协程示例
下面这个具体的例子，从 GitHub API 获取一个用户的信息。

**第 1 步：添加依赖 (build.gradle.kts)**
```kotlin
// Retrofit
implementation("com.squareup.retrofit2:retrofit:2.11.0") // 请使用最新版本
// Gson Converter
implementation("com.squareup.retrofit2:converter-gson:2.11.0")
// OkHttp (可选，用于自定义配置和日志拦截器)
implementation("com.squareup.okhttp3:okhttp:4.12.0")
implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
```

**第 2 步：定义数据模型 (Data Class/POJO)**
```kotlin
data class User(
    val login: String,
    val id: Int,
    val avatar_url: String,
    val name: String?,
    val company: String?
)
```

**第 3 步：定义 API 接口**
```kotlin
import retrofit2.http.GET
import retrofit2.http.Path

interface GithubApiService {
    /**
     * 使用 suspend 关键字，使其成为一个协程挂起函数
     * Retrofit 会自动处理线程切换和结果返回
     */
    @GET("users/{username}")
    suspend fun getUser(@Path("username") username: String): User
}
```

**第 4 步：创建 Retrofit 实例**
通常我们会把它放在一个单例或者依赖注入的模块中。

```kotlin
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {

    private const val BASE_URL = "https://api.github.com/"

    // 创建一个日志拦截器，用于打印网络请求和响应的日志
    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY // 打印最详细的日志
    }

    // 创建一个 OkHttpClient，并添加日志拦截器
    private val okHttpClient = OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .build()

    val instance: GithubApiService by lazy {
        val retrofit = Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient) // 设置自定义的 OkHttpClient
            .addConverterFactory(GsonConverterFactory.create())
            // 注意：对于 suspend 函数，Retrofit 内部已经自动处理了 CallAdapter，无需显式添加
            .build()
        
        retrofit.create(GithubApiService::class.java)
    }
}
```

**第 5 步：发起网络请求**
在 ViewModel 或者其他有协程作用域的地方调用。

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.launch

class MyViewModel : ViewModel() {

    fun fetchGithubUser(username: String) {
        // 在 ViewModel 的协程作用域中启动一个新协程
        viewModelScope.launch {
            try {
                // 直接调用接口方法，就像调用一个普通函数一样
                val user = RetrofitClient.instance.getUser(username)
                
                // 请求成功，处理 user 数据
                println("User Name: ${user.name}")
                println("User Company: ${user.company}")
                
            } catch (e: Exception) {
                // 请求失败，处理异常
                println("Request failed: ${e.message}")
            }
        }
    }
}
```

这个例子清晰地展示了 Retrofit 的威力：定义接口、创建实例、然后直接调用方法。所有复杂的网络细节都被完美地隐藏了起来。
## 小结
**Retrofit 不是一个网络请求库，而是一个网络请求的“封装”或“适配”库。** 它站在巨人 OkHttp 的肩膀上，通过注解和动态代理技术，将开发者从繁琐的网络请求实现中解放出来，让我们能够用更符合直觉、更健壮、更易于维护的方式与 REST API 进行交互。

在今天的 Android 开发中，Retrofit + OkHttp + Kotlin Coroutines + Moshi/Gson 的组合已经成为应用架构网络层的“黄金标准”，是每个 Android 开发者都应该熟练掌握的技能。

## Ktor
Ktor 是一个由 JetBrains 开发的，用于在 Kotlin 中构建连接应用的异步框架。它旨在提供一个轻量级、灵活且高度可扩展的网络应用框架，既可以用于构建服务器端应用（如 RESTful API、微服务、Web 网站），也可以用于构建多平台 HTTP 客户端应用。

![](/assets/img/blog/blogs_ktor_cover.png)

## Android平台上的基础使用
首先添加gradle依赖：
```groovy
implementation "io.ktor:ktor-client-core:$ktor_version"
implementation "io.ktor:ktor-client-android:$ktor_version"
```

之后就可以使用HttpClient定制化，例如添加日志、内容协商、序列化等功能。然后就可以使用HttpClient发送请求了。

```kotlin
class KtorClient {

    companion object {
        const val TAG = "KtorClient"
    }

    private val client = HttpClient(CIO) {
        install(Logging) {
            level = LogLevel.ALL
        }
        install(ContentNegotiation) {
            json(Json {
                prettyPrint = true
                isLenient = true
            })
        }
    }

    suspend fun getGithubRepos(userName: String) = withContext(Dispatchers.IO) {
        client.get("https://api.github.com/users/${userName}/repos")
            .body<List<GithubRepoItem>>()
    }

    fun release() {
        client.close()
    }
}
```

> CIO 是 Ktor 自己的纯 Kotlin 实现的 I/O 引擎。它的设计目标是轻量级、无额外依赖、并完全基于 Kotlin 协程构建。这意味着它能最大程度地利用 Kotlin 协程的异步特性，提供高效且非阻塞的 I/O 操作。它直接利用 Kotlin 协程的调度和挂起机制来处理网络事件。它的内部实现尽可能地避免了阻塞操作，并且通过协程调度来管理并发连接。

## 设计理念
### 纯 Kotlin 和协程优先（Kotlin and Coroutines First）
Ktor 完全基于 Kotlin 语言构建，充分利用 Kotlin 的语言特性，例如 DSL（领域特定语言）、扩展函数、协程等。

Ktor 的异步编程模型是基于 Kotlin 协程实现的。这意味着开发者可以使用看似 **同步的命令式代码来编写异步逻辑** ，极大地简化了并发编程的复杂性，避免了回调地狱，并提高了代码的可读性和可维护性。每个请求都会在 Ktor 中启动一个新的协程来处理，从而实现高效的并发。
### 轻量级和非侵入式（Lightweight and Unopinionated）
Ktor 不强加固定的项目结构或技术栈。它允许开发者根据自己的需求选择日志、模板引擎、消息队列、持久化、序列化、依赖注入等各种技术。

它提供了一个松散耦合的架构，你可以只使用你需要的功能，而不是一个庞大的全功能框架。这种灵活性使得 Ktor 非常适合构建微服务或需要高度定制化的应用。

Ktor 的 API 大多是函数调用和 Lambda 表达式，结合 Kotlin 的 DSL 能力，使代码看起来声明式且简洁。
### 高度可扩展性（Highly Extensible via Plugins/Features）
Ktor 采用 “插件” 机制来实现其核心功能和可扩展性。诸如内容协商、身份验证、日志、会话管理、压缩等功能都是通过安装插件来实现的。

这种统一的拦截机制允许在请求/响应处理管道的不同阶段插入自定义逻辑。开发者可以轻松地编写自己的插件来扩展 Ktor 的功能，或者集成第三方库。
### 多平台支持（Multiplatform Support）
Ktor 不仅仅局限于 JVM。Ktor Client 模块支持 Kotlin Multiplatform Mobile (KMM)，允许在 Android、iOS、桌面以及服务器端共享网络逻辑。这使得 Ktor 成为构建跨平台应用的理想选择。
### 类型安全路由（Type-Safe Routing）
Ktor 提供了类型安全的路由机制，允许通过类而不是字符串来定义 API 结构和路由参数。这在编译时就可以验证路由参数和路径，减少了常见的运行时错误，并使得重构更加安全和可管理。
