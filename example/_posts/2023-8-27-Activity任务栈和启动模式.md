---
layout: post
description: > 
  本文为Activity任务栈和四种启动模式的介绍
image: 
  path: /assets/img/blog/blogs_activity_stack.png
  srcset: 
    1920w: /assets/img/blog/blogs_activity_stack.png
    960w:  /assets/img/blog/blogs_activity_stack.png
    480w:  /assets/img/blog/blogs_activity_stack.png
accent_image: /assets/img/blog/blogs_activity_stack.png
excerpt_separator: <!--more-->
sitemap: false
---
# Activity任务栈和启动模式
## 任务栈
应用任务栈定义
Task是用户在执行某项工作时与之互动的一系列 Activity 的集合。这些 Activity 按照每个 Activity 打开的顺序排列在一个 TaskRecord 中。TaskRecord即任务栈，或者叫返回堆栈 ( back Stack ) ，是一种栈的数据结构，按照“后进先出”的规则管理着其中的元素。
Task的前后台切换
应用冷启动时，系统首先会创建一个Task name 为应用包名的一个Task，应用的主Activity放入栈底作为根Activity，后面打开新的Activity就会依次压入栈中。用户按Home键退回桌面，这个Task就退回后台，在后台时，Task 中的所有 Activity 都会停止，但Task 的 TaskRecord 会保持不变，这样一来，用户再次点击图标时，Task 就可以返回到“前台”，以便用户可以从他们离开的地方继续操作。如果用户是按返回键，则Task内的Activity就会被一个个推出去，最后根Activity也退出，这个Task就不存在了。
launchmode
Task里的顺序不会变，只会进出。所以用户多次打开同一个界面时，默认会重复创建多次这个Activity。在按返回键时又会依次显示。想改变这种默认的方式，可以通过定义启动模式来确定 Activity 的新实例如何与当前的 Task 关联。
相关的属性：

```xml
<activity
    android:taskAffinity="string"
    android:allowTaskReparenting=["true" | "false"]
    android:alwaysRetainTaskState=["true" | "false"]
    android:clearTaskOnLaunch=["true" | "false"]
    android:finishOnTaskLaunch=["true" | "false"]
    android:launchMode=["standard" | "singleTop" | "singleTask" | "singleInstance"]/>
```

taskAffinity亲和性表示这个Activity更倾向于加载到哪一个栈里。一般都默认设置成包名所对应这个Task里。
allowTaskReparenting当下一次将启动 Activity 的 Task 转至前台时，Activity 是否能从该 Task 转移至与其有相似性的 Task 。Activity启动时会默认记录在拉起它的那个Task内，这个属性如果是true，那么下次这个Activity亲和性对应的Task被创建出来的时候，它就会切到其倾向的Task的栈顶，如果这个属性为false，那其就会留在这个Task内。

## launchMode
主要有四种：
1. standard，默认模式。每次启动一个 Activity 都会重新创建一个新的实例，不管这个实例之前是否已经存在。
2. singleTop，栈顶复用模式。该模式下，如果待启动的 Activity 已经位于Task 的栈顶，则该 Activity 的实例不会被创建，而是直接复用已有的实例，同时该 Activity 的 onNewIntent( ) 方法会被回调。如果待启动的 Activity 的实例不是位于Task 的栈顶，那么系统会为这个 Activity 创建新的实例，并将其放入 Task 中。singletop应用：避免多次创建，比如点击一个按钮启动一个activity，如果快速点击多次会导致反复启动，一种办法是在点击事件里过滤，另一个办法是设置目标activity是singletop
3. singleTask栈内复用，是一个单实例模式，如果目标activity在栈顶，则直接复用，如果其在栈内不在栈顶，则把它上面的activity全部推出栈，让其显示。如果栈内没有，则比较当前Task和目标Activity的taskAffinity，二者一致则在这个栈内创建它，如果不一致，则为其新开一个栈，其作为这个新Task里的根Activity。一般从主页跳转到各个Activity，都希望可以只保留一个主页的实例，而不是多个。AS创建的项目，MainActivity的默认配置就是singleTask。
4. singleInstance，强制的单实例模式。该模式下，Task 栈里面只能有这一个 Activity 。当 Activity 被启动后，系统会为它创建一个新的 Task 栈，然后把该 Activity 的实例压入该 Task 栈中，并且该 Task 栈不允许其他 Activity 压入其中。该 Activity 实例是 Task 中唯一的 Activity。比如首次进入应用的介绍页，就是一个单实例，跳转到其他Activity后不允许再返回到这个界面。