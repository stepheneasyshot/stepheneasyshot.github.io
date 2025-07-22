---
layout: post
description: > 
  本文介绍了借助llama.cpp开源框架运行Android平台上的AI小模型的流程
image: 
  path: /assets/img/blog/blogs_android_ai_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_ai_cover.png
    960w:  /assets/img/blog/blogs_android_ai_cover.png
    480w:  /assets/img/blog/blogs_android_ai_cover.png
accent_image: /assets/img/blog/blogs_android_ai_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】Android平台运行AI小模型
接上文：

[【AI】大模型开发流程和运行环境简介](./2025-7-19-【AI】大模型开发流程和运行环境简介.md)

## Android 端集成方法
## llama.cpp
作为一名 Android 开发者，llama.cpp 打开了在移动应用中实现**端侧 AI** 的全新大门。

可以通过 Android NDK (Native Development Kit) 将 llama.cpp 的 C/C++ 代码编译为 `.so` 库，并在您的 Java/Kotlin 代码中通过 JNI (Java Native Interface) 调用它。社区中也有一些封装好的 Android 库或示例项目，可以帮助您更快速地开始。llama.cpp 让您可以在 Android 应用中直接拥有一个“本地的智能大脑”，从而创造出前所未有的智能应用体验。

接下来介绍下如何在项目中集成llama.cpp，从而加载gguf模型，运行本地的小模型。直接由开源项目 [SmolChat-Android Public](https://github.com/shubham0204/SmolChat-Android) 分析。

## LiteRT
来自Google官方的开源项目：

[google-ai-edge gallery](https://github.com/google-ai-edge/gallery)

首先介绍下几个基础框架：
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

主要特性
* 针对设备端机器学习进行了优化：LiteRT 解决了五项关键的 ODML 约束条件：延迟时间（无需往返服务器）、隐私性（没有个人数据离开设备）、连接性（无需连接到互联网）、大小（缩减了模型和二进制文件大小）和功耗（高效推理和缺少网络连接）。
* 支持多平台：与 Android 和 iOS 设备、嵌入式 Linux 和微控制器兼容。
* 多框架模型选项：AI Edge 提供了一些工具，可将 TensorFlow、PyTorch 和 JAX 模型转换为 FlatBuffers 格式 (.tflite)，让您能够在 LiteRT 上使用各种先进的模型。您还可以使用可处理量化和元数据的模型优化工具。
* 支持多种语言：包括适用于 Java/Kotlin、Swift、Objective-C、C++ 和 Python 的 SDK。
* 高性能：通过 GPU 和 iOS Core ML 等专用代理实现硬件加速。

### 运行流程
本节介绍了在设备上运行 LiteRT（简称 Lite Runtime）模型以根据输入数据进行预测的过程。这是 使用 LiteRT 解释器来实现，该解释器使用静态图排序和 自定义（非动态）内存分配器，以确保将负载、初始化 和执行延迟时间

LiteRT 推理通常遵循以下步骤：
* 加载模型：将 .tflite 模型加载到内存中，其中包含模型的执行图。
* 转换数据：将输入数据转换为预期格式并 维度。模型的原始输入数据通常与模型预期的输入数据格式不匹配。例如，您可能需要调整图片大小或更改图片格式，以使其与模型兼容。
* 运行推理：执行 LiteRT 模型以进行预测。这个 这个步骤涉及使用 LiteRT API 执行模型。它涉及 例如构建解释器和分配张量等步骤。
* 解释输出：以有意义的方式解释输出张量 对您的应用非常有用例如，一个模型可能只返回 概率列表。您可以将概率映射到相关类别并设置输出格式。

TensorFlow 推理 API 适用于最常见的移动设备和嵌入式设备 Android、iOS 和 Linux 等平台，支持多种编程语言。

在大多数情况下，API 设计反映的是性能优先于易用性。LiteRT 专为在小型设备上快速推理而设计，因此 API 会避免不必要的复制，但会牺牲便捷性。在所有库中，LiteRT API 可让您加载模型、提供输入并检索推理输出。

在 **Android** 上，可以使用 Java 或 C++ API 执行 LiteRT 推理。通过 Java API 提供了便利，可以直接在 Android 应用中使用 activity 类。C++ API 提供了更高的灵活性和速度，但可能需要 编写 JNI 封装容器以在 Java 层和 C++ 层之间移动数据。

在 **iOS** 上，LiteRT 可在 Swift 和 Objective-C iOS 库中使用。您也可以使用 C API 编写代码。

在 **Linux** 平台上，您可以使用 C++ 中提供的 LiteRT API 运行推理。

加载和运行 LiteRT 模型涉及以下步骤：
* 将模型加载到内存中。
* 根据现有模型构建 Interpreter。
* 设置输入张量值。
* 调用推理。
* 输出张量值。

### llama.cpp 的两种运行方案
#### 一、使用Termux命令行编译运行
笔者没有实操，主要实践的后一种方案，第一种方案直接参考的掘金文章：

[安卓手机部署阿里的Qwen3-0.6B（llama.cpp，ollama）](https://juejin.cn/post/7506727402821042191)

这种方法就是将 `Android` 设备当作 `Linux` 设备来使用，手机需要安装Termux.

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

编译llama.cpp

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

直接下载gguf文件，避免转换步骤：

[Qwen3-0.6B-GGUF](https://huggingface.co/Qwen/Qwen3-0.6B-GGUF)

直接在命令行中启动：

```
llama-cli -m Qwen3-0.6B-Q8_0.gguf
```

> llama-cli，即 ​​CLI 模式​​（Command-Line Interface 模式）是指通过命令行直接运行模型进行推理（文本生成）的方式，而不是通过 API 或图形界面。这是 llama.cpp 最基础的使用方式，适合本地测试、脚本调用或服务器部署。

也可以以server方式启动：

```
llama-server -m Qwen3-0.6B-Q8_0.gguf --port 8080 --host 0.0.0.0
```

在同一个局域网中，电脑端可以直接通过 `openai` 的开发套件，和手机端运行的服务进行通信：

```python
import requests
import json
import time

# API_URL = "http://192.168.31.86:8080/v1/chat/completions"
API_URL = "http://192.168.31.44:8080/v1/chat/completions"
# API_URL = "http://127.0.0.1:8080/v1/chat/completions"


payload = {
    "model": "Qwen3-0.6B-Q8_0",  # llama-server 中可随意写
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

#### 二、使用JNI开发接口和llama.cpp交互
这种方案就是比较符合 `Android` 设备上运行的直观预期，通过一个APP页面来承载功能，在应用中，以用户友好的 `UX交互` 来和本地模型进行通信。

![](/assets/img/blog/blogs_ai_llamacpp_smollchat.png){:width="300" height="620" loading="lazy"}

底层使用 `llama.cpp` 加载和执行 GGUF 模型。

由于 llama.cpp 是用纯 C/C++ 编写的，因此很容易利用 AndroidStudio 的NDK工具，和应用app一起编译运行。

首先，定义JNI函数，第一步需要通过 `kotlin` 代码来加载 `gguf` 文件。

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

extern "C" JNIEXPORT jlong JNICALL
Java_io_shubham0204_smollm_GGUFReader_getGGUFContextNativeHandle(JNIEnv* env, jobject thiz, jstring modelPath) {
    jboolean         isCopy        = true;
    const char*      modelPathCStr = env->GetStringUTFChars(modelPath, &isCopy);
    gguf_init_params initParams    = { .no_alloc = true, .ctx = nullptr };
    gguf_context*    ggufContext   = gguf_init_from_file(modelPathCStr, initParams);
    env->ReleaseStringUTFChars(modelPath, modelPathCStr);
    return reinterpret_cast<jlong>(ggufContext);
}

extern "C" JNIEXPORT jlong JNICALL
Java_io_shubham0204_smollm_GGUFReader_getContextSize(JNIEnv* env, jobject thiz, jlong nativeHandle) {
    gguf_context* ggufContext       = reinterpret_cast<gguf_context*>(nativeHandle);
    int64_t       architectureKeyId = gguf_find_key(ggufContext, "general.architecture");
    if (architectureKeyId == -1)
        return -1;
    std::string architecture       = gguf_get_val_str(ggufContext, architectureKeyId);
    std::string contextLengthKey   = architecture + ".context_length";
    int64_t     contextLengthKeyId = gguf_find_key(ggufContext, contextLengthKey.c_str());
    if (contextLengthKeyId == -1)
        return -1;
    uint32_t contextLength = gguf_get_val_u32(ggufContext, contextLengthKeyId);
    return contextLength;
}

extern "C" JNIEXPORT jstring JNICALL
Java_io_shubham0204_smollm_GGUFReader_getChatTemplate(JNIEnv* env, jobject thiz, jlong nativeHandle) {
    gguf_context* ggufContext       = reinterpret_cast<gguf_context*>(nativeHandle);
    int64_t       chatTemplateKeyId = gguf_find_key(ggufContext, "tokenizer.chat_template");
    std::string   chatTemplate;
    if (chatTemplateKeyId == -1) {
        chatTemplate = "";
    } else {
        chatTemplate = gguf_get_val_str(ggufContext, chatTemplateKeyId);
    }
    return env->NewStringUTF(chatTemplate.c_str());
}
```

然后根据 `llama.cpp` 的几个核心的方法，如加载，对话等功能，来编写对接的接口类 