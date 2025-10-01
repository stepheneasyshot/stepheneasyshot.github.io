---
layout: post
description: > 
  本文介绍了借助llama.cpp开源框架运行Android平台上的AI小模型的流程
image: 
  path: /assets/img/blog/blogs_ai_llamacpp_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_ai_llamacpp_cover.png
    960w:  /assets/img/blog/blogs_ai_llamacpp_cover.png
    480w:  /assets/img/blog/blogs_ai_llamacpp_cover.png
accent_image: /assets/img/blog/blogs_ai_llamacpp_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】端侧模型部署llama.cpp篇
接上文：

[【AI】大模型开发流程和运行环境简介](./2025-8-5-【AI】LLM开发流程和运行环境简介.md)

## llama.cpp
作为一名 Android 开发者，llama.cpp 打开了在移动应用中实现**端侧 AI** 的全新大门。

**llama.cpp** 是一个由 Georgi Gerganov 开发的开源项目，其核心是用 **C/C++** 编写的。它的主要目标是让大型语言模型 (LLMs) 能够在各种硬件上高效地运行**推理**，尤其是在**本地设备上**，包括那些没有高端独立 GPU 的普通电脑或移动设备。该项目的愿景是 **“民主化”LLMs 的部署** ，让每个人都能在自己的设备上体验和使用这些强大的模型，而不仅仅依赖于昂贵的云服务。

对于Android平台，我们可以通过 `Android NDK` 将 `llama.cpp` 的 C/C++ 代码编译为 `.so` 库，并在您的 `Java/Kotlin` 代码中通过 JNI 调用它。`llama.cpp` 让您可以在 Android 应用中直接拥有一个“本地的智能大脑”，从而创造出前所未有的智能应用体验。

`llama.cpp` 的设计理念是追求**极致的效率、最小化外部依赖、广泛的硬件兼容性以及高度的灵活性**。

![](/assets/img/blog/blogs_ai_llamacpp.png)

llama.cpp 并非一个虚拟机，而是一个高效的、C/C++ 实现的 LLM 推理引擎。它通过模型量化、GGUF 格式、底层硬件优化、多平台支持以及灵活的 API 接口，极大地降低了在个人电脑和边缘设备上运行大型语言模型的门槛。

## LLM的数据结构概念
### 张量
前面的文章介绍了自注意力机制，这个是现代LLM的核心机制，它允许模型在处理序列数据（如文本）时，能够关注不同位置的信息。而这整个过程都是通过**张量（Tensor）**的存储和高效计算来实现的。

以 **自注意力机制（Self-Attention）** 为例，详细说明张量如何组织数据、存储参数和执行计算。
### 1. 存储张量：输入数据和模型参数
#### A. 输入张量 (Input Tensor)
当用户输入一句话（如：“**我爱学习编程**”）时，模型会将每个词转换成一个数字向量（称为词嵌入，Word Embedding）。

* **张量名称：** $\mathbf{X}$ (Input Embedding)
* **张量形状：** $(\text{序列长度}, \text{嵌入维度})$

如果输入是 4 个词，每个词用 1024 维的向量表示，则 $\mathbf{X}$ 的形状是 **$(4, 1024)$**。

#### B. 权重张量 (Weight Tensors)
模型需要学习三组权重矩阵来将输入 $\mathbf{X}$ 转化为查询（Query）、键（Key）和值（Value）向量。

* **张量名称：** $\mathbf{W}_Q, \mathbf{W}_K, \mathbf{W}_V$
* **张量形状：** $(\text{嵌入维度}, \text{输出维度})$

如果输出维度也是 1024，则这三个权重张量都是 **$(1024, 1024)$** 的 2 维矩阵。它们是模型的**永久知识**。
### 2. 计算张量：从输入到 Q/K/V

**目标：** 在一次计算中，同时将序列中的**所有**词的嵌入向量，转换为它们的 Q、K、V 向量。

#### A. 查询张量（Query Tensor）的生成

$$\mathbf{Q} = \mathbf{X} \cdot \mathbf{W}_Q$$

* **操作：** 这是一个巨大的**张量乘法**（即矩阵乘法）。
* **过程：** 在 GPU/TPU 上，张量计算框架（如 PyTorch/TensorFlow）会并行执行：序列长度 4 $\times$ 嵌入维度 1024 的输入，与嵌入维度 $1024 \times 1024$ 的权重矩阵相乘。
* **结果 $\mathbf{Q}$ 的形状：** **$(4, 1024)$**。

**关键：** 传统的编程需要一个 `for` 循环，遍历 4 个词，分别计算。但使用张量，这 4 个词的计算是**一次性并行完成**的，这正是深度学习速度飞快的原因。

#### B. 键张量（Key Tensor）和值张量（Value Tensor）的生成

同理，$\mathbf{K}$ 和 $\mathbf{V}$ 也通过张量乘法并行计算出来：

$$\mathbf{K} = \mathbf{X} \cdot \mathbf{W}_K \quad \text{和} \quad \mathbf{V} = \mathbf{X} \cdot \mathbf{W}_V$$

结果 $\mathbf{K}$ 和 $\mathbf{V}$ 的形状也都是 **$(4, 1024)$**。

### 3. 核心计算：注意力分数的计算

**目标：** 计算 $\mathbf{Q}$ 和 $\mathbf{K}$ 之间的相关性分数。

$$\text{注意力分数} = \mathbf{Q} \cdot \mathbf{K}^T$$

* **操作：** 另一次张量乘法（$\mathbf{K}$ 需要先转置 $T$）。
* **过程：** 形状为 $(4, 1024)$ 的 $\mathbf{Q}$ 张量与形状为 $(1024, 4)$ 的 $\mathbf{K}^T$ 张量相乘。
* **结果 $\text{分数}$ 的形状：** **$(4, 4)$**。

这个 $4 \times 4$ 的张量矩阵就是**注意力矩阵**，其中的每个数值 $S_{ij}$ 代表了第 $i$ 个词在理解语境时，应该给第 $j$ 个词分配多少“注意力”。

张量在大模型中扮演的角色可以归纳为以下几个方面：
1.  **统一数据格式：** 无论是输入、输出、权重还是中间计算结果，一切都是张量，确保了计算流程的统一性。
2.  **实现并行化：** 它们是专门为 GPU/TPU 等并行计算硬件设计的，使得数百万次的乘法和加法可以在一个指令周期内同时执行，是 LLM 训练和推理的性能基石。
3.  **模型参数存储：** 模型的“知识”——数十亿的权重和偏差，都以巨大的多维张量形式存储在显存中。

### 权重与偏差
### 权重 (Weights) ：信息的“把关人”
在神经网络中，权重是模型学习到的、用来决定输入数据重要性的参数。**权重**的作用是决定一个神经元从上一层接收到的众多信号中，哪些是**重要**的，哪些是**不重要**的。

它们本质上是乘数，应用于输入数据 (x)。一个输入值乘以一个权重 (w) 后，就会进入下一层的神经元。

权重的大小决定了该输入特征对下一层神经元的影响程度。权重越大，表示对应的输入特征越重要，对最终输出的影响也越大。权重越小（接近零），表示该特征不太重要。

在模型训练过程中，算法（如梯度下降）会不断调整这些权重的值，直到模型能够准确地完成任务（例如，正确识别图片中的物体）。

**权重**是模型在训练过程中学到的，用来对输入特征进行**优先排序**和**量化重要性**的参数。它是模型的核心“知识”。

举一个例子，想象你正在做饭（输出），你需要考虑以下输入：

| 输入 ($x$) | 含义 |
| :--- | :--- |
| $x_1$ | 食材的新鲜度 |
| $x_2$ | 厨师的经验 |
| $x_3$ | 餐具的精致度 |

在模型训练之前，这三项对“味道好不好”的贡献是未知的。模型通过学习，会给它们分配**权重** ($w$)：

