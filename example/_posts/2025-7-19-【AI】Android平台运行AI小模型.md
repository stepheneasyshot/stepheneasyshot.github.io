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
# Android平台运行AI小模型
接上文：

[【AI】大模型开发流程和运行环境简介](./2025-7-19-【AI】大模型开发流程和运行环境简介.md)

## llama.cpp
**llama.cpp** 是一个由 Georgi Gerganov 开发的开源项目，其核心是用 **C/C++** 编写的。它的主要目标是让大型语言模型 (LLMs) 能够在各种硬件上高效地运行**推理**，尤其是在**本地设备上**，包括那些没有高端独立 GPU 的普通电脑或移动设备。该项目的愿景是 **“民主化”LLMs 的部署** ，让每个人都能在自己的设备上体验和使用这些强大的模型，而不仅仅依赖于昂贵的云服务。

`llama.cpp` 的设计理念是追求**极致的效率、最小化外部依赖、广泛的硬件兼容性以及高度的灵活性**。
### 轻量与高效的 C/C++ 实现
`llama.cpp` 完全使用 C 和 C++ 编写，不依赖于大型的深度学习框架（如 TensorFlow 或 PyTorch）的运行时库。这意味着它编译出来的程序**体积非常小，运行时内存占用也低**。这对于资源受限的移动设备来说至关重要。

除了底层的 BLAS (基本线性代数子程序库) 或特定硬件的计算库（如 CUDA、Metal）， `llama.cpp` 几乎没有其他复杂的外部依赖。这使得它非常容易编译和部署。

`llama.cpp` 对 CPU 进行了大量底层优化，利用了各种 CPU 指令集（如 x86-64 上的 AVX/AVX2/AVX-512，以及 ARM 上的 Neon）。这让 LLMs 可以在没有独立 GPU 的设备上，仅仅依靠 CPU 也能获得令人惊讶的推理速度。
### 广泛的硬件加速支持
除了强大的 CPU 优化，llama.cpp 还支持多种 GPU 和专用硬件加速：

* **Apple Silicon (Metal)**：针对苹果 M 系列芯片的 Metal API 进行了优化，充分利用了其强大的统一内存架构和神经网络引擎。
* **NVIDIA GPU (CUDA)**：支持 NVIDIA 显卡，通过 CUDA 加速推理，性能非常出色。
* **AMD GPU (hipBLAS)**：兼容 AMD GPU。
* **Intel GPU (SYCL/oneAPI)**：支持英特尔的集成和独立显卡。
* **通用 GPU 后端 (Vulkan/OpenCL)**：这对于 Android 开发者尤为重要。通过 **OpenCL** 后端，llama.cpp 能够利用 Android 设备中 SoC（System on Chip）内置的 GPU 或 DSP（数字信号处理器，如高通骁龙的 Hexagon DSP）进行加速，将推理负载从 CPU 转移到更擅长并行计算的硬件上。
* **CPU + GPU 混合推理**：对于一些较大的模型，即使单张 GPU 的显存不足以容纳整个模型，llama.cpp 也能将模型的某些层加载到 GPU 上运行，其余部分则在 CPU 上运行，实现资源的有效利用。

### 使用 GGUF 文件格式进行模型存储
**GGUF** 文件是专门为高效存储 LLM 权重和元数据而设计的。

GGUF 格式支持从全精度 (FP32) 到半精度 (FP16) 以及多种低精度（如 INT8、INT5、INT4、INT2）的**模型量化**。量化可以在牺牲极小精度损失的情况下，大幅**减小模型体积，降低内存占用和计算需求**。这对于在手机上运行大型模型至关重要，能让原本无法加载的模型变得可用。

GGUF 文件不仅包含模型权重，还打包了模型的词表、超参数、架构信息和特殊 token ID 等所有运行所需的元数据。一个 GGUF 文件就是一个独立的、可运行的模型包。

此外，GGUF 支持内存映射。这意味着操作系统可以直接将文件内容映射到内存中，而无需将整个模型完全复制到 RAM。这大大加快了模型加载速度，并允许在物理内存不足的情况下也能运行大模型（通过操作系统的虚拟内存管理）。
## GGUF文件
**GGUF (GGML Universal Format)** 是一种专门为 **llama.cpp** 项目设计和优化的**二进制文件格式**。它的主要目的是高效地存储和加载大型语言模型的权重（参数）和元数据。

GGUF 格式的诞生是为了解决早期 GGML 格式的一些限制，并提供了**更好的灵活性、可扩展性和向后兼容性**。可以把它想象成一个 **包含了模型“大脑”里所有知识的“盒子”** ，这个盒子设计得非常紧凑和高效，便于在各种设备上快速打开和使用。
### 命名约定
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

