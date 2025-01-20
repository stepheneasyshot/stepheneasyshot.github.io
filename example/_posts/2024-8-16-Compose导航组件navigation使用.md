---
layout: post
description: > 
  本文介绍了Jetpack Compose里页面跳转的navigation组件的使用
image: 
  path: /assets/img/blog/blogs_compose_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_compose_cover.png
    960w:  /assets/img/blog/blogs_compose_cover.png
    480w:  /assets/img/blog/blogs_compose_cover.png
accent_image: /assets/img/blog/blogs_compose_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Compose导航组件navigation使用
Compose导航组件是Jetpack Compose中的一个重要组件，用于管理应用程序中的页面导航流程。它提供了一种简单而灵活的方式来管理不同的屏幕和页面之间的导航。

之前的View架构一般是单Activity，多个Fragment，或者多Activity模式。Compose则是多Activity，多个Composable。

页面跳转单方式有很多，官方推荐的是使用Navigation组件。

## 依赖配置
主要有三个地方：
* 首先是navigation-compose组件的依赖配置。
* 界面在导航时有传参数的需求的话，需要使用kotlin的序列化注解来标注数据类或者单例类，需要配置kotlin的序列化插件。
* 最后是序列化的依赖配置。

```toml
[versions]
kotlin = "2.1.0"
navigation = "2.8.5"
serialization = "1.7.3"

[libraries]
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }

serialization = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version = "serialization"}

[plugins]
jetbrains-kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

## Navigation三要素

### NavHost

包含当前导航目的地的界面元素。也就是说，当用户浏览应用时，该应用实际上会在导航宿主中切换目的地。

### NavGraph
一种数据结构，用于定义应用中的所有导航目的地以及它们如何连接在一起。

### NavController

用于管理目的地之间导航的中央协调器。该控制器提供了一些方法，可在目的地之间导航、处理深层链接、管理返回堆栈等。

类比我们开车的场景，NavHost就是车，NavGraph就是路，NavController就是司机。

首先在起始地点，然后确定路线，然后司机控制车去往目的地。

## 使用
第一步，起始地点，在应用中，就是应用的首页，开屏进入之后的第一个页面。随便取一个HomePage

```kotlin
@Composable
fun HomePage() {
    Column {
        Text(text = "Home Page")
        Button(onClick = { /*TODO*/ }) {
            Text(text = "Go to Detail")
        }
    }
}
```

第二步，确定路线，也就是NavGraph，定义导航图。这里需要先定义好需要跳转的页面。

```kotlin

@Composable
fun HomePage(homeDate: HomeData, homeToAbout: () -> Unit) {
    Box(modifier = Modifier.fillMaxSize(1f), contentAlignment = Alignment.Center) {
        Column {
            Text(text = "HomePage data: ${homeDate.name}")
            Button(onClick = homeToAbout) {
                Text(text = "HomeToAbout")
            }
        }
    }
}

@Composable
fun AboutPage(backStack: () -> Unit) {
    Box(
        modifier = Modifier.fillMaxSize(1f),
        contentAlignment = Alignment.Center
    ) {
        Text(text = "AboutPage")
        Button(onClick = backStack) {
            Text(text = "Goto HomePage")
        }
    }
}
```

导航过程中传参数和页面标记，我们定义两个数据类来标记：

```kotlin
@Serializable
data class HomeData(val name: String)

@Serializable
object About
```

创建导航图：

```kotlin
val navController = rememberNavController()
val graph = remember {
    navController.createGraph(startDestination = HomeData("initial data")) {
        composable<HomeData> { navBackStackEntry ->
            val homeData = navBackStackEntry.toRoute<HomeData>()
            HomePage(homeDate = homeData) {
                navController.navigate(About)
            }
        }
        composable<About> {
            AboutPage {
                navController.navigate(HomeData("about page to home page"))
            }
        }
    }
}
NavHost(navController = navController, graph = graph)
```

使用时，更简化的写法可以像下面这样。

直接将NavGraph的第二个参数放在末尾，NavHost后面写成lambda的形式。

```kotlin
NavHost(navController = navController, startDestination = ScreenTitle.Home.name) {
    composable(route = ScreenTitle.Home.name) {
        HomeScreen(
            weatherScreenState,
            onNavToAbout = { navController.navigate(ScreenTitle.About.name) },
            onNavToAuthor = { navController.navigate(ScreenTitle.Author.name) })
    }
    composable(route = ScreenTitle.About.name) {
        AboutScreen(onBack = { navController.popBackStack() })
    }
    composable(route = ScreenTitle.Author.name) {
        AuthorScreen(onBack = { navController.popBackStack() })
    }
}

```

为了统一管理提高可扩展性，我们可以使用一个密封类来管理所有的页面的导航路由数据。

```kotlin
@Serializable
sealed class Screen(val route: String) {
    @Serializable
    object MainPage : Screen("mainPage")

    @Serializable
    object ArticlePage : Screen("articlePage")

    @Serializable
    object PicturePage : Screen("picturePage")

    @Serializable
    object ElsePage : Screen("elsePage")
}
```