* **如果模型发现“食材的新鲜度”对味道影响最大**，它会给 $x_1$ 分配一个很高的**权重** $w_1$（比如 $w_1 = 5.0$）。
* **如果模型发现“餐具的精致度”对味道影响很小**，它会给 $x_3$ 分配一个很小的**权重** $w_3$（比如 $w_3 = 0.1$）。

#### 权重在公式中的体现
每个输入值都会被它对应的权重**放大或缩小**，然后所有结果相加：

$$\text{加权输入} = w_1x_1 + w_2x_2 + w_3x_3 + \dots$$

### 偏差 (Biases) ：输出的“校准器”
偏差是模型学习到的另一个参数，通常添加到权重乘以输入值的乘积之和上。

它是一个加数（b），用于平移或调整神经元的激活输出。在计算中，它通常位于激活函数之前。**偏差**的作用是提供一个额外的、**独立于输入**的恒定值，用于调整神经元的激活阈值，让它更容易或更难被“激活”。

**偏差** 允许神经元在所有输入都是零的情况下仍能产生一个非零的激活输出。它赋予了模型更大的灵活性，使其能够更好地拟合数据。如果没有偏差，决策边界（Decision Boundary）将必须穿过原点，这会极大地限制模型的表达能力。

即 **偏差** 允许模型在不改变任何权重的情况下，**平移**（Shift）激活函数曲线。这使得模型能更好地捕捉和拟合数据中的各种模式，是调整**决策边界**的关键。

像权重一样，偏差的值也是在训练过程中不断学习和调整的。
#### 1. 偏差独立于输入 ($+ b$)
沿用做饭的例子：

$$\text{加权输入} = (w_1x_1 + w_2x_2 + w_3x_3)$$

如果所有的输入 $x$ 都很差（都是 0），那么加权输入就是 0。这意味着无论你使用什么激活函数，输出可能总是很低。

**偏差 ($b$) 的引入打破了这个限制：**

* 如果模型学到一个**正的偏差**（$b = +2.0$），它就像是一个“基准分”。即使所有食材 ($x$) 都很普通，最终的味道（输出）也会有一个较高的起始点。这使得神经元更容易被激活。
* 如果模型学到一个**负的偏差**（$b = -2.0$），它就像是一个“惩罚分”。这使得神经元必须接收到非常好的加权输入，才能被激活。

### 偏差公式中的体现
偏差 ($b$) 在加权输入求和之后被简单地**加上**去：

$$\text{神经元的净输入} (z) = \underbrace{(w_1x_1 + w_2x_2 + \dots)}_{\text{加权输入}} + \underbrace{b}_{\text{偏差}}$$

这个 **净输入 $z$** 随后会送入**激活函数**，以产生该神经元的最终输出。

### 32位浮点数(FP32)
**32 位浮点数**（通常简写为 **FP32** 或 **float**）是一种在计算机中表示非整数（带有小数）数值的标准方式。**FP32** 被称为“传统的”或“全精度”格式，它一直以来都是 AI 模型参数的默认标准。

**“32 位”** 指的是用 **32 个二进制位**（0 或 1）来存储一个数字，包括：
* **符号位**（1 位）：决定数字是正数还是负数。
* **指数位**（8 位）：决定小数点的位置。
* **尾数/有效数字位**（23 位）：决定数字的精确值。

之所以提到 **张量，权重，偏差** ，是因为这几个是LLM接收输入到推理，再到输出的核心指标，并且它们都是 **FP32** 格式的。

FP32 提供足够的精度来表示权重和偏差的值。在传统的 AI 模型训练和推理中，**FP32 是标准的数据类型**。

每个权重或偏差都需要 32 位（4 字节）的内存来存储。使用 FP32 进行计算，对硬件资源的消耗（内存带宽和计算能力）较大。

而 **llama.cpp** 为了能在性能较弱的端侧设备上运行大模型，一大修改点就是 **模型的量化** 。

这是 **GGML** 最重要的贡献之一。GGML 支持多种量化策略，允许开发者在模型大小、推理速度和精度之间进行权衡。
### 模型量化
**量化** 是一种技术，可以将模型中通常以 32 位浮点数（FP32）或 16 位浮点数（FP16）存储的权重和激活值，转换为更低精度的格式，如 8 位整数（INT8）、4 位整数（INT4），甚至更低。降低了存储和传输模型的成本。使得更大的模型能够加载到有限的 `RAM` 或 `VRAM` 中。

模型的量化（Model Quantization）是一个在机器学习，尤其是深度学习领域中非常重要的概念，其核心目的是让模型变得**更小、更快、更节能**。

简单来说，**模型的量化就是降低模型中数据的精度**。

一个 `FP32` 的数字需要 4 字节（32位）。一个有数十亿参数的模型需要巨大的内存，**计算开销相当高**，浮点数计算比整数计算更耗时、耗电。

**量化是如何工作的？**

**量化** 就是将这些高精度的浮点数，转换成低精度的**整数**或更小的浮点数，比如：
* **8 位整数 ($INT8$)**
* **4 位整数 ($INT4$ 或 $Q4\_K$)**
* **16 位浮点数 ($FP16$ 或 $BF16$)**

假设你有一个模型的权重值是 `0.785`。在 `FP32` 中，它需要 4 个字节来存储。通过量化，它可以被映射到一个 8 位整数，比如 `200`，同时有一个**缩放因子**（Scale）和**零点**（Zero Point）来帮助在推理时近似还原原来的浮点数。

**量化带来的好处**

| 优势 | 描述 |
| :--- | :--- |
| **模型尺寸减小** | 将 $FP32$（4 字节）量化到 $INT4$（0.5 字节），理论上模型大小可以**缩小 8 倍**。这使得模型可以部署到内存有限的设备上。 |
| **推理速度加快** | 低精度的数据在特定的硬件（如 CPU、移动端 NPU）上可以更快地计算，因为它减少了内存访问，并允许使用更高效的整数运算单元。 |
| **能耗降低** | 更少的计算和更小的内存访问量，意味着模型在运行时消耗的电量更少，这对于移动设备和边缘计算非常重要。 |

量化类型分为两种：

* 训练后量化 (Post-Training Quantization, PTQ)，这是最常见的量化方式。模型在**训练完成后**，才进行量化转换。这种方式不需要重新训练模型，实现简单，成本低。对精度损失不太敏感，或希望快速部署模型的场景。
* 量化感知训练 (Quantization-Aware Training, QAT)，这是在**训练过程中**就模拟量化后的低精度运算。由于模型在训练时就“知道”自己会被量化，因此可以更好地调整参数以**最小化精度损失**，通常能获得比 PTQ 更好的性能。适合对精度要求较高，且愿意投入额外训练时间的场景。

在移动设备（如 Android 手机）上部署深度学习模型时，**$INT8$ 量化**是常用的优化手段，因为移动设备的内存和计算资源都相对有限。
## llama.cpp 框架特点
### 轻量与高效的 C/C++ 实现
`llama.cpp` 完全使用 C 和 C++ 编写，不依赖于大型的深度学习框架（如 TensorFlow 或 PyTorch）的运行时库。这意味着它编译出来的程序**体积非常小，运行时内存占用也低**。这对于资源受限的移动设备来说至关重要。

除了底层的 BLAS (基本线性代数子程序库) 或特定硬件的计算库（如 CUDA、Metal）， `llama.cpp` 几乎没有其他复杂的外部依赖。这使得它非常容易编译和部署。

`llama.cpp` 对 CPU 进行了大量底层优化，利用了各种 CPU 指令集（如 x86-64 上的 AVX/AVX2/AVX-512，以及 ARM 上的 Neon）。这让 LLMs 可以在没有独立 GPU 的设备上，仅仅依靠 CPU 也能获得令人惊讶的推理速度。
### 广泛的硬件加速支持
除了强大的 CPU 优化，llama.cpp 还支持多种 GPU 和专用硬件加速，这意味着它是**跨平台的**，跨平台支持在任何行业都备受推崇，无论是游戏、人工智能还是其他类型的软件。赋予开发者在他们想要的系统和环境中运行软件所需的自由永远不是一件坏事。