## GGUF 的主要特点和优势

1.  **二进制格式，高效存储和加载**：
    * GGUF 文件是二进制的，这意味着它直接存储原始数据，而不是可读的文本格式（如 JSON 或 YAML）。这使得文件更小，并且解析和加载速度更快，这对于在 Android 设备上快速启动模型至关重要。
2.  **广泛的硬件兼容性**：
    * GGUF 格式本身是硬件无关的，但它与 llama.cpp 项目紧密结合，从而能利用 llama.cpp 在 CPU 和多种 GPU（包括 Android 设备的 NPU/DSP）上的优化。
3.  **支持模型量化 (Quantization)**：
    * 这是 GGUF 的核心优势之一。它支持多种精度级别的量化（例如，从全精度 FP32 到 FP16、INT8、INT5、INT4 甚至 INT2）。量化可以大幅**减小模型的文件大小**，同时**降低推理时的内存占用和计算需求**。这对于资源有限的 Android 设备来说至关重要，因为它能让更大的模型运行起来。
    * 例如，一个 70 亿参数的 FP16 模型可能有 13GB 大小，但经过 INT4 量化后可能只有 4GB 左右。
4.  **包含完整的模型元数据**：
    * GGUF 文件不仅仅包含模型的权重，还包含了模型运行所需的各种**元数据**，如：
        * **词表 (Vocabulary)**：模型理解的单词或 token 列表。
        * **超参数 (Hyperparameters)**：如模型层数、注意力头数、隐藏层大小等。
        * **架构信息**：模型是基于 Llama、Mistral 还是其他架构。
        * **特殊 Token ID**：如 EOS (End-Of-Sequence), BOS (Beginning-Of-Sequence) 等。
        * 这些元数据都存储在一个文件中，简化了模型的管理和加载过程。
5.  **内存映射 (Memory Mapping) 支持**：
    * GGUF 文件支持内存映射。这意味着操作系统可以直接将文件内容映射到内存中，而不需要将整个文件完全复制到 RAM。这大大减少了模型的加载时间，并且即使手机物理内存不足以完全容纳模型，也可以通过操作系统管理虚拟内存来运行。
6.  **向后兼容性与可扩展性**：
    * GGUF 格式在设计时考虑了向后兼容性，这意味着即使 llama.cpp 项目更新，旧的 GGUF 文件通常也能正常工作。同时，它也具有良好的可扩展性，可以方便地添加新的特性和信息。
7.  **社区支持和模型转换**：
    * 现在，许多流行的开源大型语言模型（如 Llama 2/3、Mistral、Gemma、Qwen、Phi-2 等）都有社区贡献者将其转换为 GGUF 格式。这意味着您可以轻松找到这些模型的 GGUF 版本，直接用于 llama.cpp 或您的 Android 应用。

## GGUF 文件的结构
一个 GGUF 文件大致可以分为两个主要部分：

1.  **头部 (Header)**：包含 GGUF 文件的魔数（用于识别文件类型）、版本号以及一些全局元数据（如模型总层数、维度等）。
2.  **元数据 (Metadata) 和张量 (Tensors)**：
    * **KV 对 (Key-Value Pairs)**：存储了模型的各种超参数、架构信息、词表和一些其他配置。这些都是以键值对的形式存储的，方便访问。
    * **张量数据 (Tensor Data)**：这是模型真正的权重数据。每个张量都会有其名称、维度和数据类型（例如，量化后的 INT4、INT8 或 FP16 等）。这些数据会按照特定的对齐方式存储，以确保高效读取。

![](/assets/img/blog/blogs_gguf_file_structure.png)

