---
layout: post
description: > 
  本文旨在介绍一个AI Agent系统中有哪些模块，以及他们是如何协作的。
image: 
  path: /assets/img/blog/blogs_ai_agent_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_ai_agent_cover.png
    960w:  /assets/img/blog/blogs_ai_agent_cover.png
    480w:  /assets/img/blog/blogs_ai_agent_cover.png
accent_image: /assets/img/blog/blogs_ai_agent_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】从基础模型搭建一个Agent系统
OpenAI 的应用研究主管 Lilian Weng 撰写了一篇博客，认为 AI Agent 可能会成为新时代的开端。她提出了 Agent=LLM + 规划技能 + 记忆 + 工具使用的基础架构，其中 LLM 扮演了 Agent 的 “大脑”，在这个系统中提供推理、规划等能力。

结构图：

![](/assets/img/blog/blogs_lilian_agent_blog.png)

*  **文章标题：LLM 驱动的自主智能体 (LLM Powered Autonomous Agents)**
*   **原作者：** Lilian Weng
*   **发布时间：** 2023年6月23日
*   **原文地址：** [https://lilianweng.github.io/posts/2023-06-23-agent/](https://lilianweng.github.io/posts/2023-06-23-agent/)

---

### **1. 智能体系统概述 (Agent System Overview)**

构建以 LLM（大语言模型）为核心控制器的智能体是一个非常酷的概念。诸如 **AutoGPT**, **GPT-Engineer** 和 **BabyAGI** 等概念验证演示（Proof-of-concepts demos）都是极具启发性的例子。LLM 的潜力远不止于生成优美的文案、故事、散文或程序；它可以被构建成一个强大的通用问题解决器。

在一个 LLM 驱动的自主智能体系统中，LLM 充当智能体的“大脑”，并辅以以下几个关键组件：

*   **规划 (Planning)**
    *   **子目标与分解 (Subgoal and decomposition)：** 智能体将大型任务分解为更小、更易管理的子目标，从而高效处理复杂任务。
    *   **反思与精炼 (Reflection and refinement)：** 智能体能够对过去的行为进行自我批评和自我反思，从错误中学习，并为未来的步骤进行优化，从而提高最终结果的质量。
*   **记忆 (Memory)**
    *   **短期记忆 (Short-term memory)：** 我认为所有的“上下文学习 (In-context learning)”都属于利用模型的短期记忆。
    *   **长期记忆 (Long-term memory)：** 这使智能体具备保留和回忆（无限）信息的能力，通常通过利用外部向量存储（Vector Store）和快速检索来实现。
*   **工具使用 (Tool use)**
    *   智能体学会调用外部 API 来获取模型权重中缺失的信息（通常在预训练后很难改变），包括当前信息、代码执行能力、访问专有信息源等。

---

### **2. 组件一：规划 (Planning)**

复杂任务通常涉及许多步骤。智能体需要知道它们是什么，并提前规划。

#### **2.1 任务分解 (Task Decomposition)**
**思维链 (Chain of Thought, CoT)** 已成为一种标准的提示技术，用于提升模型在复杂任务上的表现。模型被指示“逐步思考”，利用更多的测试时计算资源将困难任务分解为更小、更简单的步骤。CoT 将大任务转化为多个可管理的任务，并揭示了模型的思维过程。

**思维树 (Tree of Thoughts, ToT)** 通过在每个步骤中探索多种推理可能性来扩展 CoT。它首先将问题分解为多个思维步骤，并为每一步生成多个思维，从而创建树状结构。搜索过程可以是 BFS（广度优先搜索）或 DFS（深度优先搜索），每个状态通过分类器（通过提示）或多数投票进行评估。

任务分解可以通过以下方式完成：
1.  通过简单的提示让 LLM 自行完成，例如 `"XYZ 的步骤。\n1."`, `"实现 XYZ 的子目标是什么？"`。
2.  使用特定任务的指令；例如，写小说时用 `"写一个故事大纲"`。
3.  结合人类输入。

另一种截然不同的方法是 **LLM+P**，它依赖于外部的经典规划器（Classical Planner）来进行长视野规划。此方法利用 **PDDL (Planning Domain Definition Language)** 作为中间接口来描述规划问题。在此过程中，LLM：
1.  将问题翻译为“Problem PDDL”。
2.  请求经典规划器基于现有的“Domain PDDL”生成 PDDL 计划。
3.  将 PDDL 计划翻译回自然语言。
本质上，规划步骤被外包给了外部工具，这在某些机器人设置中很常见，但在许多其他领域并非如此。

#### **2.2 自我反思 (Self-Reflection)**
自我反思是智能体通过精炼过去的行为决策并纠正先前的错误来迭代改进自身的关键方面。在不可避免需要试错的现实世界任务中，这一点至关重要。

*   **ReAct (Reasoning + Acting)**
    ReAct 通过将动作空间扩展为“特定任务的离散动作”和“语言空间”的组合，在 LLM 中集成了推理和行动。前者使 LLM 能够与环境交互（例如使用 Wikipedia 搜索 API），而后者促使 LLM 生成自然语言形式的推理轨迹。
    ReAct 的提示模板包含 LLM 的明确思考步骤，格式大致如下：
    ```text
    思考 (Thought): ...
    行动 (Action): ...
    观察 (Observation): ...
    ... (重复多次)
    ```
    在知识密集型任务和决策任务的实验中，`ReAct` 的表现都优于移除了 `思考` 步骤的 `Act-only` 基线。

*   **Reflexion (反射)**
    Reflexion 是一个为智能体配备动态记忆和自我反思能力以提高推理技能的框架。Reflexion 具有标准的 RL（强化学习）设置，其中奖励模型提供简单的二元奖励，动作空间遵循 ReAct 的设置（用语言增强特定任务的动作空间）。在每次动作 $a_t$ 后，智能体计算一个启发式函数 $h_t$，并根据自我反思结果决定是否重置环境以开始新一轮尝试。
    *   **启发式函数**：确定轨迹何时效率低下或包含幻觉（Hallucination）并应停止。效率低下指花费太长时间未成功。幻觉指在环境中遇到导致相同观察结果的一连串相同动作。
    *   **自我反思**：通过向 LLM 展示两个失败轨迹与理想反思的示例对来创建。然后将反思添加到智能体的工作记忆中（最多三个），作为查询 LLM 的上下文。

*   **事后诸葛亮思维链 (Chain of Hindsight, CoH)**
    CoH 通过明确展示给模型一系列经过注释的过去输出（包含反馈），鼓励模型基于自身输出进行改进。人类反馈数据是 $D_h = \{(x, y_i, r_i, z_i)\}_{i=1}^n$ 的集合，其中 $x$ 是提示，$y_i$ 是模型补全，$r_i$ 是 $y_i$ 的人类评分，$z_i$ 是相应的人类提供的事后诸葛亮反馈。
    假设反馈元组按奖励排序 ($r_n \ge r_{n-1} \ge \dots \ge r_1$)。该过程是监督微调，数据是一个序列形式 $\tau_h = (x, z_i, y_i, z_j, y_j, \dots, z_n, y_n)$。模型微调的目标是仅预测 $y_n$（基于序列前缀），使模型能够基于反馈序列进行自我反思以产生更好的输出。可选地，模型在测试时可以接收多轮带有人类注释者的指令。
    *   为防止过拟合，CoH 添加了一个正则化项以最大化预训练数据集的对数似然。
    *   为防止走捷径和复制（因为反馈序列中有许多常见词），在训练期间随机屏蔽 0%-5% 的过去 token。

*   **算法蒸馏 (Algorithm Distillation, AD)**
    AD 将 CoH 的思想应用到了强化学习任务的跨回合轨迹中，其中“算法”被封装在长历史条件策略中。假设智能体与环境交互多次，且在每个回合中智能体变得越来越好，AD 将这种学习历史连接起来并输入模型。因此，预期下一个预测动作将导致比前几次更好的性能。目标是学习 RL 的过程，而不是训练特定任务的策略本身。

---

### **3. 组件二：记忆 (Memory)**

#### **3.1 记忆的类型 (Types of Memory)**
记忆可以定义为用于获取、存储、保留和随后检索信息的过程。人脑中有几种类型的记忆：
1.  **感觉记忆 (Sensory Memory)：** 最早的记忆阶段，提供在原始刺激结束后的感官信息（视觉、听觉等）印象保留能力，通常仅持续几秒钟。
2.  **短期记忆 (Short-Term Memory, STM) 或 工作记忆 (Working Memory)：** 存储我们当前意识到并需要执行复杂认知任务（如学习和推理）的信息。短期记忆容量有限（约 7 个组块），持续 20-30 秒。
3.  **长期记忆 (Long-Term Memory, LTM)：** 可以将信息存储很长时间（几天到几十年），容量基本无限。分为：
    *   **外显/陈述性记忆 (Explicit/Declarative)：** 事实和事件的记忆（如情景记忆、语义记忆）。
    *   **内隐/程序性记忆 (Implicit/Procedural)：** 无意识的记忆，如骑自行车或打字。

**在 AI 智能体中的映射：**
*   **感觉记忆：** 学习原始输入（文本、图像等）的嵌入表示。
*   **短期记忆：** 上下文学习 (In-context learning)。受限于 Transformer 的有限上下文窗口，它是短暂的。
*   **长期记忆：** 外部向量存储（Vector Store），智能体在查询时可通过快速检索访问。

#### **3.2 最大内积搜索 (Maximum Inner Product Search, MIPS)**
外部记忆可以缓解有限注意力跨度的限制。标准做法是将信息的嵌入表示保存到支持快速 MIPS 的向量存储数据库中。为了优化检索速度，通常选择 **近似最近邻 (ANN)** 算法来以牺牲一点精度换取巨大的速度提升。

常见的 ANN 算法：
*   **LSH (Locality-Sensitive Hashing)：** 局部敏感哈希，将相似项目映射到相同桶中。
*   **ANNOY (Approximate Nearest Neighbors Oh Yeah)：** 使用随机投影树（Random Projection Trees）。
*   **HNSW (Hierarchical Navigable Small World)：** 受小世界网络启发，构建层级化的小世界图。
*   **FAISS (Facebook AI Similarity Search)：** 假设高维空间中距离服从高斯分布，应用向量量化。
*   **ScaNN (Scalable Nearest Neighbors)：** 使用各向异性向量量化。

---

### **4. 组件三：工具使用 (Tool Use)**

工具使用是人类的显著特征。为 LLM 配备外部工具可以显著扩展模型能力。

*   **MRKL 系统**
    MRKL（Modular Reasoning, Knowledge and Language）是一种神经符号（Neuro-symbolic）架构。它包含一组“专家”模块，通用 LLM 充当路由器，将查询路由到最合适的专家模块。这些模块可以是神经网络（深度学习模型）或符号型（计算器、货币转换器、天气 API）。
    *   实验表明，让 LLM 学会调用计算器解决口头数学问题比解决明确陈述的数学问题更难，因为 LLM 往往无法可靠地提取基本算术的正确参数。这突显了“何时以及如何使用工具”比工具本身更重要。

*   **TALM 与 Toolformer**
    两者都通过微调语言模型（LM）来学习使用外部工具 API。

*   **HuggingGPT**
    一个使用 ChatGPT 作为任务规划器的框架，根据模型描述选择 HuggingFace 平台上的模型，并根据执行结果总结响应。系统包含四个阶段：
    1.  **任务规划 (Task Planning)：** LLM 将用户请求解析为多个任务（含任务类型、ID、依赖关系、参数）。
    2.  **模型选择 (Model Selection)：** LLM 从候选列表中选择合适的专家模型。
    3.  **任务执行 (Task Execution)：** 专家模型执行特定任务。
    4.  **响应生成 (Response Generation)：** LLM 接收结果并提供给用户。

*   **API-Bank**
    一个用于评估工具增强 LLM 性能的基准测试。它包含 53 个常用 API 工具、一个完整的工具增强 LLM 工作流，以及 264 个包含 568 次 API 调用的注释对话。它评估智能体在三个层面的能力：
    1.  **调用 API (Call)：** 给定描述，确定是否调用给定 API。
    2.  **检索 API (Retrieve)：** 搜索可能的 API 并通过阅读文档学习如何使用。
    3.  **规划 API (Plan)：** 针对模糊请求（如安排行程），进行多 API 调用规划。

---

### **5. 案例研究 (Case Studies)**

#### **5.1 科学发现智能体 (Scientific Discovery Agent)**
*   **ChemCrow：** 一个特定领域的例子，LLM 增强了 13 个专家设计的工具，用于完成有机合成、药物发现和材料设计任务。工作流结合了 **ReAct** 和 **MRKL**，指令模型遵循 `思考、行动、行动输入、观察` 的格式。
    *   **有趣发现：** 虽然 LLM 评估认为 GPT-4 和 ChemCrow 表现相当，但人类专家评估显示 ChemCrow 在化学正确性上远超 GPT-4。这表明使用 LLM 评估其在需要深厚专业知识领域的表现可能存在缺陷。
*   **Boiko et al. (2023)：** 探索了用于科学发现的智能体，能够使用工具浏览互联网、阅读文档、执行代码、调用机器人实验 API。但也讨论了风险（如合成违禁药物），测试中 36% 的化学武器合成请求被接受。

#### **5.2 生成式智能体模拟 (Generative Agents Simulation)**
*   **Generative Agents (Park et al. 2023)：** 一个有趣的实验，25 个由 LLM 驱动的虚拟角色生活在一个沙盒环境中（灵感来自《模拟人生》）。
    *   **记忆流 (Memory Stream)：** 长期记忆模块，记录代理经历。
    *   **检索 (Retrieval)：** 根据相关性、**新颖性 (Recency)**、**重要性 (Importance)**（询问 LM 区分琐事和核心记忆）来检索上下文。
    *   **反思 (Reflection)：** 生成关于过去事件的更高层次摘要（例如：“我昨晚睡得很好” -> “我现在感觉精力充沛”）。
    *   **结果：** 模拟产生了涌现的社会行为，如信息传播、关系记忆和社交活动协调。

#### **5.3 概念验证示例 (Proof-of-Concept Examples)**
*   **AutoGPT：** 引起了广泛关注，它将 LLM 作为主要控制器。虽然存在可靠性问题（由于自然语言接口），但这是一个很酷的概念验证。AutoGPT 的大量代码都用于格式解析。
*   **GPT-Engineer：** 根据自然语言规范创建整个代码仓库。它首先会列出需要澄清的超级短子弹列表，然后选择一个澄清问题等待用户回答，最后进入代码编写模式。

---

### **6. 挑战 (Challenges)**

在构建以 LLM 为中心的智能体时，存在一些常见的局限性：
1.  **有限的上下文长度 (Finite context length)：** 限制了历史信息、详细指令、API 调用上下文和响应的包含。虽然向量存储可以提供更大的知识库，但其表示能力不如全注意力机制强大。
2.  **长期规划与任务分解的挑战 (Challenges in long-term planning)：** 在冗长的历史记录上进行规划和有效探索解决方案空间仍然很困难。当面临意外错误时，LLM 难以调整计划。
3.  **自然语言接口的可靠性 (Reliability of natural language interface)：** 当前的智能体系统依赖自然语言作为 LLM 与外部组件（记忆、工具）之间的接口。然而，模型输出的可靠性值得怀疑，LLM 可能会出现格式错误或偶尔表现出叛逆行为（拒绝遵循指令）。因此，许多智能体演示代码都集中在解析模型输出上。
