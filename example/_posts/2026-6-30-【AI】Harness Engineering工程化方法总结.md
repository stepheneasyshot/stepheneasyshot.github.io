---
layout: post
description: > 
  本文旨在介绍Harness Engineering的概念演化由来
image: 
  path: /assets/img/blog/blogs_cover_ai_engineeering.png
  srcset: 
    1920w: /assets/img/blog/blogs_cover_ai_engineeering.png
    960w:  /assets/img/blog/blogs_cover_ai_engineeering.png
    480w:  /assets/img/blog/blogs_cover_ai_engineeering.png
accent_image: /assets/img/blog/blogs_cover_ai_engineeering.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】Harness Engineering工程化方法总结
从22年的GPT，及之后生成式人工智能爆发至今，大语言模型的底层能力已经完成了一次次跨越式的蜕变。然而在产业落地、尤其是智能体开发的前沿演进中，越来越多的人发现：决定 AI 应用最终成败的，往往不仅是那个博古通今的“模型大脑”，更是围绕它搭建的系统工程。

如果把大模型比作一匹拥有无穷神力的“旷野烈马”，那么我们要在现实世界中驾驭它走向既定终点，就必须演进出三套截然不同、却互为犄角的底层工程学科：

* **提示词工程（Prompt Engineering）：** 研究如何用最精准、最具逻辑的指令，唤醒 AI 的深度思考能力；
* **上下文工程（Context Engineering）：** 动态调整的“视野与记忆”，研究如何跨越海量信息与高昂成本的物理限制，在万级 Token 中动态提炼出最关键的业务背景与知识输入；
* **脚手架工程（Harness Engineering）：** 它是确保绝对安全的“缰绳与防具”，研究如何在大模型非确定性的提议下，构建一个隔离沙箱与自动化反馈的闭环系统，确保 AI 的行动绝对可靠。

从探索 AI **能理解什么**（Prompt），到管理 AI **需要知道什么**（Context），再到约束 AI **被允许做什么**（Harness），这三大热门技术正在构建起 AI Agent 时代的全新技术版图。本文将带你逐一拆解这三个核心概念，探寻大模型走向工业级生产力的真正秘密。

## Harness Engineering
过去两年，很多人认为 AI 能不能完成任务，取决于 Prompt 写得够不够好。后来大家发现，仅靠 Prompt 已经远远不够。于是又出现了 **Context Engineering**——如何给 AI 提供正确的上下文。而到了 2026 年，OpenAI 在介绍 Codex 内部研发方式时，首次提出了一个新的概念：

> **Harness Engineering**

它代表着 AI Agent 时代的软件开发方式，开始发生根本性的变化。

### 什么是 Harness？
Harness 这个词原本来自英文。最初的意思是马具、缰绳、安全带。它最大的作用不是提供动力，而是表示一种约束、控制、引导。一匹马本身拥有动力，但是如果没有缰绳来控制，它就无法完成运输任务。

同样的，LLM拥有很强的推理能力。但是没有 Harness，它也只是一个会回答问题，和人对话的大模型。所以很多人现在开始使用下面这个公式：

```
AI Agent = LLM + Harness
```

其中：

LLM 提供的是：

* 推理能力（Reasoning）
* 编码能力（Coding）
* 理解能力（Understanding）

Harness 提供的是：

* 工具
* 权限
* 环境
* 上下文
* 验证
* 反馈
* 记忆
* 工作流

真正让 Agent 能工作的，不只是模型，而是整个运行系统。


### Context Engineering 和 Prompt Engineering 的局限性
假设现在有一个典型的大型 Android 项目：

```
app/
feature-home/
feature-login/
feature-profile/
feature-order/
common/
network/
storage/
ui/
compose/
...

总代码：

250万 LOC

Module：120+

开发人员：60+

CI：GitHub Actions

测试：
Unit Test
Instrumentation Test
Compose Test
```

你的需求只有一句：

> 修复首页点击商品偶尔闪退的问题。

对于 AI 来说，它需要完成找到 Crash，分析原因，找到代码，修改，运行测试，验证，提交 PR

但是不同阶段，AI 的能力完全不同。Prompt Engineering 的核心思想就是 **把需求写清楚。**

例如：

```
你是一位资深 Android 工程师。

请帮我修复首页商品点击闪退的问题。

请遵循 Kotlin 官方编码规范。

不要修改无关代码。
```

或者再高级一点：

```
请采用 MVVM 架构。

不要影响其他 Feature。

修改完成以后给我 Diff。
```

Prompt 已经写得很好，但是问题来了，AI 根本不知道：