```cpp
enum ggml_type: uint32_t {
    GGML_TYPE_F32     = 0,
    GGML_TYPE_F16     = 1,
    GGML_TYPE_Q4_0    = 2,
    GGML_TYPE_Q4_1    = 3,
    // GGML_TYPE_Q4_2 = 4, support has been removed
    // GGML_TYPE_Q4_3 = 5, support has been removed
    GGML_TYPE_Q5_0    = 6,
    GGML_TYPE_Q5_1    = 7,
    GGML_TYPE_Q8_0    = 8,
    GGML_TYPE_Q8_1    = 9,
    GGML_TYPE_Q2_K    = 10,
    GGML_TYPE_Q3_K    = 11,
    GGML_TYPE_Q4_K    = 12,
    GGML_TYPE_Q5_K    = 13,
    GGML_TYPE_Q6_K    = 14,
    GGML_TYPE_Q8_K    = 15,
    GGML_TYPE_IQ2_XXS = 16,
    GGML_TYPE_IQ2_XS  = 17,
    GGML_TYPE_IQ3_XXS = 18,
    GGML_TYPE_IQ1_S   = 19,
    GGML_TYPE_IQ4_NL  = 20,
    GGML_TYPE_IQ3_S   = 21,
    GGML_TYPE_IQ2_S   = 22,
    GGML_TYPE_IQ4_XS  = 23,
    GGML_TYPE_I8      = 24,
    GGML_TYPE_I16     = 25,
    GGML_TYPE_I32     = 26,
    GGML_TYPE_I64     = 27,
    GGML_TYPE_F64     = 28,
    GGML_TYPE_IQ1_M   = 29,
    GGML_TYPE_COUNT,
};

enum gguf_metadata_value_type: uint32_t {
    // The value is a 8-bit unsigned integer.
    GGUF_METADATA_VALUE_TYPE_UINT8 = 0,
    // The value is a 8-bit signed integer.
    GGUF_METADATA_VALUE_TYPE_INT8 = 1,
    // The value is a 16-bit unsigned little-endian integer.
    GGUF_METADATA_VALUE_TYPE_UINT16 = 2,
    // The value is a 16-bit signed little-endian integer.
    GGUF_METADATA_VALUE_TYPE_INT16 = 3,
    // The value is a 32-bit unsigned little-endian integer.
    GGUF_METADATA_VALUE_TYPE_UINT32 = 4,
    // The value is a 32-bit signed little-endian integer.
    GGUF_METADATA_VALUE_TYPE_INT32 = 5,
    // The value is a 32-bit IEEE754 floating point number.
    GGUF_METADATA_VALUE_TYPE_FLOAT32 = 6,
    // The value is a boolean.
    // 1-byte value where 0 is false and 1 is true.
    // Anything else is invalid, and should be treated as either the model being invalid or the reader being buggy.
    GGUF_METADATA_VALUE_TYPE_BOOL = 7,
    // The value is a UTF-8 non-null-terminated string, with length prepended.
    GGUF_METADATA_VALUE_TYPE_STRING = 8,
    // The value is an array of other values, with the length and type prepended.
    ///
    // Arrays can be nested, and the length of the array is the number of elements in the array, not the number of bytes.
    GGUF_METADATA_VALUE_TYPE_ARRAY = 9,
    // The value is a 64-bit unsigned little-endian integer.
    GGUF_METADATA_VALUE_TYPE_UINT64 = 10,
    // The value is a 64-bit signed little-endian integer.
    GGUF_METADATA_VALUE_TYPE_INT64 = 11,
    // The value is a 64-bit IEEE754 floating point number.
    GGUF_METADATA_VALUE_TYPE_FLOAT64 = 12,
};

// A string in GGUF.
struct gguf_string_t {
    // The length of the string, in bytes.
    uint64_t len;
    // The string as a UTF-8 non-null-terminated string.
    char string[len];
};

union gguf_metadata_value_t {
    uint8_t uint8;
    int8_t int8;
    uint16_t uint16;
    int16_t int16;
    uint32_t uint32;
    int32_t int32;
    float float32;
    uint64_t uint64;
    int64_t int64;
    double float64;
    bool bool_;
    gguf_string_t string;
    struct {
        // Any value type is valid, including arrays.
        gguf_metadata_value_type type;
        // Number of elements, not bytes
        uint64_t len;
        // The array of values.
        gguf_metadata_value_t array[len];
    } array;
};

struct gguf_metadata_kv_t {
    // The key of the metadata. It is a standard GGUF string, with the following caveats:
    // - It must be a valid ASCII string.
    // - It must be a hierarchical key, where each segment is `lower_snake_case` and separated by a `.`.
    // - It must be at most 2^16-1/65535 bytes long.
    // Any keys that do not follow these rules are invalid.
    gguf_string_t key;

    // The type of the value.
    // Must be one of the `gguf_metadata_value_type` values.
    gguf_metadata_value_type value_type;
    // The value.
    gguf_metadata_value_t value;
};

struct gguf_header_t {
    // Magic number to announce that this is a GGUF file.
    // Must be `GGUF` at the byte level: `0x47` `0x47` `0x55` `0x46`.
    // Your executor might do little-endian byte order, so it might be
    // check for 0x46554747 and letting the endianness cancel out.
    // Consider being *very* explicit about the byte order here.
    uint32_t magic;
    // The version of the format implemented.
    // Must be `3` for version described in this spec, which introduces big-endian support.
    //
    // This version should only be increased for structural changes to the format.
    // Changes that do not affect the structure of the file should instead update the metadata
    // to signify the change.
    uint32_t version;
    // The number of tensors in the file.
    // This is explicit, instead of being included in the metadata, to ensure it is always present
    // for loading the tensors.
    uint64_t tensor_count;
    // The number of metadata key-value pairs.
    uint64_t metadata_kv_count;
    // The metadata key-value pairs.
    gguf_metadata_kv_t metadata_kv[metadata_kv_count];
};

uint64_t align_offset(uint64_t offset) {
    return offset + (ALIGNMENT - (offset % ALIGNMENT)) % ALIGNMENT;
}

struct gguf_tensor_info_t {
    // The name of the tensor. It is a standard GGUF string, with the caveat that
    // it must be at most 64 bytes long.
    gguf_string_t name;
    // The number of dimensions in the tensor.
    // Currently at most 4, but this may change in the future.
    uint32_t n_dimensions;
    // The dimensions of the tensor.
    uint64_t dimensions[n_dimensions];
    // The type of the tensor.
    ggml_type type;
    // The offset of the tensor's data in this file in bytes.
    //
    // This offset is relative to `tensor_data`, not to the start
    // of the file, to make it easier for writers to write the file.
    // Readers should consider exposing this offset relative to the
    // file to make it easier to read the data.
    //
    // Must be a multiple of `ALIGNMENT`. That is, `align_offset(offset) == offset`.
    uint64_t offset;
};

struct gguf_file_t {
    // The header of the file.
    gguf_header_t header;

    // Tensor infos, which can be used to locate the tensor data.
    gguf_tensor_info_t tensor_infos[header.tensor_count];

    // Padding to the nearest multiple of `ALIGNMENT`.
    //
    // That is, if `sizeof(header) + sizeof(tensor_infos)` is not a multiple of `ALIGNMENT`,
    // this padding is added to make it so.
    //
    // This can be calculated as `align_offset(position) - position`, where `position` is
    // the position of the end of `tensor_infos` (i.e. `sizeof(header) + sizeof(tensor_infos)`).
    uint8_t _padding[];

    // Tensor data.
    //
    // This is arbitrary binary data corresponding to the weights of the model. This data should be close
    // or identical to the data in the original model file, but may be different due to quantization or
    // other optimizations for inference. Any such deviations should be recorded in the metadata or as
    // part of the architecture definition.
    //
    // Each tensor's data must be stored within this array, and located through its `tensor_infos` entry.
    // The offset of each tensor's data must be a multiple of `ALIGNMENT`, and the space between tensors
    // should be padded to `ALIGNMENT` bytes.
    uint8_t tensor_data[];
};
```

