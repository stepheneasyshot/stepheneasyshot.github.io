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
Activity的任务栈Task Stacks，是用户在执行某项工作时与之互动的一系列 Activity 的集合。

这些 Activity 按照每个 Activity 打开的顺序排列在一个 TaskRecord 中。TaskRecord即任务栈，或者叫返回堆栈 ( back Stack ) ，是一种栈的数据结构，按照“后进先出”的规则管理着其中的元素。

### Task的前后台切换
一般来说，每一个app都有自己的任务栈，应用冷启动时，系统首先会创建一个Task name 为应用包名的一个Task，应用的主Activity放入栈底作为根Activity，后面打开新的Activity就会依次压入栈中。

* 用户按Home键退回桌面，这个Task就退回后台，在后台时，Task 中的所有 Activity 都会停止，但Task 的 TaskRecord 会保持不变，这样一来，用户再次点击图标时，Task 就可以返回到“前台”，以便用户可以从他们离开的地方继续操作。
* 如果用户是按返回键，则Task内的Activity就会被一个个推出去，最后根Activity也退出，这个Task就不存在了。

Task里的顺序不会变，只会进出。所以用户多次打开同一个界面时，默认会重复创建多次这个Activity。在按返回键时又会依次显示。想改变这种默认的方式，可以通过定义启动模式来确定 Activity 的新实例如何与当前的 Task 关联。

## Activity属性 
Activity在Manifest里可配置的相关的属性：

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

## LaunchMode
主要有四种：
### standard(标准模式)
是Activity默认的启动模式，即当前Activity未显式设置启动模式情况下，其启动模式为standard。在standard模式下，每次启动该Activity系统都会创建一个新的实例（instance）并压入当前任务栈顶，不论是否已有相同Activity实例存在。Activity实例数量没有限制，具体取决于任务栈深度。该模式适用于大多数情景，例如在一个应用中打开多个页面。

### SingleTop
当Activity启动模式设置为singleTop时，系统启动该Activity时若发现当前任务栈栈顶 **（并非系统全局）** 为该Activity实例，则会直接复用该实例（调用onNewIntent()方法）而不会重新创建；若当前任务栈栈顶不为该Activity实例时，则会创建新的实例无论当前或其他任务栈中是否已存在对应实例。该模式适用于在同一任务中频繁跳转回当前页面的场景，例如从通知点击进入消息页面。

singletop的应用：避免多次创建，比如点击一个按钮启动一个activity，如果快速点击多次会导致反复启动，一种办法是在点击事件里过滤，另一个办法是设置目标activity是singletop

### SingleTask
当Activity启动模式设置为singleTask时，系统启动该Activity时若发现存在一个任务栈 **（系统全局）** 中存在该Activity实例时，则会直接复用该实例（调用onNewIntent()方法）而不会重新创建，同时将该任务栈中位于该实例之上的其它Activity实例都将会被弹出栈，该实例作为当前任务栈栈顶；

若所有任务栈中都不存在该Activity实例时，则会在当前任务栈中创建该Activity实例作为栈顶。该实例在所有任务栈中唯一存在。该模式适用于“主页”类Activity，或者需要保证该Activity在整个应用中只有一个实例的场景，如应用的主界面、设置界面。

一般从主页跳转到各个Activity，都希望可以只保留一个主页的实例，而不是多个。AS创建的项目，MainActivity的默认配置就是singleTask。

### SingleInstance
 当Activity启动模式设置为singleInstance时， **系统全局** 只允许存在该Activity的一个实例，并且该Activity将独占一个任务栈。
 
 系统启动该Activity时，若所有任务栈中都不存在该Activity实例时，系统会使用一个独立的任务栈，并在该栈中创建该Activity实例，该任务栈只用于管理该Activity实例，唯一存在；当该Activity实例已经存在于独立的任务栈中时，系统会直接复用该实例，该实例在所有任务栈中唯一存在。
 
 该模式适用于锁屏页面、视频播放等需要独立管理的Activity，防止干扰。

