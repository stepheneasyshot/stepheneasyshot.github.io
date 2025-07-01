---
layout: post
description: > 
  本文介绍了Google推出的UI设计系统——Material Design的设计理念和配置方式。
image: 
  path: /assets/img/blog/blogs_materialdesign3_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_materialdesign3_cover.png
    960w:  /assets/img/blog/blogs_materialdesign3_cover.png
    480w:  /assets/img/blog/blogs_materialdesign3_cover.png
accent_image: /assets/img/blog/blogs_materialdesign3_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Material Design设计理念和配置方式
**Material Design（材料设计）** 是由 Google 在 2014 年推出的设计语言，旨在为多平台（包括移动端、Web、桌面端等）提供统一、直观且具有物理感的视觉与交互体验。它融合了现实世界的物理规律（如阴影、层次感）与数字交互的灵活性，强调“材料”作为设计的基本单元，通过清晰的视觉层次、响应式动画和一致的交互逻辑，提升用户对产品的认知效率与情感共鸣。

目前已经不仅仅在 Android 系统上应用，还扩展至 Web（Material Web）、Flutter（跨平台开发框架）、iOS（通过 Material Components for iOS）等平台，实现真正的“一次设计，多端适配”。

共推出过三个大版本：
   - **Material Design 1.0（2014）**：奠定基础，强调卡片、阴影和动效。  
   - **Material Design 2.0（2018，Material Theming）**：引入动态主题（Dynamic Color），支持品牌个性化定制。  
   - **Material Design 3（2021，Material You）**：以用户为中心的设计，核心是“自适应色彩”（用户可基于壁纸提取主色，系统自动适配界面配色），增强个性化与无障碍体验。