### GGUF是本地运行 LLM 的关键
GGUF 结合 llama.cpp 是在 Android 设备上实现本地 LLM 推理的核心技术栈。它使得之前只能在云端运行的模型现在可以在用户的手机上运行，带来了巨大的潜力和机会。

* **优化资源利用**：通过量化和内存映射，GGUF 文件可以显著减少模型在 Android 设备上的内存和存储占用，这对于有限的设备资源至关重要。
* **隐私保护**：模型在设备本地运行，用户的数据无需上传到云端，大大增强了数据隐私和安全性。
* **离线能力**：一旦模型下载到设备，即使没有网络连接，您的应用也能提供 LLM 功能。
* **降低成本和延迟**：无需依赖云端 API，可以节省云服务费用，并降低推理延迟，提升用户体验。

作为 Android 开发者，当你想要在你的应用中加入本地 AI 驱动的功能，比如智能聊天、文本生成、代码辅助等时，寻找或转换成 GGUF 格式的模型，并结合 llama.cpp 在 Android 上的集成，将是一个非常强大的方案。
## Android 端集成方法
作为一名 Android 开发者，llama.cpp 打开了在移动应用中实现**端侧 AI** 的全新大门。

可以通过 Android NDK (Native Development Kit) 将 llama.cpp 的 C/C++ 代码编译为 `.so` 库，并在您的 Java/Kotlin 代码中通过 JNI (Java Native Interface) 调用它。社区中也有一些封装好的 Android 库或示例项目，可以帮助您更快速地开始。llama.cpp 让您可以在 Android 应用中直接拥有一个“本地的智能大脑”，从而创造出前所未有的智能应用体验。

接下来介绍下如何在项目中集成llama.cpp，从而加载gguf模型，运行本地的小模型。直接由开源项目 [SmolChat-Android Public](https://github.com/shubham0204/SmolChat-Android) 分析。

### 