，强制的单实例模式。该模式下，Task 栈里面只能有这一个 Activity 。当 Activity 被启动后，系统会为它创建一个新的 Task 栈，然后把该 Activity 的实例压入该 Task 栈中，并且该 Task 栈不允许其他 Activity 压入其中。该 Activity 实例是 Task 中唯一的 Activity。比如首次进入应用的介绍页，就是一个单实例，跳转到其他Activity后不允许再返回到这个界面。

## 两种配置方式

### 在AndroidManifest.xml中配置
```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTop" />
```

### 在代码中配置
进行Activity的跳转时，一般像下面这样写：

```java
Intent intent = new Intent(this, MainActivity.class);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
startActivity(intent);
```

我们可以通过为Intent添加特定的Flag（标志）来指定Activity的启动行为。这些Flag会影响Activity的启动模式和任务栈的管理。示例如下：

#### 常用Flag介绍
* FLAG_ACTIVITY_NEW_TASK：这个标志会告诉系统为新启动的 Activity 创建一个新的任务栈（Task），如果该 Activity 已经存在于某个栈中，它将复用现有的任务栈并将 Activity 添加到栈顶。该标志在启动一个新的 Activity 时是强制性的，特别是在从非 Activity 上下文（如 Service 或 BroadcastReceiver）启动 Activity 时。
* FLAG_ACTIVITY_SINGLE_TOP：如果启动的 Activity 已经在栈顶，则复用它的实例，而不是创建新的实例。如果该 Activity 不在栈顶，系统将创建一个新的实例并将其推到栈顶。这个标志类似于在 launchMode 中设置 singleTop。
* FLAG_ACTIVITY_CLEAR_TOP：当启动的 Activity 已经存在于任务栈中时，它会清除这个 Activity 之上的所有其他 Activity。然后，系统会将这个 Activity 置于栈顶并复用它的实例。这通常与 * FLAG_ACTIVITY_NEW_TASK 结合使用。
* FLAG_ACTIVITY_CLEAR_TASK：这个标志会清除目标 Activity 所在的整个任务栈（Task）。所有位于该任务栈中的 Activity 都会被销毁，然后启动目标 Activity 作为新的栈根。
* FLAG_ACTIVITY_REORDER_TO_FRONT：如果目标 Activity 已经在任务栈中，但不是栈顶，系统会将它移动到栈顶，而不销毁栈中的其他 Activity，也不会创建新的实例。
* FLAG_ACTIVITY_NO_HISTORY：该标志表示启动的 Activity 不会被添加到任务栈中，也就是说当用户离开这个 Activity 后，系统将不会保存它的状态。这常用于启动短暂的页面，如登录页面、广告页面等。
* FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：这个标志会防止启动的 Activity 出现在“最近任务”（Recents）中。即使用户通过多任务按钮查看任务列表，该 Activity 也不会出现在列表中。

## Activity跳转
生命周期流程，可以参考 [【APP冷启动流程解析】](./2024-9-21-APP冷启动流程解析.md){:.heading.flip-title} 

AMS会判断拉起的Activity是否为当前Activity，如果不是，当前Activity会先进入到pause状态，执行onPause()，然后再去创建新的Activity，走目标Activity的onCreate()，onResume()生命周期。然后是当前Activity的onStop()生命周期。

即如果是Activity A 跳转到 Activity B，那么完整的流程是：

```
A的onPause() -> B的onCreate() -> B的onStart() -> B的onResume() -> A的onStop()。
```

需要注意的是，A的onPause方法如果有耗时操作，并不会拖慢B的生命周期的执行，尽管在系统层面，是先通知A去执行onPause方法，然后再去执行B的生命周期方法。

但是B的周期不依赖A的onPause方法执行完毕，所以B的生命周期方法是可以并行执行的。