* **Apple Silicon (Metal)**：针对苹果 M 系列芯片的 Metal API 进行了优化，充分利用了其强大的统一内存架构和神经网络引擎。
* **NVIDIA GPU (CUDA)**：支持 NVIDIA 显卡，通过 CUDA 加速推理，性能非常出色。
* **AMD GPU (hipBLAS)**：兼容 AMD GPU。
* **Intel GPU (SYCL/oneAPI)**：支持英特尔的集成和独立显卡。
* **通用 GPU 后端 (Vulkan/OpenCL)**：这对于 Android 开发者尤为重要。通过 **OpenCL** 后端，llama.cpp 能够利用 Android 设备中 SoC（System on Chip）内置的 GPU 或 DSP（数字信号处理器，如高通骁龙的 Hexagon DSP）进行加速，将推理负载从 CPU 转移到更擅长并行计算的硬件上。
* **CPU + GPU 混合推理**：对于一些较大的模型，即使单张 GPU 的显存不足以容纳整个模型，llama.cpp 也能将模型的某些层加载到 GPU 上运行，其余部分则在 CPU 上运行，实现资源的有效利用。

### GGUF 文件格式
#### GGML
**GGML** 是一种为机器学习设计的**文件格式和张量库**，它最初由开发者 Georgi Gerganov 创建（GGML = Gerganov's General Machine Learning）。它的核心目标是**高效地在 CPU 上运行大型机器学习模型**，特别是大型语言模型（LLMs），并且支持各种硬件平台。

GGML 的出现，是将 LLMs 带到消费级硬件上的关键一步。在它之前，运行大型模型通常需要昂贵的 GPU 和复杂的配置。GGML 通过一系列创新技术，让这些模型能够在普通的个人电脑上也能跑起来。

GGML 的核心是用 C 语言编写的，这意味着它的 **执行效率非常高** ，并且**依赖性极少**。这使得它非常轻量级，可以在各种不同的操作系统和硬件上轻松编译和运行，包括 macOS、Linux、Windows，甚至 iOS 和 Android。

GGML 最初的重点在于**在 CPU 上实现高效推理**。它利用了现代 CPU 的特性，例如 **SIMD（单指令多数据）指令集** ，比如 Intel 的 AVX、AVX2、AVX512 和 ARM 的 NEON，这些指令允许 CPU 同时处理多个数据点，从而加速矩阵乘法等并行计算。可以利用 CPU 的多个核心并行执行计算任务。
#### GGUF格式的出现
**GGUF (GPT-Generated Unified Format)** 就是 GGML 文件格式的**最新演进版本**。GGUF 在 GGML 的基础上，提供了更强的灵活性、更好的向后兼容性，并能包含更多的元数据（例如分词器信息、提示模板等），使其成为目前社区首选的本地 LLM 格式。

可以把它想象成一个 **包含了模型“大脑”里所有知识的“盒子”** ，这个盒子设计得非常紧凑和高效，便于在各种设备上快速打开和使用。

GGUF 格式支持从全精度 (FP32) 到半精度 (FP16) 以及多种低精度（如 INT8、INT5、INT4、INT2）的**模型量化**。量化可以在牺牲极小精度损失的情况下，大幅**减小模型体积，降低内存占用和计算需求**。这对于在手机上运行大型模型至关重要，能让原本无法加载的模型变得可用。

GGUF 文件不仅包含模型权重，还打包了模型的词表、超参数、架构信息和特殊 token ID 等所有运行所需的元数据。一个 GGUF 文件就是一个独立的、可运行的模型包。

此外，GGUF 支持内存映射。这意味着操作系统可以直接将文件内容映射到内存中，而无需将整个模型完全复制到 RAM。这大大加快了模型加载速度，并允许在物理内存不足的情况下也能运行大模型（通过操作系统的虚拟内存管理）。
#### 模型文件命名约定
GGUF 遵循命名约定， `<BaseName><SizeLabel><FineTune><Version><Encoding><Type><Shard>.gguf` 每个组件之间用 分隔（-如果存在）。这样做的最终目的是让人们能够一目了然地了解模型的最重要细节。

这些组件包括：
* BaseName：模型基础类型或架构的描述性名称。
* SizeLabel：参数权重类（对排行榜有用）表示为`<expertCount>x<count><scale-prefix>` 
* FineTune：模型微调目标的描述性名称（例如聊天、指导等...）
* 版本：（可选）表示模型版本号，格式为`v<Major>.<Minor>`
* 编码：表示应用于模型的权重编码方案。内容、类型组合和排列由用户代码决定，并可能根据项目需求而变化。
* 类型：表示 gguf 文件的类型及其预期用途
* Shard：（可选）表示并表明模型已被拆分为多个分片，格式为`<ShardNum>-of-<ShardTotal>`。

例如：**Mixtral-8x7B-v0.1-KQ2.gguf**：

```
型号名称：Mixtral
专家数量：8
参数数量：7B
版本号：v0.1
权重编码方案：KQ2
```

#### GGUF 文件的结构
一个 GGUF 文件大致可以分为两个主要部分：

1.  **头部 (Header)**：包含 GGUF 文件的魔数（用于识别文件类型）、版本号以及一些全局元数据（如模型总层数、维度等）。
2.  **元数据 (Metadata) 和张量 (Tensors)**：
    * **KV 对 (Key-Value Pairs)**：存储了模型的各种超参数、架构信息、词表和一些其他配置。这些都是以键值对的形式存储的，方便访问。
    * **张量数据 (Tensor Data)**：这是模型真正的权重数据。每个张量都会有其名称、维度和数据类型（例如，量化后的 INT4、INT8 或 FP16 等）。这些数据会按照特定的对齐方式存储，以确保高效读取。

![](/assets/img/blog/blogs_gguf_file_structure.png)

## 部署运行实操
接下来介绍下如何在项目中集成llama.cpp，从而加载gguf格式的模型，运行本地的小模型。
### 一、使用Termux命令行编译运行
这种方法就是将 `Android` 设备当作 `Linux` 设备来使用，手机需要安装Termux。

可以在Github Releases 选择 `termux-app_v0.118.2+github-debug_arm64-v8a.apk` 下载，并且安装到手机。

