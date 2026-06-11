---
layout: post
description: > 
  本文旨在介绍一个AI Agent系统中有哪些模块，以及他们是如何协作的。
image: 
  path: /assets/img/blog/blogs_cover_claudecode.png
  srcset: 
    1920w: /assets/img/blog/blogs_cover_claudecode.png
    960w:  /assets/img/blog/blogs_cover_claudecode.png
    480w:  /assets/img/blog/blogs_cover_claudecode.png
accent_image: /assets/img/blog/blogs_cover_claudecode.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】ClaudeCode初体验与背景知识补齐
在AI辅助编程的讨论中，人们往往过分迷恋底层大模型（LLM）的参数量和基准测试得分（Benchmark）。但我认为：**Agent（智能体）的工程设计，其重要性与模型本身完全平起平坐，甚至决定了AI能否真正进入生产力核心。**

一个再聪明的模型，如果没有良好的工程外壳（工具链调用、上下文控制、状态管理、报错自动追溯），它也只是一个“高谈阔论却无法干活”的面试者；而卓越的Agent工程，能让模型化身为真正能够自主闭环的“资深工程师”。

**Claude Code** 的定位非常不同——它不是一个简单的“代码补全弹窗”，而是一个**直接运行在终端（Terminal）、具备 Agent（智能体）能力的 CLI 工具**。它拥有文件读写、执行终端命令、运行测试甚至自主修复 Bug 的权限。

## 第一次使用记录
回顾自己作为开发者的这几年，我所经历的AI辅助编码路线，其实也是整个行业Agent技术演进的缩影。这段历史可以大概划分为三个阶段：

* **阶段一：网页问答时代（CV工程师）** ，浏览器多标签页（如早期 ChatGPT, Claude 网页版）。遇到需求或 Bug 时，在网页中用自然语言描述，等待 AI 生成代码片段。随后手动CV粘贴到本地 IDE 中，再根据报错进行微调。这种方式有比较大的局限性，即上下文极其断层。AI 对整个项目的架构设计、依赖库版本一无所知，开发者需要像“人体搬运工”一样不断喂报错信息，效率极低。
* **阶段二：IDE 插件时代（TAB工程师）** ，这时候涌现了一批在 Android Studio / IntelliJ IDEA 中的集成插件（如 GitHub Copilot, GitCode Marscode 等）。我们可以在熟悉的编辑环境中，AI 通过静态分析获取当前文件或相邻文件的部分上下文，提供行级代码补全，或者在侧边栏对选中代码进行解释和重构。这个阶段依然处于“被动响应”状态。它能帮你写一个小函数、改一个局部 Bug，但无法跨越多个模块独立完成一个大特性，更没有自主运行测试、根据编译报错自我修正的能力。
* **阶段三：AI-Native IDE 与 Agent 时代（Yes/Accept工程师）** ，以 Cursor 为代表的 AI 优先 IDE，内部引入了真正的 Agent 工程（如 Composer 模式）。它将一个复杂的研发任务拆解为不同性质的子任务（如：架构设计、代码编写、QA 测试），交由不同的专业 Agent 去协作完成。另外还有 Claude Code 这种程序员更偏爱的命令行硬核美学，它没有图形界面，直接驻留于你的终端（Shell）。同样基于多 Agent 协同体系。CC可以深度融入工作流，直接读取整个代码库，自主执行诸如 `git status`、`gradlew assembleDebug`、`pytest` 等终端命令。它修改了代码后，会自动运行编译和测试，如果发现报错，Agent 会**自主捕获终端的 Error 信息并进行下一轮的自我修正**，直到任务完全验证通过。

### 社区真实评价与使用体验
自 Anthropic 推出 Claude Code 以来，整个开发者社区对其评价呈现出一种“硬核、务实、生产力爆棚”的基调。

相比于 Cursor 需要开发者转移到新的 IDE 阵营，玩转 Neovim、tmux 或习惯于全命令行操作的硬核开发者对 Claude Code 赞誉有加。它完美符合 Unix 哲学 —— “通过文本流与现有工具无缝组合”。

区非常推崇它的 `/ultrareview`（多轮深度代码审查）和自主执行循环。开发者只需丢下一句 `“帮我把这个 Android 模块的底层网络库从 HttpURLConnection 重构为 OkHttp，并确保所有单元测试通过”`，就可以去喝咖啡了，Claude Code 会在终端里自己和编译器“死磕”。

当然，也有不少开发者（如 Reddit 的 `r/ClaudeCode` 板块）指出，当面对极其庞大且缺乏规范的陈旧代码库时，Agent 偶尔也会陷入“打补丁式”的无效循环，导致 Token 消耗极快（Token Drift）。这进一步佐证了：**开发者如何通过结构化文档（如编写 `CLAUDE.md` 规范）去引导 Agent 工程，是高效协作的关键。**

### 在国内如何用 DeepSeek 丝滑驱动 Claude Code
由于众所周知的原因，在国内直接直连 Anthropic 官方服务存在网络门槛和账号限制。然而，得益于 **DeepSeek** 的强势崛起及其对 **Anthropic 兼容 API** 的完美原生支持，我们完全可以在国内无缝复用 Claude Code 的强大 Agent 外壳，将底层大模型替换为高性价比、低延迟的 DeepSeek V4。

**为什么不使用哪些中转站呢？** 相比于将自己的项目信息和工程安全暴露给不知名的某些组织，我更愿意相信大厂的服务，所以还是选择了最近新发布的Deepseek V4系列来作为ClaudeCode的大脑。

![](/assets/img/blog/blogs_claude_code_first_try.png)

