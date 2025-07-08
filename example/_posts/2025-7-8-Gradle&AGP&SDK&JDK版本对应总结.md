---
layout: post
description: > 
  本文介绍了Gradle版本，Gradle插件版本，SDK版本，JDK版本的对应关系
image: 
  path: /assets/img/blog/blogs_gradle_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_gradle_cover.png
    960w:  /assets/img/blog/blogs_gradle_cover.png
    480w:  /assets/img/blog/blogs_gradle_cover.png
accent_image: /assets/img/blog/blogs_gradle_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Gradle&AGP&SDK&JDK版本对应总结
来自腾讯元宝AI的总结，检查无大问题后创建文章。
在 Android 开发中，Gradle、Gradle 插件（Android Gradle Plugin，简称 AGP）、JDK 和 Android SDK 的版本之间有一定的兼容性要求。以下是它们之间的对应关系（截至 2024 年 6 月，基于官方文档和社区实践）：

---

## 一、Gradle 插件版本（AGP）与 Android Studio 版本对应关系

| Android Studio 版本 | AGP（Gradle Plugin）版本 |
|---------------------|--------------------------|
| Android Studio Flamingo (2022.2.1) | 8.0.x |
| Android Studio Giraffe (2022.3.1) | 8.1.x |
| Android Studio Hedgehog (2023.1.1) | 8.2.x |
| Android Studio Iguana (2023.2.1) | 8.3.x |
| Android Studio Jellyfish (2023.3.1) | 8.4.x |
| Android Studio Koala (2024.1.1) | 8.5.x |

> **注意**：Android Studio 的每个版本通常会**推荐**使用特定版本的 AGP，建议尽量使用官方推荐的组合，以避免兼容性问题。

---

## 二、AGP 版本与 Gradle 版本对应关系

| AGP 版本 | 推荐的 Gradle 版本 |
|----------|---------------------|
| 8.5.x    | 8.7 - 8.9           |
| 8.4.x    | 8.5 - 8.7           |
| 8.3.x    | 8.3 - 8.5           |
| 8.2.x    | 8.0 - 8.3           |
| 8.1.x    | 7.6 - 8.0           |
| 8.0.x    | 7.5 - 7.6           |
| 7.4.x    | 7.5 - 7.6           |
| 7.3.x    | 7.4 - 7.5           |
| 7.2.x    | 7.3 - 7.4           |
| 7.1.x    | 7.2 - 7.3           |
| 7.0.x    | 7.0 - 7.2           |

> **说明**：Gradle 版本过高或过低都可能导致构建失败，务必参考https://developer.android.com/studio/releases/gradle-plugin#updating-gradle确认兼容性。

---

## 三、AGP 版本与 JDK 版本对应关系

| AGP 版本 | 推荐的 JDK 版本 |
|----------|------------------|
| 8.5.x    | JDK 17           |
| 8.4.x    | JDK 17           |
| 8.3.x    | JDK 17           |
| 8.2.x    | JDK 17           |
| 8.1.x    | JDK 11 或 JDK 17 |
| 8.0.x    | JDK 11 或 JDK 17 |
| 7.4.x 及以下 | JDK 11         |

> **注意**：从 AGP 8.0 开始，**官方推荐使用 JDK 11**，但从 AGP 8.1 起，**JDK 17 也得到支持**。使用 JDK 8 已不再被支持（从 AGP 7.0 起已移除对 JDK 8 的支持）。

---

## 四、AGP 版本与 Android SDK 版本（compileSdkVersion）对应关系

AGP 本身并不直接限制 `compileSdkVersion`，但为了兼容性和稳定性，建议：

- **AGP 8.x**：建议使用 `compileSdkVersion 34`（Android 14）或更高。
- **AGP 7.x**：建议使用 `compileSdkVersion 33`（Android 13）或 `34`（视情况）。
- **AGP 4.x - 6.x**：建议使用 `compileSdkVersion 30 - 32`。

> **注意**：`compileSdkVersion` 是你用来编译项目的 SDK 版本，它**不需要**与你的 `targetSdkVersion` 相同，但建议尽量保持一致或接近，以确保兼容性。

---

## 五、总结表格（AGP 8.5.x 为例）

| 项目 | 推荐版本 |
|------|----------|
| Android Studio | Jellyfish (2023.3.1) 或 Koala (2024.1.1) |
| AGP（Gradle Plugin） | 8.5.x |
| Gradle | 8.7 - 8.9 |
| JDK | JDK 17 |
| compileSdkVersion | 34（Android 14）或更高 |

---

## 六、如何查看当前项目的版本信息

1. **Gradle 版本**：
   - 打开项目根目录下的 `gradle/wrapper/gradle-wrapper.properties` 文件，查看 `distributionUrl`，例如：
     ```
     distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
     ```

2. **AGP 版本**：
   - 打开项目根目录下的 `build.gradle`（Project 级别）文件，查看：
     ```groovy
     dependencies {
         classpath "com.android.tools.build:gradle:8.5.0"
     }
     ```

3. **JDK 版本**：
   - 在 Android Studio 中：`File > Project Structure > SDK Location > JDK location`
   - 或者在终端运行 `./gradlew -v` 查看使用的 JDK 版本。

4. **compileSdkVersion**：
   - 在模块级别的 `build.gradle` 文件中查看：
     ```groovy
     android {
         compileSdkVersion 34
     }
     ```

多一条备注：

> 在 Android Studio 中使用GUI界面进行assemble，build等操作时，JDk版本是在设置里选取的对应的版本。如果是在Terminal或者其他环境下使用 `./gradlew assembleRelease` 等命令时，所使用的JDK版本是在系统的环境变量中配置的。