```
首页代码在哪里？

哪个 Module？

哪个 Activity？

哪个 ViewModel？

哪个 Repository？

哪个接口？

哪个 Crash？
```

Prompt 写的再漂亮，AI 还是不知道去哪找。Prompt Engineering最大的问题就是：

> **它只能告诉 AI "应该做什么"，不能告诉 AI "在哪里做"。**

Android 大项目里：

```
app/
feature-home/
feature-home-new/
feature-home-v2/
feature-feed/
feature-discover/
feature-video/
```

Prompt 不可能告诉 AI：

```
首页其实在：
feature-feed
而不是：
feature-home
```

更不可能告诉 AI：

```
真正 Crash 的代码
是在 common-image 库。
```

在实际项目应用中，Prompt 最大的问题是没有工程信息。只能由使用人先自己理解一遍，再将疑难点通过Prompt优化后，提供给AI，获取解决方案，自行导入。

#### 第二阶段：Context Engineering
大家发现光有 Prompt 不够时，于是开始想办法提供 Context。

例如：

```
README

Architecture.md

Coding Style

API 文档

最近修改记录

相关源码

Git Diff

Issue

Crash 日志
```

现在 Prompt 变成 Prompt + Context ，例如：

```
下面是：
HomeViewModel
ProductRepository
Crash 日志
请修复。
```

AI 的能力立刻提升很多。可以阅读、分析、修改。这就是 Context Engineering。

问题又来了。大型 Android 项目不是几个文件。而可能是250万的代码，AI 不可能一次读取全部代码到上下文。对于小点的项目，即使可以一次对话获取全部信息，也会产生巨大的噪声信息，让AI模型抓不到重点。还会出现新的问题：Context Window 不够。例如：

```
HomeFragment
↓
调用
HomeViewModel
↓
调用
Repository
↓
调用
Network
↓
调用
Cache
↓
调用
Room
↓
调用
Common Library
```

真正 Bug 可能在Common Library中，但是Context 里没有。AI 根本不知道，AI 只能猜。

还有一个问题：

Context 是静态的。给 AI 10 个源码文件，AI 修改以后。下一步怎么办？

不知道。

因为 Context 不会自动更新。AI 也不会自己继续获取。

所以Context Engineering 最大的问题就是：

> **Context 是一次性提供的。**

AI不会继续找代码，也不会继续搜索，继续阅读，继续验证

### 第三阶段：Harness Engineering
Harness 的思想完全不同，它的信息传递的理念是

> **不要一次把 Context 全给 AI。**

而是让 AI 自己获取。

例如：

AI 收到：

```
修复首页 Crash。
```

第一步：

AI 自己执行：

```
grep
rg
find
git
ls
```

搜索：

```
Home
Product
Crash
```

然后发现报错在 `feature-feed` 这个模组，则会继续打开源码，进一步发现在ProductViewModel这个类的loadProduct() 方法，然后打开 Repository，查看网络请求。直到找到真正 Crash。

这里最大的变化就是：

**Context 不再是"喂给 AI"，而是 AI 主动去获取。**

其实真正的大变化其实在后面。AI 修改代码以后：

它不会说：

```
Done.
```

而是继续：

```
./gradlew test
```

发现：

```
Build Failed
```

继续：

```
读取错误日志
```

修改。

继续：

```
./gradlew assembleDebug
```

发现：

```
Lint Error
```

继续：

```
修改
```

然后：

```
运行 Unit Test
```

最后：

```
运行 Compose Test
```

最后：

```
全部通过。
```

这整个循环：

```
写代码
↓
编译
↓
测试
↓
修复
↓
重新测试
↓
提交
```

就是 Harness。

把 AI 看成一位刚加入团队的新 Android 工程师：

Prompt Engineering：你只告诉他：“修一下首页闪退。”
Context Engineering：你再把设计文档、Crash 日志、相关源码打印出来交给他。
Harness Engineering：你不仅告诉他目标，还给了他 IDE、Git 权限、终端、Gradle、ADB、模拟器、CI、日志平台以及代码搜索工具，并允许他自主查阅文档、运行测试、反复验证直到修复完成。

前两者更像是在回答问题（Question Answering）；Harness Engineering 则是把 AI 放进一个真实的软件工程环境，让它像一名工程师一样，通过"观察 → 分析 → 执行 → 验证 → 反馈"的闭环完成整个开发任务。这也是它与 Prompt Engineering、Context Engineering 最本质的区别。

### Harness 到底包含哪些东西？
OpenAI 实际上做的事情，不是让 Codex 更聪明。而是不断完善 Codex 周围的"基础设施"。可以把 Harness 看成下面这张结构图：