我看到DeepSeek 官方提供了针对 Anthropic 格式的专属兼容端点。我们只需要在本地终端通过环境变量对 Claude Code 进行重定向代理即可。

#### 步骤一：配置环境变量
在你的终端配置文件（如 `.bashrc` 或 `.zshrc`）中，注入以下环境变量（注意：不要设置 `ANTHROPIC_API_KEY` 以免触发官方鉴权冲突）：

```bash
# 关键：指定 DeepSeek 的 Anthropic 兼容 Base URL
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"

# 填入你的 DeepSeek API Key（通过 Auth Token 变量注入）
export ANTHROPIC_AUTH_TOKEN="sk-your-deepseek-api-key-here"
unset ANTHROPIC_API_KEY

# 指定 Claude Code 运行时的核心模型与辅助模型
export ANTHROPIC_MODEL="deepseek-v4-pro"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"

# 可选：关闭非必要的官方启动流量，提升国内首屏加载速度
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1

```

#### 步骤二：启动与绕过本地权限限制

在项目根目录下启动 Claude Code。由于使用了第三方模型代理，建议加上权限绕过参数以保证本地 Tool（如执行 Bash、读写文件）的顺畅：

```bash
claude --permission-mode bypassPermissions

```

> **💡 高阶贴士（开源代理工具）：**
> 如果在配置过程中遇到证书或复杂的 JSON 元数据兼容问题，国内开发者社区目前非常流行使用开源的 **claude-tap** 或者是桌面端代理工具 **CCPG (Claude Code Proxy Gateway)**。它们能够作为本地中间件，自动抹平 DeepSeek 与 Claude Code 之间微小的 API 字段差异，实现真正的“零漏接”丝滑开发。


## Claude Code 的使用注意事项
权限过高也不一定是好事，OpenClaw之前爆出的众多安全问题也印证了这一点。我们如何在开发中高效且安全地使用它，我梳理了以下背景知识与核心注意事项：

### 1. 它是基于 CLI 的 Agent，而非单纯的 Chat

Claude Code 不是网页端聊天窗口。你把它叫出来后，它会“潜入”你的项目目录。它不仅能看代码，还能自己执行 `git status`、`gradlew test`、`pytest` 或 `make`。

### 2. `CLAUDE.md` — 给AI看的“项目说明书”

这是 Claude Code 官方极其推崇的机制。在项目根目录下创建一个 `CLAUDE.md` 文件，里面用来存放：

* **项目技术栈与构建/测试命令**（例如：如何运行单测、如何编译）。
* **代码风格与架构规范**（例如：在 Android 中必须使用 Kotlin 协程，禁止使用 RxJava；或者 Python 项目必须符合 PEP8）。
* 在项目经历了比较大的重构，比如软件架构，依赖库大升级时，最好及时更新一下这个CLAUDE.md文档，以让cc每次都可以掌握最新最准确的项目信息。

> 💡 **注意**：这个文件要精简。如果写得太长像个老太婆的裹脚布，Claude 反而会漏掉关键信息。

### 3. 三种核心工作模式

Claude Code 支持通过快捷键（如 `Shift + Tab`）或命令切换模式：

* **Ask before edits（默认/问答模式）**：它在修改任何文件前都会问你“我可以改吗？”，适合日常边写边聊。
* **Auto Mode（自动/自主模式）**：你给它一个复杂任务（如“帮我把这个模块的各种边界条件单测补齐”），它会自动连续执行“修改-运行测试-报错-再修改”的闭环，直到成功。
* **Plan Mode（计划模式）**：遇到大需求时，先让它出一份架构和执行计划书，你审批过了再敲定执行。

### 4. 严格管理 Context（上下文）污染 —— 善用 `/clear`

* **痛点**：网页端聊天习惯了“一个窗口聊到底”，但在 Claude Code 里，**对话越长，Token 消耗越恐怖（钱包在流血），且 AI 越容易糊涂**。
* **避坑**：不要搞“大杂烩会话”。刚让它修完一个网络请求的 Bug，接下来想让它写一个 UI 布局，请果断输入 `/clear` 清空当前上下文，重新开始。
* **二次修正定律**：如果 Claude 连续两次都改错了，**不要继续纠正它**。此时上下文已经充满了错误代码的干扰。正确做法是 `/clear`，然后重新组织你的 Prompt，把正确的限制条件一次性喂给它。

### 5. 权限控制：警惕危险的“自动模式”

* **痛点**：在自动模式下，Claude 会自己执行 Shell 命令。如果你的项目中包含敏感脚本，或者它理解错了意图，可能会误删文件或执行了死循环。
* **避坑**：在涉及 `rm`、环境变动或操作未提交的 Git 代码时，仔细审查它的计划。建议在干净的 Git 分支（Clean working directory）上使用它，随时准备 `git reset --hard`。

### 6. 信任但必须验证（Trust, but Verify）

* Claude 经常能写出看似无懈可击但实际上包含隐藏 Bug（如 Android 中的内存泄漏、Python 中的并发竞态问题）的代码。
* **避坑**：不要盲目 Trust。**给 Claude 一种能够自我验证的方式**。例如：“请帮我重构这个类，**并且运行项目中的测试确保没有打破现有逻辑**。” 无法通过自动化测试或编译验证的代码，绝不要直接上线。

### 7. 任务原子化（Atomization）

* **错误示范**：“帮我重构整个 Android 客户端的登录注册流程。”（任务太大，AI 容易在中间迷失，改出无数编译错误）。
* **正确示范**：
1. “先帮我检查现在的登录数据校验逻辑（Data Validation），找出漏洞。”
2. “在这个 Foundation 层增加一个密码加密的工具类。”
3. “最后去修改 ViewModel 里的调用逻辑。”
