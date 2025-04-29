---
layout: post
description: > 
  本文介绍了三种比较热门的跨平台技术的横向对比，涉及性能，易用性，开发成本等方面
image: 
  path: /assets/img/blog/blogs_cover_multiplatform.png
  srcset: 
    1920w: /assets/img/blog/blogs_cover_multiplatform.png
    960w:  /assets/img/blog/blogs_cover_multiplatform.png
    480w:  /assets/img/blog/blogs_cover_multiplatform.png
accent_image: /assets/img/blog/blogs_cover_multiplatform.png
excerpt_separator: <!--more-->
sitemap: false
---
# Flutter & RN & CMP三种跨平台对比
## 历史背景
### Flutter
Flutter的历史最早可以追溯到2014年10月，其前身是Google内部孵化的Sky项目。其是一款跨平台移动应用开发框架，它允许开发者使用单一代码库同时构建iOS和Android应用。Flutter采用了Dart编程语言，这是一种面向对象的、类型安全的编程语言，与JavaScript非常相似。Flutter的主要优势在于其快速的开发速度和流畅的用户体验。

具体的：
* 2014.10 - Flutter的前身Sky在GitHub上开源；
* 2015.10 - 经过一年的开源，Sky正式改名为Flutter；
* 2017.5 - Google I/O正式向外界公布了Flutter，这个时候Flutter才正式进去大家的视野；
* 2018.6 - 距5月Google I/O 1个月的时间，Flutter1.0预览版；
* 2018.12 - Flutter1.0发布，它的发布将大家对Flutter的学习和研究推到了一个新的起点；
* 2019.2 - Flutter1.2发布主要增加对web的支持。

### React Native
React Native是Facebook于2015年发布的一款跨平台移动应用开发框架，它允许开发者使用JavaScript和React来构建iOS和Android应用。React Native的主要优势在于其灵活的组件化开发方式和丰富的第三方库支持。
### Compose Multiplatform
Compose Multiplatform是JetBrains于2021年发布的一款跨平台移动应用开发框架，它允许开发者使用Kotlin和Jetpack Compose来构建iOS和Android应用。Compose Multiplatform的主要优势在于其简洁的语法和强大的UI组件库。
## 开发流程
### Flutter
Flutter的开发流程相对简单，开发者只需要使用Dart语言编写应用程序，然后使用Flutter SDK进行编译和打包即可。Flutter的开发流程包括以下几个步骤：
* 编写Dart代码：开发者使用Dart语言编写应用程序的业务逻辑和界面。
* 编译和打包：开发者使用Flutter SDK进行编译和打包，生成iOS和Android应用程序。
* 运行应用程序：开发者可以使用模拟器或真机运行应用程序。
### React Native
React Native的开发流程相对复杂，开发者需要使用JavaScript和React编写应用程序，然后使用React Native CLI进行编译和打包。React Native的开发流程包括以下几个步骤：
* 编写JavaScript代码：开发者使用JavaScript和React编写应用程序的业务逻辑和界面。
* 编译和打包：开发者使用React Native CLI进行编译和打包，生成iOS和Android应用程序。
* 运行应用程序：开发者可以使用模拟器或真机运行应用程序。
### Compose Multiplatform
Compose Multiplatform的开发流程相对简单，开发者只需要使用Kotlin和Jetpack Compose编写应用程序，然后使用Compose Multiplatform CLI进行编译和打包。Compose Multiplatform的开发流程包括以下几个步骤：
* 编写Kotlin代码：开发者使用Kotlin和Jetpack Compose编写应用程序的业务逻辑和界面。
* 编译和打包：开发者使用Compose Multiplatform CLI进行编译和打包，生成iOS和Android应用程序。
* 运行应用程序：开发者可以使用模拟器或真机运行应用程序。