[Releases · termux/termux-app](https://github.com/termux/termux-app/releases)

下载 `llama.cpp` 库:

```
# 切换国内源
termux-change-repo
apt list --upgradable
# 安装依赖工具
pkg install -y cmake git build-essential
# 下载 llama.cpp
git clone https://github.com/ggml-org/llama.cpp.git
# 如果git下不下来，通过scp拷贝进去
scp -P 8022 .\llama.cpp-master.zip u0_a456@192.168.31.44:~
```

![](/assets/img/blog/blogs_ai_termux_clone_llamacpp.png)

编译llama.cpp源代码：

```
# 进入目录
cd llama.cpp
# 创建build文件,并且进入文件夹
mkdir build && cd build
# 生成编译配置，-DGGML_CUDA=OFF 关闭GPU
cmake .. -DGGML_CUDA=OFF 
# 4个线程编译
make -j4
# 编译完成目录在/bin
ls ~/llama.cpp/build/bin
# bin添加到环境变量中
echo 'export PATH=$PATH:~/llama.cpp/build/bin/' >> ~/.bashrc
source ~/.bashrc
```

![](/assets/img/blog/blogs_ai_termux_build_llamacpp.png)

直接从 `Hugging Face` 下载 `.gguf` 文件，避免转换步骤。我下载的是 `DeepSeek-R1-Distill-Qwen-1.5B-Q2_K.gguf` 。网络问题，推荐在电脑端下载完毕，通过USB使用adb或者文件模式，推送到手机端。

注意Termux默认是无法操作手机文件系统的，需要执行命令来获取权限，初始化文件管理系统。

```
termux-setup-storage
```

![](/assets/img/blog/blogs_ai_termux_get_file_permissions.png)

出现管理所有文件的权限授予弹窗，打开之后将文件复制到内部目录：

![](/assets/img/blog/blogs_ai_termux_copy_model_gguf_file.png)

可以先试试运行效果，使用 `llama-cli` 直接在命令行中启动：

```
llama-cli -m DeepSeek-R1-Distill-Qwen-1.5B-Q2_K.gguf
```

> llama-cli，即 ​​CLI 模式​​（Command-Line Interface 模式）是指通过命令行直接运行模型进行推理（文本生成）的方式，而不是通过 API 或图形界面。这是 llama.cpp 最基础的使用方式，适合本地测试、脚本调用或服务器部署。

运行效果如下：

![](/assets/img/blog/blogs_ai_termux_run_deepseek_model.png)

也可以使用 `llama-server` 的方式启动：

```
llama-server -m DeepSeek-R1-Distill-Qwen-1.5B-Q2_K.gguf --port 8080 --host 0.0.0.0
```

![](/assets/img/blog/blogs_ai_termux_run_deepseek_model_as_server.png)

#### 手机端client直接访问
通过 `llama-server` 作为服务器启动之后，我们可以直接**在手机端**编写client，通过http请求来访问，直接和这个服务交互。这里大部分可以复用之前写的和**deepseek官方api**的请求逻辑。

网络Repository代码：

```kotlin
class DeepseekChatRepository(private val ktorClient: KtorClient) {

    companion object {
        const val BASE_URL =
            "https://api.deepseek.com"
        const val LOCAL_SERVER = "http://0.0.0.0:8080/v1"
        const val COMMON_SYSTEM_PROMT = "你是一个人工智能系统，可以根据用户的输入来返回生成式的回复"
        const val ENGLISH_SYSTEM_PROMT =
            "You are a English teacher, you can help me improve my English skills, please answer my questions in English."
        const val API_KEY = "xxxxxxxxxxxxxxxxx"
        const val MODEL_NAME = "deepseek-chat"
    }

    suspend fun localLLMChat(chat: String) = withContext(Dispatchers.IO) {
        ktorClient.client.post("${LOCAL_SERVER}/chat/completions") {
            // 配置请求头
            headers {
                append("Content-Type", "application/json")
            }
            setBody(
                DeepSeekRequestBean(
                    model = "DeepSeek-R1-Distill-Qwen-1.5B-Q2_K",
                    max_tokens = 256,
                    temperature = 0.7f,
                    stream = false,
                    messages = listOf(
                        RequestMessage(COMMON_SYSTEM_PROMT, ChatRole.SYSTEM.roleDescription),
                        RequestMessage(chat, ChatRole.USER.roleDescription)
                    )
                )
            )
        }.body<LocalModelResult>()
    }
}
```

界面上维护一个chatListState，里面是一个

```kotlin
data class AiChatUiState(
    val chatList: List<ChatItem> = listOf(),
    val listSize: Int = chatList.size
) {
    fun toUiState() = AiChatUiState(chatList = chatList, listSize = listSize)
}

data class ChatItem(
    val content: String,
    val role: ChatRole,
)
```

界面观察这个State响应式刷新即可。

运行结果：

![](/assets/img/blog/blogs_local_llm_server_client.png){:width="300" height="620" loading="lazy"}

#### 局域网内其他设备访问
除了同一设备直接访问本地服务，在同一个局域网中，比如电脑端，我们也可以使用Python，通过 `openai` 的Python开发套件，和手机端运行的服务进行通信：

```python
import requests
import json
import time

API_URL = "http://192.168.31.44:8080/v1/chat/completions"

payload = {
    "model": "DeepSeek-R1-Distill-Qwen-1.5B-Q2_K",  # llama-server 中可随意写
    "messages": [
        {"role": "system", "content": "你是一个英语学习助手。"},
        {"role": "user", "content": "请用中文解释单词 ability 的含义，并给出一个英文例句。"}
    ],
    "temperature": 0.7,
    "max_tokens": 256,
    "stream": False
}

# 记录开始时间
start_time = time.time()

# 发送请求
response = requests.post(API_URL, headers={"Content-Type": "application/json"}, data=json.dumps(payload))

# 记录结束时间
end_time = time.time()

if response.ok:
    result = response.json()
    message = result['choices'][0]['message']['content']
    print("模型回复：\n", message)
    
    # 处理 token usage 和速度统计
    usage = result.get("usage", {})
    total_tokens = usage.get("total_tokens", "未知")
    elapsed = end_time - start_time
    
    print(f"\n总 tokens: {total_tokens}")
    print(f"耗时: {elapsed:.2f} 秒")
    if isinstance(total_tokens, int) and elapsed > 0:
        print(f"生成速度: {total_tokens / elapsed:.2f} tokens/秒")
else:
    print("请求失败，状态码：", response.status_code)
    print(response.text)
```

### 二、单进程集成方案
上面那种在Termux中运行模型的方式还是感觉比较麻烦，每次也需要手动开启服务。

下面这种方案就是比较符合 `Android` 设备上运行的直观预期，通过一个APP页面来承载功能，在一个应用中，以用户友好的 `UX交互` 来和本地模型进行通信。使用JNI开发接口和llama.cpp交互。

![](/assets/img/blog/blogs_ai_llamacpp_smollchat.png){:width="300" height="620" loading="lazy"}

底层依然是使用 `llama.cpp` 加载和执行 GGUF 模型。

由于 llama.cpp 是用纯 C/C++ 编写的，因此很容易利用 AndroidStudio 的NDK工具，和应用app一起编译运行。

首先，定义JNI函数，第一步需要加载 `gguf` 文件。在 Android 应用中，需要使用 Kotlin 语言来定义页面需要用到的接口，再到 Native 层使用 `llama.cpp` 的能力，来编写 C++ 的桥接代码。

```kotlin
class GGUFReader {
    companion object {
        init {
            System.loadLibrary("ggufreader")
        }
    }

    private var nativeHandle: Long = 0L

    suspend fun load(modelPath: String) =
        withContext(Dispatchers.IO) {
            nativeHandle = getGGUFContextNativeHandle(modelPath)
        }

    fun getContextSize(): Long? {
        assert(nativeHandle != 0L) { "Use GGUFReader.load() to initialize the reader" }
        val contextSize = getContextSize(nativeHandle)
        return if (contextSize == -1L) {
            null
        } else {
            contextSize
        }
    }

    fun getChatTemplate(): String? {
        assert(nativeHandle != 0L) { "Use GGUFReader.load() to initialize the reader" }
        val chatTemplate = getChatTemplate(nativeHandle)
        return chatTemplate.ifEmpty {
            null
        }
    }

    private external fun getGGUFContextNativeHandle(modelPath: String): Long

    private external fun getContextSize(nativeHandle: Long): Long

    private external fun getChatTemplate(nativeHandle: Long): String
}
```

`nativeHandle` 是一个长整型（Long）变量，代表指向本地（C/C++）端创建的 gguf_context 的指针。在 Native 代码里，gguf_context 是一个上下文对象，负责管理 GGUF 文件的读取操作。nativeHandle 唯一标识这个上下文对象，方便 Kotlin 代码引用。借助 nativeHandle 能把本地对象的地址传递给 Kotlin 代码，进而在 Kotlin 代码里调用本地函数操作这些对象。

定义的三个JNI方法作用分别如下：
* `getGGUFContextNativeHandle()` ： 加载模型文件，返回模型上下文的指针。
* `getContextSize()` ： 获取模型上下文的大小，即模型参数的数量。
* `getChatTemplate()` ： 获取模型的聊天模板，用于生成聊天对话的提示。

在Native层的代码中，实现也非常简单，引入 `llama.cpp` 中的 `gguf.h` 头文件。

在获取上下文指针的方法中，传入模型文件的绝对地址字符串，调用 `gguf_init_from_file` ，即可获取到 `gguf_context` 对象指针，转换回 `jlong` 类型传递给Kotlin即可。

第二，在获取模型参数数量的方法中，需要先从 `gguf_context` 中找到 `architecture` 字段，再根据 `architecture` 字段的值，拼接出 `context_length` 字段的名称，最后调用 `gguf_get_val_u32` 方法获取参数数量。

第三个方法是获取模型的聊天模板，需要先找到分词器 `tokenizer.chat_template` 字段，调用 `gguf_get_val_str` 方法获取字符串值。

```cpp
#include "gguf.h"
#include <jni.h>
#include <string>

/**
 * @brief 获取 GGUF 上下文的本地句柄。
 *
 * 该函数通过 Java Native Interface (JNI) 从 Java 层调用，用于根据给定的模型文件路径
 * 初始化 GGUF 上下文，并返回其本地句柄。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPath 包含 GGUF 模型文件路径的 Java 字符串对象。
 * @return 指向初始化后的 GGUF 上下文的 jlong 类型句柄，若初始化失败可能为 nullptr 对应的 jlong 值。
 */
extern "C" JNIEXPORT jlong JNICALL
Java_com_stephen_llamacppbridge_GgufFileReader_getGGUFContextNativeHandle(JNIEnv *env, jobject thiz,
                                                                          jstring modelPath) {
    jboolean isCopy = true;
    const char *modelPathCStr = env->GetStringUTFChars(modelPath, &isCopy);
    // 初始化 GGUF 上下文所需的参数，不分配额外内存，上下文指针初始化为 nullptr
    gguf_init_params initParams = {.no_alloc = true, .ctx = nullptr};
    // 根据模型文件路径和初始化参数创建 GGUF 上下文
    gguf_context *ggufContext = gguf_init_from_file(modelPathCStr, initParams);
    env->ReleaseStringUTFChars(modelPath, modelPathCStr);
    return reinterpret_cast<jlong>(ggufContext);
}

/**
 * @brief 获取 GGUF 模型的上下文大小。
 *
 * 该函数通过 JNI 从 Java 层调用，根据给定的 GGUF 上下文本地句柄，
 * 查找并返回模型的上下文大小。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param nativeHandle 指向 GGUF 上下文的本地句柄。
 * @return 模型的上下文大小，若查找失败则返回 -1。
 */
extern "C" JNIEXPORT jlong JNICALL
Java_com_stephen_llamacppbridge_GgufFileReader_getContextSize(JNIEnv *env, jobject thiz,
                                                              jlong nativeHandle) {
    // 将 jlong 类型的本地句柄转换为 gguf_context 指针
    gguf_context *ggufContext = reinterpret_cast<gguf_context *>(nativeHandle);
    // 查找模型架构信息对应的键 ID
    int64_t architectureKeyId = gguf_find_key(ggufContext, "general.architecture");
    // 若未找到架构信息键 ID，返回 -1
    if (architectureKeyId == -1)
        return -1;
    // 获取模型架构信息
    std::string architecture = gguf_get_val_str(ggufContext, architectureKeyId);
    // 构建上下文长度信息对应的键名
    std::string contextLengthKey = architecture + ".context_length";
    // 查找上下文长度信息对应的键 ID
    int64_t contextLengthKeyId = gguf_find_key(ggufContext, contextLengthKey.c_str());
    // 若未找到上下文长度信息键 ID，返回 -1
    if (contextLengthKeyId == -1)
        return -1;
    uint32_t contextLength = gguf_get_val_u32(ggufContext, contextLengthKeyId);
    return contextLength;
}

/**
 * @brief 获取 GGUF 模型的聊天模板。
 *
 * 该函数通过 JNI 从 Java 层调用，根据给定的 GGUF 上下文本地句柄，
 * 查找并返回模型的聊天模板。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param nativeHandle 指向 GGUF 上下文的本地句柄。
 * @return 包含聊天模板的 Java 字符串对象，若未找到则返回空字符串对应的 Java 字符串。
 */
extern "C" JNIEXPORT jstring JNICALL
Java_com_stephen_llamacppbridge_GgufFileReader_getChatTemplate(JNIEnv *env, jobject thiz,
                                                               jlong nativeHandle) {
    // 将 jlong 类型的本地句柄转换为 gguf_context 指针
    gguf_context *ggufContext = reinterpret_cast<gguf_context *>(nativeHandle);
    // 查找聊天模板信息对应的键 ID
    int64_t chatTemplateKeyId = gguf_find_key(ggufContext, "tokenizer.chat_template");
    // 存储聊天模板的字符串
    std::string chatTemplate;
    // 若未找到聊天模板信息键 ID，将聊天模板设为空字符串
    if (chatTemplateKeyId == -1) {
        chatTemplate = "";
    } else {
        // 若找到聊天模板信息键 ID，获取聊天模板信息
        chatTemplate = gguf_get_val_str(ggufContext, chatTemplateKeyId);
    }
    // 将 C++ 字符串转换为 Java 字符串并返回
    return env->NewStringUTF(chatTemplate.c_str());
}
```

然后根据 `llama.cpp` 的几个核心的方法，如加载，对话等功能，来编写对接的接口 `C++` 类。

头文件定义如下：

```cpp
/**
管理大语言模型（LLM）的推理过程，包括模型加载、聊天消息处理、推理循环等功能
 */
class LLMInference {
    // llama.cpp 特定类型的成员变量
    /// llama 上下文指针，用于管理模型的运行时状态
    llama_context *_ctx;
    /// llama 模型指针，指向加载的大语言模型
    llama_model *_model;
    /// llama 采样器指针，用于从模型输出中采样生成下一个 token
    llama_sampler *_sampler;
    /// 当前采样得到的 llama token
    llama_token _currToken;
    /// llama 批处理结构，用于批量处理输入 token
    llama_batch _batch;

    // 存储聊天中用户/助手消息的容器
    /// 存储聊天过程中用户和助手的消息列表
    std::vector<llama_chat_message> _messages;
    // 存储将聊天模板应用到 `_messages` 中所有消息后生成的字符串
    /// 存储应用聊天模板后的格式化消息
    std::vector<char> _formattedMessages;
    // 存储追加到 `_messages` 中的最后一个查询的 token
    /// 存储最后一次查询的 token 列表
    std::vector<llama_token> _promptTokens;
    /// 上一次格式化消息的长度
    int _prevLen = 0;
    /// 聊天模板字符串指针
    const char *_chatTemplate;

    // 存储给定查询的完整响应
    /// 存储当前查询的完整响应内容
    std::string _response;
    /// 缓存响应的 token 片段
    std::string _cacheResponseTokens;
    // 是否在 `_messages` 中缓存先前的消息
    /// 是否存储聊天历史消息的标志
    bool _storeChats;

    // 响应生成指标
    /// 记录响应生成所花费的总时间（微秒）
    int64_t _responseGenerationTime = 0;
    /// 记录生成的 token 总数
    long _responseNumTokens = 0;

    // 对话过程中消耗的上下文窗口长度
    /// 记录对话过程中已使用的上下文大小
    int _nCtxUsed = 0;

    bool _isValidUtf8(const char *response);

public:
    void loadModel(const char *modelPath, float minP, float temperature, bool storeChats,
                   long contextSize,
                   const char *chatTemplate, int nThreads, bool useMmap, bool useMlock);

    void addChatMessage(const char *message, const char *role);

    float getResponseGenerationTime() const;

    int getContextSizeUsed() const;

    void startCompletion(const char *query);

    std::string completionLoop();

    void stopCompletion();

    ~LLMInference();
};
```

cpp实现：

```cpp
#include "LLMInference.h"
#include "llama.h"
#include "gguf.h"
#include <android/log.h>
#include <cstring>
#include <iostream>

// 定义日志标签，用于在 Android 日志系统中标识本模块的日志
#define TAG "[SmolLMAndroid-Cpp]"
// 定义信息日志宏，方便打印信息级别的日志
#define LOGi(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
// 定义错误日志宏，方便打印错误级别的日志
#define LOGe(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)

// 函数声明：对输入文本进行分词处理
// vocab: 指向 llama 词表的指针
// text: 待分词的字符串
// add_special: 是否添加特殊标记
// parse_special: 是否解析特殊标记，默认值为 false
std::vector<llama_token> common_tokenize(
        const struct llama_vocab *vocab,
        const std::string &text,
        bool add_special,
        bool parse_special = false);

// 函数声明：将 llama 标记转换为对应的词块
// ctx: 指向 llama 上下文的指针
// token: llama 标记
// special: 是否为特殊标记，默认值为 true
std::string common_token_to_piece(
        const struct llama_context *ctx,
        llama_token token,
        bool special = true);

/**
 * @brief 加载 LLM 模型并初始化相关参数
 *
 * 该函数负责根据传入的参数加载 LLM 模型，创建模型上下文、采样器等，
 * 并对这些组件进行初始化配置。
 *
 * @param model_path 模型文件的路径
 * @param minP 采样时使用的最小概率阈值
 * @param temperature 采样时使用的温度参数
 * @param storeChats 是否存储聊天记录
 * @param contextSize 模型的上下文大小
 * @param chatTemplate 聊天模板字符串
 * @param nThreads 推理时使用的线程数量
 * @param useMmap 是否使用内存映射来加载模型
 * @param useMlock 是否使用内存锁定
 */
void
LLMInference::loadModel(const char *model_path, float minP, float temperature, bool storeChats,
                        long contextSize,
                        const char *chatTemplate, int nThreads, bool useMmap, bool useMlock) {
    // 打印加载模型时使用的各项参数，方便调试和日志记录
    LOGi("loading model with"
         "\n\tmodel_path = %s"
         "\n\tminP = %f"
         "\n\ttemperature = %f"
         "\n\tstoreChats = %d"
         "\n\tcontextSize = %li"
         "\n\tchatTemplate = %s"
         "\n\tnThreads = %d"
         "\n\tuseMmap = %d"
         "\n\tuseMlock = %d",
         model_path, minP, temperature, storeChats, contextSize, chatTemplate, nThreads, useMmap,
         useMlock);

    // 创建一个 llama_model_params 实例，并使用默认参数初始化
    llama_model_params model_params = llama_model_default_params();
    // 设置是否使用内存映射加载模型
    model_params.use_mmap = useMmap;
    // 设置是否使用内存锁定
    model_params.use_mlock = useMlock;
    // 从指定路径加载模型
    _model = llama_model_load_from_file(model_path, model_params);

    // 检查模型是否加载成功
    if (!_model) {
        // 若加载失败，打印错误日志
        LOGe("failed to load model from %s", model_path);
        // 抛出运行时错误异常
        throw std::runtime_error("loadModel() failed");
    }

    // 创建一个 llama_context_params 实例，并使用默认参数初始化
    llama_context_params ctx_params = llama_context_default_params();
    // 设置模型的上下文大小
    ctx_params.n_ctx = contextSize;
    // 设置推理时使用的线程数量
    ctx_params.n_threads = nThreads;
    // 禁用性能指标统计
    ctx_params.no_perf = true;
    // 基于加载的模型初始化 llama 上下文
    _ctx = llama_init_from_model(_model, ctx_params);

    // 检查上下文是否创建成功
    if (!_ctx) {
        // 若创建失败，打印错误日志
        LOGe("llama_new_context_with_model() returned null)");
        // 抛出运行时错误异常
        throw std::runtime_error("llama_new_context_with_model() returned null");
    }

    // 初始化采样器参数，使用默认参数
    llama_sampler_chain_params sampler_params = llama_sampler_chain_default_params();
    // 禁用采样器的性能指标统计
    sampler_params.no_perf = true;
    // 初始化采样器链
    _sampler = llama_sampler_chain_init(sampler_params);
    // 向采样器链中添加最小概率采样器
    llama_sampler_chain_add(_sampler, llama_sampler_init_min_p(minP, 1));
    // 向采样器链中添加温度采样器
    llama_sampler_chain_add(_sampler, llama_sampler_init_temp(temperature));
    // 向采样器链中添加分布采样器，使用默认种子
    llama_sampler_chain_add(_sampler, llama_sampler_init_dist(LLAMA_DEFAULT_SEED));

    // 初始化格式化消息缓冲区，大小为模型的上下文大小
    _formattedMessages = std::vector<char>(llama_n_ctx(_ctx));
    // 清空消息列表
    _messages.clear();
    // 复制聊天模板字符串
    _chatTemplate = strdup(chatTemplate);
    // 设置是否存储聊天记录
    this->_storeChats = storeChats;
}

/**
 * @brief 向聊天消息列表中添加一条消息
 *
 * @param message 消息内容
 * @param role 消息角色，如 "user" 或 "assistant"
 */
void
LLMInference::addChatMessage(const char *message, const char *role) {
    // 将消息角色和内容复制后添加到消息列表中
    _messages.push_back({strdup(role), strdup(message)});
}

/**
 * @brief 获取响应生成的速度
 *
 * 计算方式为生成的 token 数量除以生成所用的总时间（秒）
 *
 * @return float 响应生成速度，单位为 token/秒
 */
float
LLMInference::getResponseGenerationTime() const {
    return (float) _responseNumTokens / (_responseGenerationTime / 1e6);
}

/**
 * @brief 获取当前已使用的上下文大小
 *
 * @return int 当前已使用的上下文大小
 */
int
LLMInference::getContextSizeUsed() const {
    return _nCtxUsed;
}

/**
 * @brief 开始完成任务，处理用户输入并准备推理
 *
 * @param query 用户输入的查询内容
 */
void
LLMInference::startCompletion(const char *query) {
    // 如果不存储聊天记录，则重置相关状态
    if (!_storeChats) {
        _prevLen = 0;
        _formattedMessages.clear();
        _formattedMessages = std::vector<char>(llama_n_ctx(_ctx));
    }
    // 重置响应生成时间和生成的 token 数量
    _responseGenerationTime = 0;
    _responseNumTokens = 0;
    // 将用户查询添加到聊天消息列表中
    addChatMessage(query, "user");
    // 应用聊天模板，格式化聊天消息
    int newLen = llama_chat_apply_template(_chatTemplate, _messages.data(), _messages.size(), true,
                                           _formattedMessages.data(), _formattedMessages.size());
    // 检查格式化后的消息长度是否超过缓冲区大小
    if (newLen > (int) _formattedMessages.size()) {
        // 若超过，则调整缓冲区大小
        _formattedMessages.resize(newLen);
        // 重新应用聊天模板
        newLen = llama_chat_apply_template(_chatTemplate, _messages.data(), _messages.size(), true,
                                           _formattedMessages.data(), _formattedMessages.size());
    }
    // 检查聊天模板应用是否失败
    if (newLen < 0) {
        // 若失败，抛出运行时错误异常
        throw std::runtime_error(
                "llama_chat_apply_template() in LLMInference::startCompletion() failed");
    }
    // 提取格式化后的提示信息
    std::string prompt(_formattedMessages.begin() + _prevLen, _formattedMessages.begin() + newLen);
    // 对提示信息进行分词处理
    _promptTokens = common_tokenize(llama_model_get_vocab(_model), prompt, true, true);

    // 创建一个 llama_batch 实例，包含单个序列
    // 详情可参考 llama_batch_init 函数
    _batch.token = _promptTokens.data();
    _batch.n_tokens = _promptTokens.size();
}

// 代码来源：
// https://github.com/ggerganov/llama.cpp/blob/master/examples/llama.android/llama/src/main/cpp/llama-android.cpp#L38
/**
 * @brief 检查输入的字符串是否为有效的 UTF-8 编码
 *
 * @param response 待检查的字符串
 * @return bool 若为有效 UTF-8 编码返回 true，否则返回 false
 */
bool
LLMInference::_isValidUtf8(const char *response) {
    // 若输入为空指针，认为是有效的 UTF-8 编码
    if (!response) {
        return true;
    }
    // 将输入字符串转换为无符号字符指针
    const unsigned char *bytes = (const unsigned char *) response;
    int num;
    // 遍历字符串中的每个字节
    while (*bytes != 0x00) {
        if ((*bytes & 0x80) == 0x00) {
            // U+0000 到 U+007F 的单字节编码
            num = 1;
        } else if ((*bytes & 0xE0) == 0xC0) {
            // U+0080 到 U+07FF 的双字节编码
            num = 2;
        } else if ((*bytes & 0xF0) == 0xE0) {
            // U+0800 到 U+FFFF 的三字节编码
            num = 3;
        } else if ((*bytes & 0xF8) == 0xF0) {
            // U+10000 到 U+10FFFF 的四字节编码
            num = 4;
        } else {
            // 不符合 UTF-8 编码规则，返回 false
            return false;
        }

        bytes += 1;
        // 检查后续的续字节是否符合 UTF-8 编码规则
        for (int i = 1; i < num; ++i) {
            if ((*bytes & 0xC0) != 0x80) {
                // 续字节不符合规则，返回 false
                return false;
            }
            bytes += 1;
        }
    }
    // 所有字节都符合 UTF-8 编码规则，返回 true
    return true;
}

/**
 * @brief 完成任务的循环函数，进行模型推理和响应生成
 *
 * @return std::string 生成的有效 UTF-8 词块，若无效则返回空字符串，若生成结束则返回 "[EOG]"
 */
std::string
LLMInference::completionLoop() {
    // 检查输入的长度是否超过模型的上下文大小
    uint32_t contextSize = llama_n_ctx(_ctx);
    _nCtxUsed = llama_kv_self_used_cells(_ctx);
    if (_nCtxUsed + _batch.n_tokens > contextSize) {
        // 若超过，抛出运行时错误异常
        throw std::runtime_error("context size reached");
    }

    // 记录模型推理开始时间
    auto start = ggml_time_us();
    // 运行模型进行解码
    if (llama_decode(_ctx, _batch) < 0) {
        // 若解码失败，抛出运行时错误异常
        throw std::runtime_error("llama_decode() failed");
    }

    // 采样一个 token，并检查是否为生成结束标记
    _currToken = llama_sampler_sample(_sampler, _ctx, -1);
    if (llama_vocab_is_eog(llama_model_get_vocab(_model), _currToken)) {
        // 若为生成结束标记，将响应添加到聊天消息列表中
        addChatMessage(strdup(_response.data()), "assistant");
        // 清空响应缓冲区
        _response.clear();
        // 返回生成结束标记
        return "[EOG]";
    }
    // 将采样的 token 转换为对应的词块
    std::string piece = common_token_to_piece(_ctx, _currToken, true);
    // 打印转换后的词块信息
    LOGi("common_token_to_piece: %s", piece.c_str());
    // 记录模型推理结束时间
    auto end = ggml_time_us();
    // 累加响应生成时间
    _responseGenerationTime += (end - start);
    // 累加生成的 token 数量
    _responseNumTokens += 1;
    // 将生成的词块添加到缓存中
    _cacheResponseTokens += piece;

    // 重新初始化 batch，使用新生成的 token
    _batch.token = &_currToken;
    _batch.n_tokens = 1;

    // 检查缓存中的词块是否为有效的 UTF-8 编码
    if (_isValidUtf8(_cacheResponseTokens.c_str())) {
        // 若有效，将其添加到响应缓冲区中
        _response += _cacheResponseTokens;
        std::string valid_utf8_piece = _cacheResponseTokens;
        // 清空缓存
        _cacheResponseTokens.clear();
        // 返回有效的 UTF-8 词块
        return valid_utf8_piece;
    }

    // 若无效，返回空字符串
    return "";
}

/**
 * @brief 停止完成任务，处理收尾工作
 */
void
LLMInference::stopCompletion() {
    // 如果存储聊天记录，将响应添加到聊天消息列表中
    if (_storeChats) {
        addChatMessage(_response.c_str(), "assistant");
    }
    // 清空响应缓冲区
    _response.clear();
    // 获取模型的聊天模板
    const char *tmpl = llama_model_chat_template(_model, nullptr);
    // 应用聊天模板，计算格式化后的消息长度
    _prevLen = llama_chat_apply_template(tmpl, _messages.data(), _messages.size(), false, nullptr,
                                         0);
    // 检查聊天模板应用是否失败
    if (_prevLen < 0) {
        // 若失败，抛出运行时错误异常
        throw std::runtime_error(
                "llama_chat_apply_template() in LLMInference::stopCompletion() failed");
    }
}

/**
 * @brief LLMInference 类的析构函数，释放相关资源
 */
LLMInference::~LLMInference() {
    // 打印析构信息
    LOGi("deallocating LLMInference instance");
    // 释放聊天消息列表中动态分配的内存
    for (llama_chat_message &message: _messages) {
        free(const_cast<char *>(message.role));
        free(const_cast<char *>(message.content));
    }
    // 释放聊天模板动态分配的内存
    free(const_cast<char *>(_chatTemplate));
    // 释放采样器资源
    llama_sampler_free(_sampler);
    // 释放 llama 上下文资源
    llama_free(_ctx);
    // 释放 llama 模型资源
    llama_model_free(_model);
}
```

接下来还要定义外部封装的方法，类似GGUFReader，获取一个模型的指针，以此来访问：

```kotlin
  /**
     * 加载模型
     */
    private external fun loadModel(
        modelPath: String,
        minP: Float,
        temperature: Float,
        storeChats: Boolean,
        contextSize: Long,
        chatTemplate: String,
        nThreads: Int,
        useMmap: Boolean,
        useMlock: Boolean,
    ): Long

    /**
     * 添加聊天消息
     */
    private external fun addChatMessage(
        modelPtr: Long,
        message: String,
        role: String,
    )

    /**
     * 获取响应生成速度
     */
    private external fun getResponseGenerationSpeed(modelPtr: Long): Float

    /**
     * 获取上下文使用大小
     */
    private external fun getContextSizeUsed(modelPtr: Long): Int

    /**
     * 关闭模型
     */
    private external fun close(modelPtr: Long)

    /**
     * 开始完成任务
     */
    private external fun startCompletion(
        modelPtr: Long,
        prompt: String,
    )

    /**
     * 完成循环
     */
    private external fun completionLoop(modelPtr: Long): String

    /**
     * 停止完成任务
     */
    private external fun stopCompletion(modelPtr: Long)
```

本地实现：

```cpp
#include "LLMInference.h"
#include <jni.h>

/**
 * @brief 加载 LLM 模型。
 *
 * 该函数通过 JNI 从 Java 层调用，用于加载指定路径的 LLM 模型，并返回指向模型实例的指针。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPath 包含模型文件路径的 Java 字符串对象。
 * @param minP 令牌被考虑的最小概率，即核采样（top-P sampling）的阈值。
 * @param temperature 采样温度，控制输出的随机性。
 * @param storeChats 是否存储聊天记录。
 * @param contextSize 模型的上下文大小。
 * @param chatTemplate 包含聊天模板的 Java 字符串对象。
 * @param nThreads 用于推理的线程数。
 * @param useMmap 是否使用内存映射加载模型。
 * @param useMlock 是否锁定模型内存。
 * @return 指向 LLMInference 实例的 jlong 类型指针，若加载失败可能抛出 Java 异常。
 */
extern "C" JNIEXPORT jlong JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_loadModel(JNIEnv* env, jobject thiz, jstring modelPath, jfloat minP,
                                                         jfloat temperature, jboolean storeChats, jlong contextSize,
                                                         jstring chatTemplate, jint nThreads, jboolean useMmap, jboolean useMlock) {
    // 标识是否复制字符串内容的标志
    jboolean    isCopy           = true;
    // 将 Java 字符串转换为 C 风格的 UTF-8 字符串，获取模型路径
    const char* modelPathCstr    = env->GetStringUTFChars(modelPath, &isCopy);
    // 创建一个新的 LLMInference 实例
    auto*       llmInference     = new LLMInference();
    // 将 Java 字符串转换为 C 风格的 UTF-8 字符串，获取聊天模板
    const char* chatTemplateCstr = env->GetStringUTFChars(chatTemplate, &isCopy);

    try {
        // 调用 LLMInference 实例的 loadModel 方法加载模型
        llmInference->loadModel(modelPathCstr, minP, temperature, storeChats, contextSize, chatTemplateCstr, nThreads,
                                useMmap, useMlock);
    } catch (std::runtime_error& error) {
        // 若加载过程中抛出异常，在 Java 层抛出 IllegalStateException 异常
        env->ThrowNew(env->FindClass("java/lang/IllegalStateException"), error.what());
    }

    // 释放之前获取的 C 风格的模型路径字符串
    env->ReleaseStringUTFChars(modelPath, modelPathCstr);
    // 释放之前获取的 C 风格的聊天模板字符串
    env->ReleaseStringUTFChars(chatTemplate, chatTemplateCstr);
    // 将 LLMInference 实例指针转换为 jlong 类型并返回
    return reinterpret_cast<jlong>(llmInference);
}

/**
 * @brief 向聊天记录中添加消息。
 *
 * 该函数通过 JNI 从 Java 层调用，用于向已加载的 LLM 模型的聊天记录中添加消息。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 * @param message 包含要添加消息内容的 Java 字符串对象。
 * @param role 包含消息角色（如 "user", "system" 等）的 Java 字符串对象。
 */
extern "C" JNIEXPORT void JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_addChatMessage(JNIEnv* env, jobject thiz, jlong modelPtr, jstring message,
                                                              jstring role) {
    // 标识是否复制字符串内容的标志
    jboolean    isCopy       = true;
    // 将 Java 字符串转换为 C 风格的 UTF-8 字符串，获取消息内容
    const char* messageCstr  = env->GetStringUTFChars(message, &isCopy);
    // 将 Java 字符串转换为 C 风格的 UTF-8 字符串，获取消息角色
    const char* roleCstr     = env->GetStringUTFChars(role, &isCopy);
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto*       llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    // 调用 LLMInference 实例的 addChatMessage 方法添加消息
    llmInference->addChatMessage(messageCstr, roleCstr);
    // 释放之前获取的 C 风格的消息内容字符串
    env->ReleaseStringUTFChars(message, messageCstr);
    // 释放之前获取的 C 风格的消息角色字符串
    env->ReleaseStringUTFChars(role, roleCstr);
}

/**
 * @brief 获取响应生成速度。
 *
 * 该函数通过 JNI 从 Java 层调用，用于获取已加载的 LLM 模型生成响应的速度。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 * @return 响应生成速度，单位可能因实现而异。
 */
extern "C" JNIEXPORT jfloat JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_getResponseGenerationSpeed(JNIEnv* env, jobject thiz, jlong modelPtr) {
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto* llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    // ... 已有代码 ...
    return llmInference->getResponseGenerationTime();
}

/**
 * @brief 获取已使用的上下文大小。
 *
 * 该函数通过 JNI 从 Java 层调用，用于获取已加载的 LLM 模型当前已使用的上下文大小。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 * @return 已使用的上下文大小，单位可能因实现而异。
 */
extern "C" JNIEXPORT jint JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_getContextSizeUsed(JNIEnv* env, jobject thiz, jlong modelPtr) {
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto* llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    // 调用 LLMInference 实例的 getContextSizeUsed 方法获取已使用的上下文大小
    return llmInference->getContextSizeUsed();
}

/**
 * @brief 关闭 LLM 模型并释放资源。
 *
 * 该函数通过 JNI 从 Java 层调用，用于关闭已加载的 LLM 模型并释放相关资源。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 */
extern "C" JNIEXPORT void JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_close(JNIEnv* env, jobject thiz, jlong modelPtr) {
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto* llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    // 删除 LLMInference 实例，释放资源
    delete llmInference;
}

/**
 * @brief 启动响应生成过程。
 *
 * 该函数通过 JNI 从 Java 层调用，用于启动已加载的 LLM 模型根据给定提示生成响应的过程。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 * @param prompt 包含生成响应所需提示的 Java 字符串对象。
 */
extern "C" JNIEXPORT void JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_startCompletion(JNIEnv* env, jobject thiz, jlong modelPtr, jstring prompt) {
    // 标识是否复制字符串内容的标志
    jboolean    isCopy       = true;
    // 将 Java 字符串转换为 C 风格的 UTF-8 字符串，获取提示内容
    const char* promptCstr   = env->GetStringUTFChars(prompt, &isCopy);
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto*       llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    // 调用 LLMInference 实例的 startCompletion 方法启动响应生成过程
    llmInference->startCompletion(promptCstr);
    // 释放之前获取的 C 风格的提示内容字符串
    env->ReleaseStringUTFChars(prompt, promptCstr);
}

/**
 * @brief 循环生成响应片段。
 *
 * 该函数通过 JNI 从 Java 层调用，用于在已启动响应生成过程后，循环获取 LLM 模型生成的响应片段。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 * @return 包含生成响应片段的 Java 字符串对象，若出现异常则返回 nullptr。
 */
extern "C" JNIEXPORT jstring JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_completionLoop(JNIEnv* env, jobject thiz, jlong modelPtr) {
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto* llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    try {
        // 调用 LLMInference 实例的 completionLoop 方法获取响应片段
        std::string response = llmInference->completionLoop();
        // 将 C++ 字符串转换为 Java 字符串并返回
        return env->NewStringUTF(response.c_str());
    } catch (std::runtime_error& error) {
        // 若生成过程中抛出异常，在 Java 层抛出 IllegalStateException 异常
        env->ThrowNew(env->FindClass("java/lang/IllegalStateException"), error.what());
        return nullptr;
    }
}

/**
 * @brief 停止响应生成过程。
 *
 * 该函数通过 JNI 从 Java 层调用，用于停止已启动的 LLM 模型响应生成过程。
 *
 * @param env JNI 环境指针，用于与 Java 虚拟机交互。
 * @param thiz 调用该本地方法的 Java 对象引用。
 * @param modelPtr 指向 LLMInference 实例的 jlong 类型指针。
 */
extern "C" JNIEXPORT void JNICALL
Java_com_stephen_llamacppbridge_LlamaCppBridge_stopCompletion(JNIEnv* env, jobject thiz, jlong modelPtr) {
    // 将 jlong 类型的指针转换为 LLMInference 实例指针
    auto* llmInference = reinterpret_cast<LLMInference*>(modelPtr);
    // 调用 LLMInference 实例的 stopCompletion 方法停止响应生成过程
    llmInference->stopCompletion();
}
```

将这个模组直接封装成一个aar，也可以直接被其他模组依赖编译。

外部使用时，先将gguf文件从手机下载路径复制到内部目录，也可以直接在线从 `Hugging Face` 上下载到本地内部目录。然后调用 `loadModel()` 、 `getResponseAsFlow()` 等接口来加载模型，获取生成的对话回复。

运行结果如下，模型加载和对话回复：

![](/assets/img/blog/blogs_local_llm_load_models.png){:width="600" height="360" loading="lazy"}