```
                 Human

                    │

             High Level Goal

                    │

          +-------------------+
          |   Harness Layer   |
          +-------------------+

      Prompt
      Context
      Memory
      Tool Calling
      Permissions
      File System
      Git
      Terminal
      Browser
      Logs
      Metrics
      CI
      Test
      Review
      Retry
      Verification

                    │

                  LLM

                    │

               Generated Code
```

可以发现真正复杂的已经不是 Prompt，而是 Harness的架构。

### Harness Engineering 的核心思想
一句话概括：

> **不要教 AI 怎么写代码，而是让 AI 能够自己完成整个开发流程。**

OpenAI 在内部几乎不再人工写代码，而是让 Codex：

```
读取需求
↓
阅读代码
↓
分析影响范围
↓
修改代码
↓
生成测试
↓
运行测试
↓
查看日志
↓
修复 Bug
↓
重新测试
↓
提交 PR
↓
AI Review
↓
继续修改
↓
直到全部通过
```

整个过程形成闭环。人类负责只制定目标。Agent 负责执行目标。

### 工程师的角色正在变化
这是 OpenAI 提出的一个非常重要的观点。以前的软件工程师：

```
需求
↓
自己写代码
↓
测试
↓
上线
```

未来的软件工程师：

```
需求
↓
设计 Harness
↓
定义规则
↓
设计反馈
↓
Agent 自动开发
```

工程师不再是：

> Code Producer（代码生产者）

而更像：

> System Designer（系统设计者）

也就是让 AI 能够稳定工作的工程师。

### Harness 的几个核心组成

#### 1. Context System（上下文系统）
AI 最大的问题不是不会写，而是不知道：

```
哪些代码最重要？
```

因此需要：

```
代码索引

知识库

Architecture Docs

AGENTS.md

Coding Rules

Dependency Graph
```

OpenAI 提到，他们没有把所有内容塞进一个巨大的 `AGENTS.md` 文件，而是尽量让知识分层、按需提供，让 Agent 获得的是"地图"，而不是一本几百页的说明书。

#### 2. Tool System（工具系统）

Agent 不应该只能聊天。

它应该拥有：

```
Git

Shell

IDE

Browser

Debugger

Docker

CI

GitHub

Jira

Slack
```

这些工具全部开放给 AI。

AI 自己调用。

而不是人复制粘贴。

#### 3. Feedback System（反馈系统）

AI 最大的问题：

不知道自己错了。

Harness 就负责告诉它：

```
编译失败

↓

测试失败

↓

日志异常

↓

页面打不开

↓

性能下降

↓

重新修改
```

整个系统形成：

```
Observe

↓

Analyze

↓

Fix

↓

Verify

↓

Repeat
```

OpenAI 甚至让 Agent 能够读取日志、指标和 Trace，并利用浏览器自动操作 UI 来复现问题、验证修复效果，而不是只依赖静态代码分析。

#### 4. Verification（验证系统）

很多 AI Coding 最大的问题：

```
它说：

Done.
```

实际上：

```
不能运行。
```

Harness 会定义：

```
必须：

Build Success

+

Unit Test Pass

+

E2E Pass

+

Lint Pass

+

Security Pass
```

否则：

不能结束任务。

#### 5. Observability（可观测性）
这是 OpenAI 花了大量篇幅介绍的一部分。他们让 Agent 可以查看：

```
Log
Metric
Trace
Chrome
DOM
Screenshot
Performance
```

于是AI 不只是会写。还能自己 Debug。自己定位问题。自己验证修复。


## 总结 
对于Prompt Engineering 和 Context Engineering，Harness 并没有取代前两者，而是把它们全部包含进去。

模型之间的能力差距正在不断缩小。GPT、Claude、Gemini、Qwen 等模型，在很多编码任务上的基础能力已经越来越接近。

真正决定 Agent 是否可靠的，往往不是模型本身，而是模型所处的运行环境：上下文如何组织、工具如何接入、权限如何控制、失败后如何恢复、结果如何验证。社区越来越多的实践也在强调："更好的 Harness 往往比更换模型带来的收益更大。所以 `Harness Engineering` 并不是一种新的 AI 模型，也不是一种新的 Prompt 技巧。它是一种面向 AI Agent 的软件工程方法论，其核心目标是为智能体构建一个可观察、可验证、可反馈、可控制的运行环境。

未来的软件开发，竞争的重点将不再只是"谁拥有更强的大模型"，而是**谁能够设计出更优秀的 Harness**。在 Agent 时代，工程师创造价值的方式，也将从"亲自编写代码"逐渐转向"设计能够持续放大 AI 能力的系统"。
