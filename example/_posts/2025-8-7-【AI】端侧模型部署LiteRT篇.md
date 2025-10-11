---
layout: post
description: > 
  本文介绍了借助Google的LiteRT框架在Android平台上运行端侧AI模型的流程
image: 
  path: /assets/img/blog/blogs_google_edge_ai_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_google_edge_ai_cover.png
    960w:  /assets/img/blog/blogs_google_edge_ai_cover.png
    480w:  /assets/img/blog/blogs_google_edge_ai_cover.png
accent_image: /assets/img/blog/blogs_google_edge_ai_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】端侧模型部署LiteRT篇
## LiteRT简介
LiteRT（Lite Runtime 的缩写，前身为 **TensorFlow Lite**）是 Google 推出的一款**轻量级**、**高性能**的深度学习推理框架。它专门设计用于在**资源受限**的设备上（如移动设备、嵌入式系统和边缘计算设备）高效地部署和运行机器学习模型。

![](/assets/img/blog/blogs_ai_litert_cover.png){:width="400" height="200"}

LiteRT 的核心价值在于提供一个**快速**、**小巧**且**高效**的运行时环境，使训练好的机器学习模型能够在设备上本地执行推理。主要特性：

* 针对设备端机器学习进行了优化：LiteRT 解决了五项关键的 ODML 约束条件：延迟时间（无需往返服务器）、隐私性（没有个人数据离开设备）、连接性（无需连接到互联网）、大小（缩减了模型和二进制文件大小）和功耗（高效推理和缺少网络连接）。
* 支持多平台：与 Android 和 iOS 设备、嵌入式 Linux 和微控制器兼容。
* 多框架模型选项：AI Edge 提供了一些工具，可将 TensorFlow、PyTorch 和 JAX 模型转换为 FlatBuffers 格式 (.tflite)，让您能够在 LiteRT 上使用各种先进的模型。您还可以使用可处理量化和元数据的模型优化工具。
* 支持多种语言：包括适用于 Java/Kotlin、Swift、Objective-C、C++ 和 Python 的 SDK。
* 高性能：通过 GPU 和 iOS Core ML 等专用代理实现硬件加速。

LiteRT 将模型打包成一种名为 **FlatBuffers** 的高效可移植格式，文件扩展名为 **.tflite**。

### LiteRT 的部署运行流程
#### 1. 模型加载
第一步，将 **.tflite** 文件（包含模型的执行图、权重和偏差）加载到内存中。

LiteRT 模型采用 **FlatBuffers** 格式，这种格式允许直接映射到内存，避免了传统序列化/反序列化所需的额外解析和内存复制，从而加快加载速度。
#### 2. 构建解释器 (Interpreter)
**LiteRT Interpreter** 解释器是执行模型的核心组件。它使用**静态图排序**和**自定义（非动态）内存分配器**。

在内存分配方面， `Interpreter` 在运行时根据模型图预先分配好所需的张量内存，避免了推理过程中的动态内存分配开销，确保推理延迟稳定且较低。
#### 3. 设置硬件加速器（Delegate/委派）
这一步是 LiteRT 实现高性能的关键。LiteRT 引入了 **Delegate（委派）** 机制，这是一个 API 接口，用于将模型的部分或全部操作卸载到设备上的特定 **硬件加速器** 上执行，而不是仅仅依赖 CPU。

常见的加速器：
* **GPU**：通过 **MLDrift**（新的 GPU 加速实现）或旧的 GPU Delegate 进行加速。
* **NPU/DSP**：通过 **NNAPI**（Android 上的神经网络 API）或高通、联发科等供应商特定的 SDK 来利用神经处理单元。
* **Edge TPU**：Google 专用的边缘张量处理单元。
* **XNNPack**：一个高度优化的 CPU 浮点运算库。

Delegate 会检查模型中的哪些操作可以在加速器上运行，并将它们打包交给加速器执行。
#### 4. 数据预处理与运行推理 (Invoke)
数据预处理截断会将输入数据（如图像、音频）转换为模型期望的格式和维度。通过调用 **`Interpreter::Invoke()`**（或 LiteRT Next 中的 **`CompiledModel::Run()`**）方法，LiteRT 解释器执行模型图中的操作。如果设置了 Delegate，相应的操作将在硬件加速器上执行。
#### 5. 解释输出
获取输出张量的值，并将其转换为对应用有意义的结果（例如，将概率列表映射到具体的类别标签，或绘制目标检测的边界框）。
### 支持的 AI 模型
LiteRT 主要用于**推理**（Inference），它可以运行各种类型的经过优化的机器学习模型，特别是那些针对**视觉**、**音频**和**自然语言处理 (NLP)** 任务的模型：

