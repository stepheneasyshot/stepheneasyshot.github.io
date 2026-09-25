---
title: "Android Agent——AppFunctions"
description: "从能力声明到跨应用执行，解析端侧智能体的接口机制。"
pubDate: 2026-09-25
category: ["AI", "Android"]
featured: false
draft: false
image: "/images/blog/blogs_android_agent_appfunctions_cover.webp"
---

让手机助手回答一个问题，和让它真正完成一件事，中间还隔着应用。比如“查一下周五的出行安排，再保存成一条备忘”：助手既要取得行程数据，也要找到笔记应用的写入入口，还得知道最后究竟有没有保存成功。

如果只能通过界面操作，这个任务会变成一串识别、点击、输入和页面检查。应用其实已经拥有查询和保存的业务代码，缺少的是一套能让助手发现、理解并调用这些代码的公共接口。AppFunctions 就处在这一层：**由 Android 管理应用能力的声明、发现与执行，让获得授权的 Agent 通过结构化接口使用应用。**

本文从一次跨应用任务出发，梳理它与 MCP、Android 既有通信机制的关系，再进入接入代码、运行链路与工程边界。文中的笔记应用是用于解释设计的示例，公开产品案例与官方样例会单独标明来源。

> 本文资料核对于 **2026 年 9 月 25 日**。Jetpack 最新版本为 **`1.0.0-alpha12`**，发布于 9 月 23 日，尚无稳定版；官方仍将 AppFunctions 标为实验性预览，完整 Agent 接入只向有限应用和系统 Agent 开放。实现并通过本地测试，不等于已经获得 Gemini 的生产接入资格。参见[版本记录](https://developer.android.com/jetpack/androidx/releases/appfunctions)与[官方 FAQ](https://developer.android.com/ai/appfunctions#frequently-asked-questions-faqs)。

## 给应用增加一个面向 Agent 的入口

AppFunctions 由 Android 平台 API 和配套 Jetpack 库组成。应用可以选择暴露“查找笔记”“创建记录”等业务操作，由系统登记其参数、返回值和说明，供调用方发现和执行。Google 将这套集成方式称为 **Android MCP**。[官方概览](https://developer.android.com/ai/appfunctions)重点强调的是应用提供工具的能力。

因此，接入 AppFunctions 不要求应用先内置一个大模型。一个普通笔记应用也可以提供 `createNote`，模型推理和任务规划由调用方承担。相反，即使应用已经能在本地运行模型，也不会因此自动获得其他应用的数据与操作权限。

从开发视角看，这套机制涉及三个角色：

| 角色 | 承担的工作 | 在出行备忘场景中的行为 |
| --- | --- | --- |
| Agent / 获授权调用方 | 理解目标、选择工具、准备参数、组织多步操作 | 查询行程，整理内容，再请求保存 |
| Android 系统 | 提供能力目录、检查调用资格、分发请求 | 找到目标函数并连接对应实现 |
| 目标应用 | 定义接口、执行业务校验、操作自己的数据 | 行程应用返回安排，笔记应用创建记录 |

这里最有价值的变化，是应用能够把业务意图直接表达出来。界面里的“新建”“确定”“保存”可能分散在多个页面中，对外则可以收敛成一个有明确输入输出的“创建笔记”。这样的接口能随 UI 改版继续存在，也能让调用方直接读取业务结果。

覆盖范围取决于应用实际提供了什么。对于没有接入的功能，助手仍需要其他交互方式。Google 在公开介绍中同时推进 AppFunctions 与 UI 自动化：前者让开发者明确提供能力，后者覆盖尚无专门集成的交互。[官方产品说明](https://android-developers.googleblog.com/2026/02/the-intelligent-os-making-ai-agents.html)

## “Android MCP”应该怎样理解？

MCP 的工具模型允许服务端公布工具名称、描述与输入结构，客户端发现工具后发起调用。AppFunctions 也强调这条“描述能力 → 发现能力 → 结构化调用”的链路。对 Agent 而言，两者都能成为工具来源。[MCP Tools 规范](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)

它们的集成方式有明显区别。AppFunctions 面向 Android 应用和系统能力目录，业务实现运行在目标应用中；标准 MCP 则定义 JSON-RPC 消息及传输机制，包括本地 `stdio` 和 Streamable HTTP。**“MCP 必须在云端运行”并不成立**，本地子进程就是协议明确支持的方式。[MCP Transports 规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)

Google 使用“应用作为本地 MCP server”来解释 Android MCP 的角色分工，但 Android 开发者的实际接入方式是 Kotlin 声明、元数据和平台 API。仅给应用添加 AppFunctions，并不会自动得到一个可被任意桌面 MCP 客户端连接的 HTTP 地址。是否支持这样的互通，需要额外的桥接实现，不能从名称直接推导。[官方集成实践](https://developer.android.com/blog/posts/build-intelligent-android-apps-integrate-into-android-s-intelligence-system-using-app-functions?hl=en)

对于熟悉 Android 的开发者，更容易产生的疑问是：已有 Intent、AIDL 和 ContentProvider，为什么还需要它？这些机制解决的问题可以放在同一张表里看：

| 机制 | 主要表达的内容 | 调用方通常需要知道什么 |
| --- | --- | --- |
| Intent / App Link | 请求一个动作，或进入特定应用入口 | Action、URI、参数约定及接收组件的行为 |
| AIDL / Binder | 明确定义的跨进程方法调用 | 双方约定的接口、类型与连接方式 |
| ContentProvider | 通过 URI 访问受控数据 | URI、字段、查询约定与访问权限 |
| AppFunctions | 可被发现、描述和调用的应用业务能力 | 系统公布的函数元数据、输入输出和可用状态 |

Intent 本身也有动作描述和匹配能力，AIDL 也能承载复杂业务，因此不能把已有机制简单理解成“无法完成任务”。AppFunctions 的增量是把**面向调用方的能力描述、系统发现与统一执行入口**组织在一起，减少每个 Agent 分别适配每个应用私有接口的工作。底层跨进程执行仍然建立在 Android 的通信与组件机制之上。相关基础可参阅 [Intent](https://developer.android.com/guide/components/intents-filters)、[AIDL](https://developer.android.com/develop/background-work/services/aidl) 与 [ContentProvider](https://developer.android.com/guide/topics/providers/content-provider-basics) 文档。

## 从函数声明到业务执行

AppFunctions 的链路可以分成两段：安装前准备能力契约，运行时查找并执行能力。先看专用 Service 这一条实现路线：

```text
编译阶段
Kotlin 业务入口 + 类型与说明
    → KSP 生成服务分发代码和函数元数据 XML
    → 随应用打包，由 Manifest 关联

运行阶段
Agent 发现函数并准备参数
    → AppFunctionManager
    → Android 系统检查与分发
    → 目标应用的 AppFunctionService
    → 应用业务层
    → 结构化结果 / 错误返回 Agent
```

### 能力目录记录的是接口契约

在 Service 型接入中，`@AppFunctionServiceEntryPoint` 标记抽象服务入口，`@AppFunction` 标记允许对外调用的方法。KSP 据此生成具体服务类、请求分发代码以及元数据 XML；Manifest 则把生成的服务和资源关联起来。系统由此知道应用有哪些函数，以及调用时需要什么数据。[服务入口 API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionServiceEntryPoint)

这也解释了为什么函数描述会成为接口的一部分。启用 `isDescribedByKDoc` 后，KDoc 会进入能力元数据，供调用方理解函数用途与参数含义。应用级元数据还能说明多个函数之间的操作约定。[接入指南](https://developer.android.com/ai/appfunctions/add-appfunctions)

公开 AOSP 实现中，可以看到 AppSearch 元数据观察、状态读写与按用户同步的逻辑。这些代码服务于能力目录维护。登记“能够查询笔记”与索引“全部笔记正文”是两件不同的事：前者让系统认识接口，后者仍取决于应用具体提供什么数据能力。[AOSP 系统服务实现](https://android.googlesource.com/platform/frameworks/base/+/HEAD/services/appfunctions/java/com/android/server/appfunctions/AppFunctionManagerServiceImpl.java)

### 函数存在，还要看当前是否可用

调用方不仅需要知道函数签名，也需要知道它现在能否执行。例如，同一个“保存到工作空间”函数，在用户退出账户后可能需要禁用；应用升级后，参数定义也可能变化。

当前 Jetpack 将元数据和状态分开：`searchAppFunctions` 用于查询定义，`getAppFunctionStates` 获取运行状态，`observeAppFunctions` 提供变化通知。查询其他应用还受权限与包可见性约束；没有搜到某个函数，不宜立即归因于提供方没有声明。[AppFunctionManager API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionManager)

这些接口为工具选择提供信息，具体选择仍由调用方完成。工程上也应处理“发现时可用、执行时已经失效”的情况：目录可以缓存，但执行结果必须按当时状态判断。

### 系统分发，应用执行业务

Service 路线中，调用请求经 Binder 进入系统服务。公开实现会读取真实调用 UID/PID、验证调用包与目标用户，再检查相关策略、执行资格和函数状态，之后解析并绑定目标服务。业务方法最终在目标应用中执行，结果沿调用链返回。这描述的是所引 AOSP 实现的职责分工，不应据此假定所有设备分支内部细节完全相同。[AOSP 调用链](https://android.googlesource.com/platform/frameworks/base/+/HEAD/services/appfunctions/java/com/android/server/appfunctions/AppFunctionManagerServiceImpl.java)

因此，创建笔记可以通过专用服务完成，无须先展示编辑页面。应用 UI 与 AppFunctions 入口则适合复用同一套 UseCase / Repository，让账户检查、数据校验和持久化规则保持一致。

回到“查询行程并生成备忘”的例子，行程查询和笔记创建仍是两次独立调用。中间的信息整理、用户确认以及失败后的处理属于任务编排。单次函数成功，只能说明该步骤完成，不能自动代表整个跨应用任务成功。

## 用一个“创建笔记”接口理解接入

下面采用 alpha12 的 Service 入口模型展示实现骨架。代码省略 import，假设应用已有 Hilt 配置以及可注入的 `NotesRepository`；仓库方法属于示例业务接口，需要由实际项目实现。它用于说明结构，未作为独立 Android 工程编译或在设备上运行。

先配置 KSP，并让运行库与编译器使用同一版本。下面的依赖版本来自[Jetpack 发布记录](https://developer.android.com/jetpack/androidx/releases/appfunctions)：

```kotlin
dependencies {
    implementation("androidx.appfunctions:appfunctions:1.0.0-alpha12")
    ksp("androidx.appfunctions:appfunctions-compiler:1.0.0-alpha12")
}
```

一次创建操作的契约，可以从“输入是什么、返回什么”开始设计。这里把标题和正文设为必填，并返回可供后续查询的笔记 ID。数据对象通过 `@AppFunctionSerializable` 描述，主构造函数中的属性会成为对外结构的一部分。[序列化类型说明](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionSerializable)

```kotlin
@AppFunctionSerializable(isDescribedByKDoc = true)
data class NoteResult(
    /** 已保存笔记的稳定标识，可用于后续查询。 */
    val id: String,
    /** 最终保存的标题。 */
    val title: String,
)

@RequiresApi(36)
@AndroidEntryPoint
@AppFunctionServiceEntryPoint(
    serviceName = "NotesAppFunctionService",
    appFunctionXmlFileName = "notes_functions",
)
abstract class BaseNotesAppFunctionService : AppFunctionService() {
    @Inject lateinit var notesRepository: NotesRepository

    /**
     * 在当前登录账户下创建一条笔记。
     *
     * @param title 非空标题。
     * @param content 非空正文。
     * @return 已持久化笔记的标识与标题。
     */
    @AppFunction(isDescribedByKDoc = true)
    suspend fun createNote(title: String, content: String): NoteResult {
        if (title.isBlank() || content.isBlank()) {
            throw AppFunctionInvalidArgumentException("标题和正文不能为空")
        }
        return withContext(Dispatchers.IO) {
            // 示例业务方法：负责账户校验、写入，并返回保存后的对象。
            val saved = notesRepository.createForCurrentAccount(title, content)
            NoteResult(id = saved.id, title = saved.title)
        }
    }
}
```

这里有两个容易忽略的细节。首先，描述与代码必须一致：“当前登录账户”应由业务层真正校验，KDoc 无法代替执行条件。其次，当前注解契约说明函数默认在主线程执行；`suspend` 本身不会自动切换线程，可能阻塞的业务需要选择合适的调度器。取消请求也需要业务代码配合处理。[AppFunction API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunction)

KSP 生成的具体类是 `NotesAppFunctionService`。在 Manifest 的 `<application>` 内关联它，才能把声明接到系统入口上；类名和 XML 文件名需要与前面的配置一致：

```xml
<service
    android:name="com.example.notes.NotesAppFunctionService"
    android:permission="android.permission.BIND_APP_FUNCTION_SERVICE"
    android:exported="true">
    <property
        android:name="android.app.appfunctions.schema"
        android:value="app_functions_schema.xsd" />
    <property
        android:name="android.app.appfunctions.v2"
        android:value="notes_functions.xml" />
    <intent-filter>
        <action android:name="android.app.appfunctions.AppFunctionService" />
    </intent-filter>
</service>
```

这是与上述服务入口对应的声明片段。完整项目还应按[官方接入指南](https://developer.android.com/ai/appfunctions/add-appfunctions)配置应用级 `app_metadata`；其中可以解释账户、默认目标等跨函数约定。Service 入口与动态注册入口使用的元数据配置有所不同，复制配置时需要核对具体路线。

这个示例只展示基本创建流程。若要用于真实的 Agent 写操作，还需要进一步处理重复调用与失败恢复，后文会继续展开。

## 动态能力：让“当前页面”也能提供工具

“创建笔记”通常是应用级能力，与用户当前打开哪个页面无关。但“取得当前选中的段落”“把正在浏览的商品加入购物车”依赖具体界面。把这些操作一律做成全局服务，就需要额外解释“当前”究竟指什么。

动态注册路线为这类需求提供了明确入口：先用 `@AppFunctionSignature` 声明函数签名，编译生成元数据与适配器，再在运行时注册具体实现。**动态的是处理函数的实现与可用期，签名仍需要提前声明。**[动态签名 API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionSignature)

| 维度 | 专用 Service | 动态注册实现 |
| --- | --- | --- |
| 适合的能力 | 查询数据、创建记录等应用级业务 | 操作当前页面、当前对象或当前选区 |
| 实现入口 | 系统解析并绑定专用服务 | Activity / Service 上注册的运行时实现 |
| 生命周期 | 不要求已有页面实例 | 受注册状态、组件与进程状态影响 |
| 调用时的上下文 | 通过显式参数传入业务对象 | 可结合 Activity 实例定位当前对象 |

在 alpha12 中，动态注册相关接口仍带实验性标记，并要求 API 37。注册上下文被销毁、实现被解除注册等情况都会影响可用性；不能把动态注册对象当成一个随时能由系统重新创建的持久服务。[AppFunctionManager API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionManager)

`GLOBAL` 与 `ACTIVITY` 作用域进一步解决实例区分问题。假设文档应用同时打开两个窗口，它们都提供“读取选中段落”，调用时需要通过 `activityId` 指向正确实例。这里的 scope 表示生命周期与实例唯一性范围；数据访问权限仍需要另外判断。官方签名示例还建议把注册与 `RESUMED` 等适当生命周期绑定，避免页面已经离开，工具却继续指向旧对象。[动态签名及生命周期示例](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionSignature)

对应用设计而言，这意味着工具目录可以同时容纳“整个应用会做什么”和“这个页面此刻能做什么”。后者有助于减少通过截图猜测上下文的工作，但也要求调用方接受能力随界面变化而失效。

## 接口质量决定 Agent 能否可靠使用

下面几项是结合上述机制提出的工程建议。它们需要在应用和调用方中落地，不能仅靠添加注解获得。

### 把业务语义写进契约

一个 `process(input: String)` 很容易实现，却把解释输入的责任全部留给调用方。换成 `findNotebook`、`createNote`、`getNote` 这样的操作，调用方就能逐步取得真实标识、发起写入并核对结果。粒度应围绕有业务意义的动作确定，而不必把每个底层数据库操作都暴露出来。

描述中最值得交代的是容易产生歧义的信息：时间是否带时区、金额使用什么单位、缺少目标 ID 时会怎样、创建与覆盖是否是不同操作。像“删除前确认”这样的规则，还应由可信代码验证；自然语言说明主要用于帮助选择和构造请求。

**默认值尤其值得单独检查。**当前 `AppFunctionSerializable` 契约说明：构造参数设置默认值后，字段可以被标为可选，但请求缺少字段时，并不保证使用 Kotlin 声明的那个值。非空基本类型会按 JVM 默认值处理，可空类型为 `null`，集合则按相应空值规则处理。[序列化 API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionSerializable)

因此，不宜直接假设 `val limit: Int = 20` 在外部请求缺参时必然得到 20。更清晰的接口可以使用 `val limit: Int? = null`，再由业务代码执行 `limit ?: 20`，并校验上下界。这样默认行为成为显式业务规则，也便于测试。

### 区分调用资格、业务权限与用户意图

当前平台文档把 `EXECUTE_APP_FUNCTIONS` 标为 `normal` 权限，同时明确保留运行时 allowlist 检查及可能的附加要求。因此，添加 Manifest 权限只是接入条件的一部分，不能由此推导普通应用已经能调用所有目标应用。[权限参考](https://developer.android.com/reference/android/Manifest.permission#EXECUTE_APP_FUNCTIONS)

系统准入之后，业务仍要判断当前账户能否访问目标对象、该对象是否存在，以及是否允许修改。允许助手调用消息应用，也不意味着某一次发送的收件人和正文一定符合用户意图。敏感或破坏性操作应把确认对象和实际执行对象对应起来。

还要明确数据会走到哪里。AppFunctions 的本地执行路径并不限制调用方必须使用端侧模型；官方指南提醒，系统 Agent 可能在服务器上处理查询。[数据与功能暴露建议](https://developer.android.com/ai/appfunctions/add-appfunctions#considerations-of-types-of-functionalities-to-make-available) 设计查询接口时，可以先返回完成任务所需的摘要和标识，再按需获取详细内容，避免一次性扩大数据暴露范围。

从 Agent 实现角度，我还会把工具返回的网页、邮件和笔记内容视为外部数据。结果中的文字即使看起来像指令，也不应自行扩大下一次调用的权限或替用户确认操作。这是工具消费端需要维护的信任边界。

### 为结果核验与重试留下接口

“函数返回成功”之外，更实用的信息是创建后的业务 ID、最终状态和必要的版本信息。调用方可以据此展示结果，也可以在后续步骤中引用同一个对象。

写操作尤其需要考虑一种情况：数据库已经写入，但进程在返回结果前退出。调用方看到失败后再次执行，可能产生两条相同笔记。若业务允许，可以接收调用方生成的请求 ID，并在持久化事务内以它做唯一约束：同一请求再次到达时返回已有结果。仅仅在内存中记录“调用过了”无法覆盖进程重启。

取消也有类似边界。协程停止不等于此前完成的写入被撤销；业务事务、外部请求和已经发送的消息，各自有不同的恢复方式。对于“查询 → 创建 → 分享”这样的跨应用流程，需要明确每一步结果、重试条件，以及部分成功时如何向用户解释。

## 如何验证：先固定输入，再加入自然语言

最初的验证目标可以很小：用固定参数调用一个函数，检查真实业务数据，再验证失败输入。这样能够先排除服务声明、元数据生成和业务逻辑问题，之后再评估模型是否选对工具、填对参数。

官方提供了 `adb shell cmd app_function` 命令行工具。开发实践文章将其放在 Android 17 及更新设备的测试路径中，因此先查看目标设备实际支持的命令，再执行列表和调用操作：[官方测试说明](https://developer.android.com/blog/posts/build-intelligent-android-apps-integrate-into-android-s-intelligence-system-using-app-functions?hl=en)。

```bash
adb shell cmd app_function
adb shell cmd app_function list-app-functions
```

确认实际函数标识后，可以按设备帮助中的格式传入参数。若上述示例包名为 `com.example.notes`，声明得到的标识与下面一致，调用形态如下：

```bash
adb shell "cmd app_function execute-app-function \
  --package com.example.notes \
  --function 'com.example.notes.BaseNotesAppFunctionService#createNote' \
  --parameters '{\"title\":\"出行备忘\",\"content\":\"周五出发前检查证件\"}'"
```

函数标识应以生成结果或设备查询结果为准。执行后应读取实际保存的数据；只检查命令是否返回，无法发现保存到了错误账户、错误笔记本等业务问题。

进一步测试可以使用官方 [AppFunctions Testing Agent](https://github.com/android/appfunctions/blob/main/agent/README.md)。它同时支持确定性输入和基于模型的测试，其特权启动方式用于开发调试。仓库中早期预发布版本已经标注因 API 破坏性变化而弃用，应按当前源码与说明准备工具。[预发布说明](https://github.com/android/appfunctions/releases)

一轮有价值的验证应覆盖几类不同故障：应用冷启动时能否执行，账户退出后是否正确拒绝，缺参和非法参数如何返回，重复请求是否产生重复数据，以及页面退出后动态能力是否失效。之后再加入自然语言任务，观察工具选择和多步编排质量，能更容易定位问题发生在哪一层。

## 目前落地到哪里了？

公开产品中，Google 在 2026 年 2 月介绍了 Galaxy S26 上 Gemini 与 Samsung Gallery 的 AppFunctions 集成：用户提出照片查询，助手调用相册能力，并在会话里展示返回的照片。这说明应用数据确实可以通过明确接口进入助手交互，但案例不能外推为所有相册应用或设备都已接入。[官方案例](https://android-developers.googleblog.com/2026/02/the-intelligent-os-making-ai-agents.html)

开发者参考则更适合看样例。官方 [ChatApp](https://github.com/android/appfunctions) 展示联系人查找、消息发送和通话等通信能力；[JetPacker 集成文章](https://developer.android.com/blog/posts/build-intelligent-android-apps-integrate-into-android-s-intelligence-system-using-app-functions?hl=en) 展示如何把旅行应用已有业务组织成可供 Agent 使用的函数。两者的价值在于接口划分和工程实现，不代表普通第三方应用已经获得同样的生产准入。

当前还存在几条不同的版本边界：

| 层次 | 截至本文核对时的状态 |
| --- | --- |
| 平台基础能力 | 官方概览以 Android 16 / API 36 及以上为基础路线 |
| Jetpack 库 | `1.0.0-alpha12`，API 仍在调整 |
| 动态注册 | 当前相关接口要求 API 37，并标为实验性 |
| Gemini 完整集成 | 官方仍描述为有限预览 / EAP，申请不等于准入 |
| 交互会话控制 | 平台参考出现标记为 37.2 的相关 API，需要单独核对设备支持 |

对旧系统还有一个细节：alpha12 的 `getInstance` 实现包含 API 34+ 且设备提供 App Function 扩展库的支持路径，并排除 profile 用户场景。这是有设备前提的兼容分支，不能据此宣称任意 Android 14/15 应用添加 Maven 依赖即可使用。调用方应以实际取得的 Manager 和目标接口可用性为准。[Manager API](https://developer.android.com/reference/kotlin/androidx/appfunctions/AppFunctionManager)

最近的 API 变化也说明了阅读教程时为什么要带上版本：alpha10 引入服务入口模型，alpha11 分离元数据和状态，alpha12 增加动态注册、`activityId` 与 scope，并移除旧的 `AppFunctionContext` 和 `AppFunctionConfiguration`。这些变化会影响示例能否直接迁移，接入时应把运行库、编译器与示例代码一起核对。[完整变更记录](https://developer.android.com/jetpack/androidx/releases/appfunctions)

另一个值得观察的方向是用户控制。平台参考中的 [`AppInteractionSession`](https://developer.android.com/reference/android/app/AppInteractionSession) 标记为版本 37.2，可在连续交互中关联用户同意，并在达到最大生命周期后关闭。它为多步任务提供交互上下文，但本身不承担业务事务或模型对话记忆。这里能够确认的是 API 已被公开描述，实际可用范围仍需按平台版本与设备验证。

## 应用接口开始面向另一类使用者

AppFunctions 带来的变化，可以落到一个具体的开发问题上：一个业务功能除了供页面调用，是否也能以边界清楚、结果可核验的方式，供获得授权的 Agent 使用？如果答案是肯定的，应用就多了一种完成用户任务的入口。

当前最值得投入的是接口本身的质量：把业务动作表达清楚，约束参数和数据范围，返回能够继续使用的结果，并让错误与重试行为可预测。这些工作既影响模型调用的可靠性，也决定应用在跨应用任务中能否成为一个可信的执行方。

从现有设计看，我更关注两条后续进展：动态能力如何与页面上下文结合，以及用户如何在连续任务中理解和控制应用访问。它们会影响 Agent 实际使用工具的体验；生态开放和稳定版本的时间，则应继续以官方公告为准。

## 延伸阅读

- [AppFunctions 官方概览与 FAQ](https://developer.android.com/ai/appfunctions)：能力定位、开放状态和使用场景。
- [接入 AppFunctions](https://developer.android.com/ai/appfunctions/add-appfunctions)：应用声明、元数据配置和验证入口。
- [Jetpack AppFunctions 版本记录](https://developer.android.com/jetpack/androidx/releases/appfunctions)：迁移前优先核对 API 变化。
- [alpha12 官方源码包](https://dl.google.com/dl/android/maven2/androidx/appfunctions/appfunctions/1.0.0-alpha12/appfunctions-1.0.0-alpha12-sources.jar)：本文用它复核了注解、参数默认值、动态注册与兼容分支，避免只依据持续变化的主分支。
- [Android 官方样例与测试 Agent](https://github.com/android/appfunctions)：从应用提供方和调用方两侧理解接入。
- [AOSP AppFunctionManagerServiceImpl](https://android.googlesource.com/platform/frameworks/base/+/HEAD/services/appfunctions/java/com/android/server/appfunctions/AppFunctionManagerServiceImpl.java)：阅读服务式执行的系统调用链。
- [MCP Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools) 与 [Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)：区分工具模型与具体通信协议。