Google 提供了丰富的设计资源（如 [Material Design Guidelines](https://m3.material.io/)、Figma 组件库），开发者社区（如 GitHub）也有大量开源实现。  

对于设计师，设计工具诸如 Adobe XD、Sketch等，也可以集成 Material 插件，支持快速原型设计。

Material Design以其统一的设计语言降低开发与设计成本，提升跨团队协作效率。同时，自适应设计与动态主题的特点，可以满足多设备、个性化需求。

但是，也有一大部分设计师和开发者认为其规范过于严格，可能限制创新设计。而且过度依赖阴影和层次感可能导致界面视觉复杂度增加（尤其在低配设备上）。

### **一、核心理念：数字世界的“材料”隐喻**
Material Design 的核心是将数字界面类比为现实世界中的“材料”（如纸张、墨水），但赋予其数字化特性（如无限延展性、动态响应）。这种隐喻并非追求完全模拟物理世界，而是通过“材料”的抽象化表达（如阴影表示层次、颜色传递状态），让用户快速理解界面元素的逻辑关系，降低认知成本。
### **二、设计原则：四大支柱**
1. **真实感与层次感（Material as Metaphor）**  
   - 以“材料”为基本单元，通过阴影深度（elevation）表现元素的层级关系（如卡片浮于背景之上）。  
   - 颜色、形状和动效模拟现实中的光影变化，增强界面的物理真实感，但避免过度拟物化。

2. **直观的动效（Bold, Graphic, Intentional）**  
   - 动效不仅是装饰，而是传递信息的工具。例如，按钮点击时的微缩反馈、页面切换时的滑动过渡，均需明确提示用户操作结果。  
   - 动效需符合物理规律（如惯性、缓动），避免突兀变化。

3. **响应式交互（Motion Provides Meaning）**  
   - 界面元素需对用户操作（点击、滑动、拖拽）做出即时反馈，例如输入框获得焦点时的放大效果、列表项拖动时的实时位置更新。  
   - 动效需引导用户注意力，帮助理解界面状态的变化（如加载中的进度指示）。

4. **跨平台一致性（Adaptive Design）**  
   - 通过统一的视觉语言（颜色、排版、图标）和交互模式，确保在不同设备（手机、平板、桌面、可穿戴设备）和场景（横屏、竖屏、暗黑模式）下保持体验连贯性。  
   - 支持自适应布局，根据屏幕尺寸动态调整组件排列（如网格系统的灵活适配）。

### **三、关键设计元素**
1. **材料（Material Surfaces）**  
   - 界面由多层“材料”构成，每层具有独立的阴影深度（elevation），通过阴影差表现叠加关系（如对话框浮于卡片之上）。  
   - 材料可伸缩、变形，但不可穿透（遵循现实世界的物理规则）。

2. **颜色与排版**  
   - **颜色**：以主色（Primary Color）和强调色（Accent Color）为核心，搭配中性色（黑白灰）构建层次。Google 提供了一套标准色板（如 Material Color System），支持动态配色（Dark Theme）。  
   - **排版**：基于 Roboto 字体（后扩展至其他开源字体），通过字号、字重（Bold/Light）和行高构建清晰的文本层级，确保可读性与信息密度平衡。

3. **图标与图形**  
   - 使用线性图标（Material Icons）和填充图标，强调简洁性与符号化表达。图标需符合用户认知习惯（如“菜单”用三条横线表示）。  
   - 图形设计注重几何形状（圆形、矩形）的组合，通过圆角、边框和阴影增强视觉层次。

4. **动效系统（Motion System）**  
   - **容器变换（Container Transform）**：元素变形时保持视觉连续性（如卡片展开为详情页）。  
   - **共享轴（Shared Axis）**：通过共用的运动方向（如左右滑动切换标签页）传递元素关联性。  
   - **淡入淡出（Fade）**：用于非关联元素的切换（如提示信息消失）。  
   - Google 提供了 Lottie 等工具支持复杂动效的实现。

### **四、应用场景与组件库**
Material Design 提供了一套完整的组件库（Material Components），涵盖常用 UI 元素的设计规范与代码实现，开发者可直接调用。核心组件包括：

- **导航**：底部导航栏（Bottom Navigation）、抽屉菜单（Drawer）、顶部应用栏（AppBar）。  
- **输入与反馈**：文本框（Text Field）、按钮（Button）、滑块（Slider）、对话框（Dialog）、Snackbar（轻量提示）。  
- **数据展示**：卡片（Card）、列表（List）、网格（Grid）、表格（Table）、图表（Charts）。  
- **高级组件**：底部表单（Bottom Sheets）、模态抽屉（Modal Drawer）、悬浮操作按钮（FAB）。

这些组件均遵循设计规范，支持跨平台适配（Android、iOS、Web），开发者可通过 Material Components 库（如 Android 的 Material Components for Android、Web 的 Material UI）快速集成。

### Compose项目使用
`Material Design` 的官网也可以自己选取设计元素，作为一个压缩包下载下来，里面有Color，Style等文件，放到项目中就可以直接使用。

也可以一步步地自己配置每一个参数的色值，集成 `ColorScheme` 即可。内部可配置的参数非常多，根据名字也可以猜到其使用场景。

```kotlin
@Immutable
class ColorScheme(
    val primary: Color,
    val onPrimary: Color,
    val primaryContainer: Color,
    val onPrimaryContainer: Color,
    val inversePrimary: Color,
    val secondary: Color,
    val onSecondary: Color,
    val secondaryContainer: Color,
    val onSecondaryContainer: Color,
    val tertiary: Color,
    val onTertiary: Color,
    val tertiaryContainer: Color,
    val onTertiaryContainer: Color,
    val background: Color,
    val onBackground: Color,
    val surface: Color,
    val onSurface: Color,
    val surfaceVariant: Color,
    val onSurfaceVariant: Color,
    val surfaceTint: Color,
    val inverseSurface: Color,
    val inverseOnSurface: Color,
    val error: Color,
    val onError: Color,
    val errorContainer: Color,
    val onErrorContainer: Color,
    val outline: Color,
    val outlineVariant: Color,
    val scrim: Color,
    val surfaceBright: Color,
    val surfaceDim: Color,
    val surfaceContainer: Color,
    val surfaceContainerHigh: Color,
    val surfaceContainerHighest: Color,
    val surfaceContainerLow: Color,
    val surfaceContainerLowest: Color,
)
```

ColorScheme 类定义了 MaterialTheme 里所有命名颜色参数，这些参数在设计应用界面时，用于确保颜色和谐、文字可读，并且能区分不同的 UI 元素和表面。

#### 主要颜色相关
* primary：主色，在应用的屏幕和组件里出现频率最高的颜色。
* onPrimary：主色上显示的文字和图标的颜色。
* primaryContainer：容器首选的色调颜色。
* onPrimaryContainer：显示在 primaryContainer 之上的内容颜色（及其状态变体）。
* inversePrimary：在需要反色方案的地方作为“主色”使用的颜色，例如 SnackBar 上的按钮。

#### 次要颜色相关
* secondary：次要颜色，用于突出和区分产品，适用于浮动操作按钮、选择控件、高亮选中文本、链接和标题等。
* onSecondary：次要颜色上显示的文字和图标的颜色。
* secondaryContainer：用于容器的色调颜色。
* onSecondaryContainer：显示在 secondaryContainer 之上的内容颜色（及其状态变体）。

#### 第三颜色相关
* tertiary：第三颜色，可用于平衡主色和次要颜色，或突出显示输入框等元素。
* onTertiary：第三颜色上显示的文字和图标的颜色。
* tertiaryContainer：用于容器的色调颜色。
* onTertiaryContainer：显示在 tertiaryContainer 之上的内容颜色（及其状态变体）。

#### 背景和表面颜色相关
* background：可滚动内容后面显示的背景颜色。
* onBackground：背景颜色上显示的文字和图标的颜色。
* surface：影响组件表面（如卡片、工作表和菜单）的颜色。
* onSurface：表面颜色上显示的文字和图标的颜色。
* surfaceVariant：与 surface 用途类似的另一种颜色选项。
* onSurfaceVariant：可用于 surface 之上内容的颜色（及其状态变体）。
* surfaceTint：用于应用色调高程的组件，叠加在 surface 之上。高程越高，该颜色的使用比例越大。
* inverseSurface：与 surface 形成强烈对比的颜色，适用于位于 surface 颜色表面之上的表面。
* inverseOnSurface：与 inverseSurface 对比度良好的颜色，适用于位于 inverseSurface 容器之上的内容。

#### 错误颜色相关
* error：错误颜色，用于指示组件中的错误，例如文本字段中的无效文本。
* onError：错误颜色上显示的文字和图标的颜色。
* errorContainer：错误容器首选的色调颜色。
* onErrorContainer：显示在 errorContainer 之上的内容颜色（及其状态变体）。

#### 边框和遮罩颜色相关
* outline：用于边界的微妙颜色，为了可访问性增加对比度。
* outlineVariant：用于装饰元素边界的实用颜色，在不需要强对比度时使用。
* scrim：遮挡内容的遮罩颜色。

#### 表面变体颜色相关
* surfaceBright：surface 的变体，无论在浅色还是深色模式下，始终比 surface 亮。
* surfaceDim：surface 的变体，无论在浅色还是深色模式下，始终比 surface 暗。
* surfaceContainer：影响组件容器（如卡片、工作表和菜单）的 surface 变体。
* surfaceContainerHigh：比 surfaceContainer 强调程度更高的容器 surface 变体，用于需要更多强调的内容。
* surfaceContainerHighest：比 surfaceContainerHigh 强调程度更高的容器 surface 变体，用于需要更多强调的内容。
* surfaceContainerLow：比 surfaceContainer 强调程度更低的容器 surface 变体，用于需要较少强调的内容。
* surfaceContainerLowest：比 surfaceContainerLow 强调程度更低的容器 surface 变体，用于需要较少强调的内容。

#### 使用colorscheme
一般来说，如果只要求适配系统自带的深浅两色，可以直接使用 `darkColorScheme()` 和 `lightColorScheme()` 这两个顶层方法创建即可，配置好其中需要的各个参数。

例如：

```kotlin
val DarkColorScheme = darkColorScheme(
    primary = Color(0xff484848),
    background = Color(0xFF010101),
    surface = Color(0xff303030),
    surfaceVariant = Color(0xff1d1d1d),
    onPrimary = Color(0xffffffff),
    secondary = Color(0xFF1a1a1a),
    tertiary = Color(0xff3d77c2),
    onSecondary = Color(0x99ffffff),
    error = Color(0x99e53c3c),
    errorContainer = Color(0xcce53c3c),
    onBackground = Color(0xff323232),
    onSurface = Color(0xff404040),
)
```

传到 `MaterialTheme` 可组合项内：

```kotlin
MaterialTheme(
   colorScheme = when (themeState.value) {
      ThemeState.DARK -> DarkColorScheme
      ThemeState.LIGHT -> LightColorScheme
      else -> if (isSystemInDarkTheme()) DarkColorScheme else LightColorScheme
   }
) {
   // ...
}
```

如果需要自定义颜色，或者需要适配更多的颜色，也可以使用 `ColorScheme` 类。

切换主题时，动态更改 `MaterialTheme` 的参数，触发其重组，内部的所有子可组合项就会跟随刷新自己的样式。

在Android 8 及以后，可以使用 `WallpaperManager` 来获取壁纸的主颜色，应用到自己APP的界面上，下面是一段简单的测试代码。（应用版本低，使用的不是Material3）

```kotlin
object WallPagerThemeManager {
    private lateinit var wallpaperManager: WallpaperManager
    private var wallpaperColors: WallpaperColors? = null

    var DynamicColorScheme = darkColors()

    fun init() {
        wallpaperManager = WallpaperManager.getInstance(appContext)
        wallpaperColors = wallpaperManager.getWallpaperColors(WallpaperManager.FLAG_SYSTEM)
        val primaryColor = wallpaperColors?.primaryColor?.toArgb()
        val secondaryColor = wallpaperColors?.secondaryColor?.toArgb()
        primaryColor?.let {
            DynamicColorScheme = darkColors(
                primary = Color(it),
                secondary = Color(secondaryColor ?: it),
            )
        }
    }
}
```