| 模型类型 | 常见应用 | 优化特点 |
| :--- | :--- | :--- |
| **计算机视觉 (CV)** | 图像分类、目标检测、图像分割、姿态估计、面部识别。 | **CNN** (如 MobileNet, EfficientNet) 经过量化和剪枝，利用 GPU 和 NPU 加速。 |
| **自然语言处理 (NLP)** | 文本分类、命名实体识别、问答系统、**小规模 LLM** (轻量级大语言模型) 推理。 | **Transformer** 模型（如 BERT 变体）经过优化，注重模型大小和低延迟。 |
| **音频处理** | 语音识别、关键词唤醒、声纹识别、环境声分类。 | 各种序列模型和特定设计的声学模型。 |
| **通用机器学习** | 分类、回归、时间序列预测。 | 各种通用的 ML 模型，通常通过 **TensorFlow**、**PyTorch** 等框架训练后转换为 `.tflite` 格式。 |

下面章节介绍下几个下游的基础框架
### TensorFlow & TensorFlowLite
**TensorFlow** 是 Google 开发并开源的一个端到端的机器学习平台。它是一个功能强大且高度灵活的软件库，主要用于构建和训练各种机器学习模型，特别是深度学习模型。

你可以把 TensorFlow 看作一个大型的工具箱，它提供了从数据准备、模型构建、训练、评估到部署的整个机器学习工作流所需的一切。

TensorFlow 具有强大的模型构建能力，支持各种神经网络架构（如卷积神经网络 CNN、循环神经网络 RNN、Transformer 等），可以构建从简单到非常复杂的模型。提供了不同抽象级别的API，例如高级的 **Keras API**，让初学者也能快速上手构建模型；同时也有低级的 API，允许专家进行更精细的控制和自定义。

支持在多核 CPU、GPU 甚至多台机器上进行分布式训练，这对于处理大规模数据集和复杂模型至关重要。提供了 TensorBoard 用于可视化训练过程、TensorFlow Extended (TFX) 用于构建生产级机器学习流水线、TensorFlow Hub 用于共享和重用预训练模型等。

最初主要用于服务器端，但现在也支持桌面（Linux、macOS、Windows）、移动设备和 Web。

TensorFlow 的核心概念之一是数据流图。它将计算表示为图中的节点，数据在图中作为张量（多维数组）流动。

**TensorFlow Lite** 是 TensorFlow 生态系统中的一个重要组成部分，它是专门为**在资源受限的设备（如手机、物联网设备、嵌入式系统和微控制器）上高效运行机器学习模型而设计的**。你可以理解为，它是 TensorFlow 模型的“压缩和优化版本”的运行时环境，让 AI 能够 **“走出云端，进入设备”** 。

和 llama.cpp 一样，TensorFlow Lite也是通过 **模型优化和压缩** 来实现有限资源时高效运行的。

将模型的参数（如权重和激活值）从浮点数转换为更小的整数类型（如 8 位整数），大大减少模型大小和内存占用，同时提高推理速度。移除模型中不重要的连接和参数。将多个操作合并为一个，减少计算开销。TFLite 解释器本身非常小巧，可以在内存和存储空间有限的设备上运行。

**TensorFlowLite在24年9月已经更名为LiteRT。** 之所以更名，是因为 TensorFlow Lite 在发展过程中**已经超越了最初仅支持 TensorFlow 模型的范畴**。它现在能够高效支持从 PyTorch、JAX 和 Keras 等其他主流机器学习框架导出的模型。为了更好地体现这种“多框架”的愿景，并强调其作为设备端高性能运行时的通用性，Google 决定将其更名为 LiteRT。

LiteRT 主要支持多平台：与 Android 和 iOS 设备、嵌入式 Linux 和微控制器兼容。
## 使用 LiteRT 来运行本地模型
应用层的集成，在 **Google** 自己的开源项目中有已经体现：

