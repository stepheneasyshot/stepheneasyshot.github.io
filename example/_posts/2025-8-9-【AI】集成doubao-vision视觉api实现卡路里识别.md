---
layout: post
description: > 
  本文介绍了Android和IOS平台集成在线视觉AI的过程
image: 
  path: /assets/img/blog/blogs_ai_doubao_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_ai_doubao_cover.png
    960w:  /assets/img/blog/blogs_ai_doubao_cover.png
    480w:  /assets/img/blog/blogs_ai_doubao_cover.png
accent_image: /assets/img/blog/blogs_ai_doubao_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【AI】集成doubao-vision视觉api实现卡路里识别
这个功能实现很早了，在开源项目 `PeachAssistant` 里已经有体现了，利用CMP跨平台技术在Android和IOS平台均完成了功能的集成。现记录一下api的介绍和两个平台的执行流程。
## 豆包API介绍
首先在个人控制台的服务管理页面开通视觉识别api权限，获取 **API_KEY** ，baseURL为 **`https://ark.cn-beijing.volces.com/api/v3`** 。图片可以使用url或者文件文件base64编码上传两种方案。

如果你要传入的图片/视频在本地，你可以将这个其转化为 **Base64** 编码，然后提交给大模型。下面是一个简单的示例代码。

**传入 Base64 编码格式时，请遵循以下规则**：

```
传入的是图片：
格式遵循data:image/<图片格式>;base64,<Base64编码>，其中，
图片格式：jpeg、png、gif等，支持的图片格式详细见图片格式说明。
Base64 编码：图片的 Base64 编码。

传入的是视频：
格式遵循data:video/<视频格式>;base64,<Base64编码>，其中，
视频格式：MP4、AVI等，支持的视频格式详细见视频格式说明。
Base64 编码：视频的 Base64 编码。
```

请求实例：

```bash
BASE64_IMAGE=$(base64 < path_to_your_image.jpeg) && curl https://ark.cn-beijing.volces.com/api/v3/chat/completions \
   -H "Content-Type: application/json"  \
   -H "Authorization: Bearer $ARK_API_KEY"  \
   -d @- <<EOF
   {
    "model": "doubao-seed-1-6-251015",
    "messages": [
      {
        "role": "user",
        "content": [
            {
            "type": "image_url",
            "image_url": {
              "url": "data:image/jpeg;base64,$BASE64_IMAGE"
            },
            {
            "type": "text",
            "text": "图里有什么"
            }
        ]
      }
    ],
    "max_tokens": 300
    }
EOF
```

可以通过 detail 字段控制图片理解的精细度。
* low：“低分辨率”模式，默认此模式，处理速度会提高，适合图片本身细节较少或者只需要模型理解图片大致信息或者对速度有要求的场景。此时 min_pixels 取值3136、max_pixels 取值1048576，超出此像素范围且小于3600w px的图片（超出3600w px 会直接报错）将会等比例缩放至范围内。
* high：“高分辨率”模式，这代表模型会理解图片更多的细节，但是处理图片速度会降低，适合需要模型理解图像细节，图像细节丰富，需要关注图片细节的场景。此时 min_pixels 取值3136、max_pixels 取值4014080，超出此像素范围且小于3600w px的图片（超出3600w px 会直接报错）的图片将会等比例缩放至范围内。

例如：

```bash
curl https://ark.cn-beijing.volces.com/api/v3/chat/completions \
   -H "Content-Type: application/json" \
   -H "Authorization: Bearer $ARK_API_KEY" \
   -d '{
    "model": "doubao-seed-1-6-251015",
    "messages": [
        {
            "role": "user",
            "content": [                
                {"type": "image_url","image_url": {"url":  "https://ark-project.tos-cn-beijing.volces.com/doc_image/ark_demo_img_1.png"},"detail": "high"},
                {"type": "text", "text": "支持输入图片的模型系列是哪个？"}
            ]
        }
    ],
    "max_tokens": 300
  }'
```

## KMP公共网络请求

```kotlin
class DoubaoVisionRepository(private val ktorClient: KtorClient) {

    companion object {
        const val BASE_URL =
            "https://ark.cn-beijing.volces.com/api/v3"
        const val VISION_SYSTEM_PROMT =
            "下图是一张食物图片，请你计算每种食物的重量和卡路里，返回一个json，其中name为String，weight为Int，calorie为Int（单位千卡），json格式：\n" +
                    "{\n" +
                    "  \"foods\": [\n" +
                    "    {\n" +
                    "      \"name\": \"食物名称\",\n" +
                    "      \"weight\": \"食物重量\",\n" +
                    "      \"calorie\": \"食物卡路里\"\n" +
                    "    }\n" +
                    "  ]\n" +
                    "}"
        const val API_KEY = "xxxxxxxxxxxx"
        const val MODEL_NAME = "doubao-1-5-vision-pro-32k-250115"
    }

    suspend fun calCalorieByAI(imageType: String, imageBase64:String) = withContext(Dispatchers.IO) {
        ktorClient.client.post("${BASE_URL}/chat/completions") {
            // 配置请求头
            headers {
                append("Content-Type", "application/json")
                append("Authorization", "Bearer $API_KEY")
            }
            setBody(
                DoubaoVisionRequest(
                    model = MODEL_NAME,
                    messages = listOf(
                        DoubaoRequestMessage(
                            role = ChatRole.SYSTEM.roleDescription,
                            content = listOf(
                                DoubaoVisionContent(
                                    type = "text",
                                    text = VISION_SYSTEM_PROMT,
                                ),
                                DoubaoVisionContent(
                                    type = "image_url",
                                    image_url = ImageUrl(
                                        url = "data:image/$imageType;base64,$imageBase64"
                                    ),
                                )
                            )
                        ),
                    )
                )
            )
        }.body<DoubaoVisionResponse>()
    }
}
```

## Android实现

### 权限申请

### 相册上传

### 实时拍照上传

## IOS实现

### 权限申请

### 相册上传

### 实时拍照上传