[google-ai-edge gallery](https://github.com/google-ai-edge/gallery)

该应用也同步上架了Play Store，项目截图：

![](/assets/img/blog/blogs_ai_google_dege_gallery.png)

进入首页可以看到主要有四种使用主题，分别是图片分析，音频描述，提示词试验，以及AI模型对话。

![](/assets/img/blog/blogs_ai_gallery_main_page.png){:width="250" height="600" loading="lazy"}

选择其中一个主题进入之后，通过 `Chrome` 浏览器授权 `Hugging face` 账号，就可以在 `Gallery` 中直接下载模型到其英应用到内部存储中。

![](/assets/img/blog/blogs_ai_gallery_download_gemma.png){:width="250" height="600" loading="lazy"}

下图是音频识别到效果，做语言翻译，物种识别效果还不错，歌曲识别准确率不高。

![](/assets/img/blog/blogs_ai_gallery_gemma_audio_scribe.png){:width="250" height="600" loading="lazy"}

### 架构解析
在 Android 应用中运行的 `LiteRT` 模型会获取数据、处理数据，并根据模型的逻辑生成预测结果。

`LiteRT` 模型需要特殊的运行时环境才能执行，并且传入模型的数据必须采用特定的数据格式（称为张量）。当模型处理数据（称为运行推理）时，它会将预测结果生成为新的张量，并将其传递给 Android 应用，以便应用执行操作，例如向用户显示结果或执行其他业务逻辑。

在功能设计层面，您的 Android 应用需要以下元素才能运行 `LiteRT` 模型：
* 用于执行模型的 `LiteRT` 运行时环境
* 模型输入处理程序，用于将数据转换为张量
* 模型输出处理脚本，用于接收输出结果张量并将其解读为预测结果

![](/assets/img/blog/blogs_ai_litert_framework.png)

### 集成流程

### MediaPipe Tasks 
底层依然基于 `LiteRT` 的运行时来运行端侧AI模型，只是在应用层进行了封装，提供了更方便的接口。

![](/assets/img/blog/blogs_ai_mediapipes_framework.png)

### AI Core 应用进行IPC通信
这个方法，便利性上较前两种方式更进了一步。Google 直接将Gemini Nano模型的下载，加载，推理都封装在 `AI Core` 中，应用层只需要调用AIDL接口和 `AI Core` 进行通信即可。

> Google介绍：Gemini Nano 是我们专为设备端任务打造的最高效模型，它直接在移动芯片上运行，从而支持一系列重要用例。设备端运行支持数据无需离开设备的功能，例如在端到端加密消息应用中提供消息回复建议。它还能通过确定性延迟实现一致的体验，即使在没有网络的情况下也能始终使用各项功能。

![](/assets/img/blog/blogs_ai_google_aicore_framework.png)

**图中的Lora是什么呢？**

LoRA (Low-Rank Adaptation) 是一种用于微调（fine-tuning）大型预训练模型的技术，比如大型语言模型 (LLMs) 或图像生成模型 (如 Stable Diffusion)。

简单来说，LoRA 的核心思想是：在不修改原始大模型参数的情况下，通过向模型中注入少量可训练的层（或称为适配器）来适应新的任务或数据。

可以类比 **Kotlin的扩展函数** 来理解。

需要注意，目前测试版的 **AI Core** 只有 **Pixel 9** 及以上的设备支持。

![](/assets/img/blog/blogs_google_ai_core_play_store.png){:width="250" height="600" loading="lazy"}

使用AICore来和Genimi Nano模型进行通信的步骤非常简单。首先加入gradle依赖：

```groovy
implementation("com.google.ai.edge.aicore:aicore:0.0.1-exp01")
```

注意最低SDK需要31及以上。

在此仅做最小功能验证，直接在Composable组合项中观察一个顶层变量 `aiCoreOutput` 这个 `StateFlow` 的状态变化，在顶层方法中出发通信逻辑，结果会更新到 `aiCoreOutput` 中。

```kotlin
val aiCoreOutput = MutableStateFlow("")

@SuppressLint("StaticFieldLeak")
val generationConfig = generationConfig {
    context = appContext
    temperature = 0.2f
    topK = 16
    maxOutputTokens = 256
}

val downloadCallback = object : DownloadCallback {
    override fun onDownloadProgress(totalBytesDownloaded: Long) {
        super.onDownloadProgress(totalBytesDownloaded)
        println("Download progress: $totalBytesDownloaded")
    }

    override fun onDownloadCompleted() {
        super.onDownloadCompleted()
        println("Download completed")
    }
}

val downloadConfig = DownloadConfig(downloadCallback)
val generativeModel = GenerativeModel(
    generationConfig = generationConfig,
    downloadConfig = downloadConfig
)

suspend fun startChat(input: String) {
    runCatching {
        val response = generativeModel.generateContent(input)
        print(response.text)
        aiCoreOutput.value = response.text.toString()
    }.onFailure { e ->
        e.printStackTrace()
    }
}

fun closeChatResponse() {
    println("Closing chat response")
    generativeModel.close()
}
```

UI界面的实现：

```kotlin
@Composable
fun AiCoreChatDemo(paddingValues: PaddingValues) {

    val input = remember {
        mutableStateOf(
            "What is Quantum Physics?"
        )
    }

    val outputState = aiCoreOutput.collectAsState()

    LaunchedEffect(Unit) {
        startChat(input.value)
    }

    LazyColumn(modifier = Modifier.padding(paddingValues)) {
        item {
            Text(text = "Input: ${input.value}")
            Spacer(modifier = Modifier.height(16.dp))
            Text(text = "Output: ${outputState.value}")
        }
    }

    DisposableEffect(Unit) {
        onDispose {
            aiCoreOutput.value = ""
            closeChatResponse()
        }
    }
}
```
