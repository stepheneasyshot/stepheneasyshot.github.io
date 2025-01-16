---
layout: post
description: > 
  本文介绍了Android平台两种Unity交互的通信架构和集成方式
image: 
  path: /assets/img/blog/blogs_unity.png
  srcset: 
    1920w: /assets/img/blog/blogs_unity.png
    960w:  /assets/img/blog/blogs_unity.png
    480w:  /assets/img/blog/blogs_unity.png
accent_image: /assets/img/blog/blogs_unity.png
excerpt_separator: <!--more-->
sitemap: false
---

# Android集成Unity的两种方式
## ​Android平台常见动效
现在市面上的形形色色Android客户端，为了更优的用户体验，我们开发的上游产品和交互往往会在界面里设计很多动效。传统的一页页的静态展示页面已经不足以满足用户的审美需求了。

而动效的分类也是花样百出的，以播放时机来说有点击触发，打开页面触发，还有可跟随手指的交互持续触发的等等。有时候一些和数据耦合性较大的动效甚至需要我们自己来手写复杂的自定义View，比如曲线图、图表类型。

而我日常碰到的大部分的动效需求，还是依赖UI设计的同时来制作提供的，像那些短时间单次的展示类动效，往往实现方式比较随意，对资源的格式要求也不太严苛。一般有以下几种方案:

### 帧动画
在Android中，帧动画是通过Drawable动画实现的。你可以创建一个AnimationDrawable对象，然后在XML中定义一系列的帧（frames），每帧可以是一个Drawable资源。然后在代码中启动这个动画。

注意确保你的每个Drawable资源的尺寸是一致的，以便在动画过程中保持帧的正确显示。

### PAG动画
pag相较于上面的帧动画对性能更加友好。PAG是腾讯公司自主研发的一套完整动画工作流解决方案。 PAG诞生于2016年，最初的原因是为了解决更为复杂的视频编辑场景下动画渲染问题，同时又覆盖了UI动画和直播场景，于2022年1月在Github开源。

其使用方法可以说相当简单，只需要先从github主页确定版本，到gradle里引入依赖，
```
implementation 'com.tencent.tav:libpag:3.2.7.40'
```
然后在我们应用的xml布局中放置pagView，没有额外的属性需要配置。最后在代码里设置其文件源，循环方式，调用播放即可。

### MP4动画
直接获取mediaplayer实例，绑定surfaceView或者TextureView，再填文件，播放视频即可。需要关注的是surfaceView播放视频一开始可能会有黑屏问题，可以用静态图占位。

## 可交互3D动效
### Kanzi动效
跟手可互动的动效，也不得不谈kanzi动效。以下介绍来自百科与官网：

Kanzi产品是行业领先的3D引擎和UI开发工具，支持高效率沉浸式3D效果，跨系统多屏互联并能与安卓生态完美融合，已经成为全球主流车厂智能座舱首选的UI开发工具和引擎。更新后的Kanzi架构可与安卓操作系统、生态系统深度兼容。Kanzi可基于安卓的任何功能提供强大的图形设计支持，确保高质量的图像效果。

对于Kanzi动效的集成使用方式，因为没有自己从头开始对接，我只按照顺序一笔带过，有不对的地方欢迎指正。首先我们集成kanzi运行所需的Runtime.aar，kanziJava支持库aar，资源文件，资源列表的txt等等，还需要在gradle里写明不可压缩的文件类型，以防止无法加载资源。

在使用上，我们先在XML布局中声明，同时通过属性填入asstes里的资源名，和资源文件绑定：

```xml
 <com.rightware.kanzi.KanziTextureView
        android:id="@+id/tx_KanziSurfaceView"
        android:layout_width="@dimen/dp_2560"
        android:layout_height="@dimen/dp_1190"
        app:clearColor="@android:color/transparent"
        app:kzbPathList="climate.kzb"
        app:layout_constraintTop_toTopOf="parent"
        app:name="climate"
        app:startupPrefabUrl="kzb://climate/StartupPrefab"
        tools:ignore="MissingConstraints" />
```

在Java代码里我们需要设置通信的工具类，在里面添加监听器来接收和上行下行信号的交互：

```java
// 数据接口定义
public interface AndroidNotifyListener {
    void notifyDataChanged(String name, String value);

    void dataSourceFinish();
}

// 添加数据接收监听和下行通信
AndroidUtils.setListeners(this);
AndroidUtils.removeListeners(this);
AndroidUtils.setValue(SourceData.RightMidMove_up2down, y);
```
## Unity动效
Unity的大名在游戏界可谓如雷贯耳，记得小时候玩的很多游戏的开屏界面即有一个大大的Unity字样和图标。

以下介绍来自百科和官网：
```
Unity是实时3D互动内容创作和运营平台。包括游戏开发、美术、建筑、汽车设计、影视在内的所有创作者，借助Unity将创意变成现实。Unity平台提供一整套完善的软件解决方案，可用于创作、运营和变现任何实时互动的2D和3D内容，支持平台包括手机、平板电脑、PC、游戏主机、增强现实和虚拟现实设备。Unity作为全球领先的 3D 引擎之一，团结引擎可以为 3D HMI提供全栈支持。即为从概念设计到量产部署的整个 HMI 工作流程提供创意咨询、性能调优、项目开发等解决方案，从而为车载信息娱乐系统和智能驾驶座舱打造令人惊叹的交互式体验。
```
其实在第一版我们项目集成的是上面的Kanzi方案，其性能表现较Unity要差一些，关键是项目推进的过程中，对方工程师对动效样式的优化也达不到评测部门的要求，后来更新迭代我们就更换了Unity方案。而本文的重点也是在于Unity3D动效的使用，案例为车载IVI系统空调app的风向调节，交互逻辑比上面举的例子更加复杂，需要实时跟手，在交互热区范围内需要不断变化动效形态，并完成双向通信，保证动效和车载信号的一致性。

### Android应用对接Unity集成的两种方案
以下提到的集成方案均可以在Unity的官方网站进行更加详细的查阅：
```
https://docs.unity.cn/cn/tuanjiemanual/Manual/hmi-android.html
```
#### 通信协议制定
第一步不是创建工程和选取方案，而是要提前根据APP产品交互逻辑，来指定和Unity之间的通信协议。有哪些功能是开关，需要调整哪些属性。例如空调app里就涉及几个出风口的打开关闭，可以以0/1来区分。还有风口的方向调节，需要互传x,y坐标值。Android和Unity之间一般是采用JSON字符串来通信的。

而且，两方通信链路和Unity的集成方式还有关，像下面要谈到的第一种进程隔离方案，就是通过集成全量的Unity依赖包，利用aar内部JNI接口来通信的，而第二种Client/Server架构就是通过Android的AIDL接口来和单独的Unity服务端进程通信的。

#### 进程隔离方案-UAAL(Render As Library)

基于UAAL（Render As Library），支持把渲染服务嵌入原生安卓APP。

* Tuanjie 引擎可作为 Render Service，嵌入原生 Android APP，为原生 Android APP 提供 3D 内容
* 支持多个 view，支持非全屏渲染，每个 APP 仅需集成 View 组件，脱离 Activity
* 支持加载多个 Tuanjie 实例

Tuanjie Editor 打包出的 Android Studio 工程或 APK 包括 Client 和 Service 两部分。

UAAL 方案的优势在于，Unity 渲染服务和原生 APP 之间的通信链路是独立的，原生 APP 可以通过 JNI 接口和 Unity 渲染服务进行通信，而 Unity 渲染服务也可以通过 JNI 接口和原生 APP 进行通信。

这种方式集成的话，Unity会将渲染引擎，资源文件，和Android上层的通信代码都打包导出到一个aar中，其体积随动效的复杂程度而变化，同时会使集成方的apk包体积增加。而且项目里有多少方要使用Unity动效，就需要多少份的渲染引擎。这个方案由客户端来负责Unity控件的创建销毁，显示隐藏，一般适用一对一，通信链路简单的，即项目中可能只有一个模块需要使用Unity动效的情况。在多模块需要使用Unity的情况下，进程隔离的方案对性能的占用也比较高。

上层使用到的控件——UnityPlayer，它是一个Unity自定义的FrameLayout，里面有他们自己实现的一系列添加view，显示，和渲染逻辑。资源文件均存在于Unity打的依赖包中，对外不开放。

#### 集成步骤
第一步，将Unity提供的aar放置于libs文件夹中，并在gradle里添加其编译引用。

```groovy
implementation files('libs/UnityAnimation_0321V4.aar')
```
第二步，gradle中配置Unity所需的NDK版本，配置abifilters，设置要将哪些架构的动态库打包到apk中，对于车机项目来说只需要固定的某一种架构即可。还有设置不压缩的文件类型，使Unity可以顺利找到资源使用。

```groovy
ndkVersion "23.1.7779620"

aaptOptions {
        noCompress = ['.tj3d', '.ress', '.resource', '.obb', '.bundle', '.tuanjieexp', 'global-metadata.so'] + tuanjieStreamingAssets.tokenize(', ')
        ignoreAssetsPattern = "!.svn:!.git:!.ds_store:!*.scc:!CVS:!thumbs.db:!picasa.ini:!*~"
    }

ndk{
    abiFilters 'arm64-v8a'
}
```
注意，我们还需要在项目的string.xml资源文件中添加Unity所需的一条String资源，否则Unity侧会空指针。

```xml
 <string name="game_view_content_description">Game view</string>
```

第三步，将要显示Unity动效的页面Activity改为继承自UnityPlayerActivity，Unity的核心显示控件，UnityPlayer，它的创建销毁，显示隐藏，由这个UnityPlayerActivity来统一管理，项目中集成这个Activity的子类再将mUnityPlayer通过addView添加到自己的根布局ViewGroup中当背景即可，而且可以在xml上面继续增加其他View控件。
第四步，封装Unity通信工具类，Android给Unity发消息可以直接通过UnityPlayer的sendMessage静态方法，传入Unity通信协议中指定的类名。

```kotlin
 UnityPlayer.UnitySendMessage(OBJ_NAME, METHOD_NAME, communicateMessage)
```
Unity使用C#开发，其给Android上层发消息则是通过反射回调信号类里的方法实现的，所以我们最好将信号管理类做成单例的，并给其Unity留下一个方法或者成员，可以拿到我们类的实例，顺利反射回调。我这里使用的是一个Kotlin类声明，并对外暴露一个公开的unityInstance成员。而这个方法onReceiveMsgFromUnity，即是Unity的反射调用，我们在其中进行信号的解析，并传到View中去，注意这个方法不是在主线程中反射的，所以后面需要优化一波。

```kotlin
object UnityMessageHelper {

    val unityInstance = this

    // Unity给Android的消息回调
    fun onReceiveMsgFromUnity(msg: String) {
        LogUtils.d(TAG, "onReceiveMsgFromUnity: $msg")
        if (listenerList.size > 0) {
            listenerList.forEach {
                it.onReceiveUnityMessage(msg)
            }
        }
    }
}
```
#### 信号类UnityMessageHelper的优化
由于我们的目标工程是空调app，在用户调节风向时的回调频率相当高，而自动扫风模式下，底层上传的数据频率也相当高，所以不适合到主线程中操作这么多的数据，我们用协程，配合Default调度器来处理这种CPU密集型的任务。两条链路，用户手指的拖动操作时，Unity反射回调的线程本身都是工作线程了，所以我们在使用自定义的接口回调到View类的时候，使用MainScope.launch包一层，确保是到主线程更新我们的UI。而自动扫风模式从域控制器接收到风口点击的坐标值时，我们拿到数据后给Unity下发信号，更新动效的指向位置。可以使用协程上下文切换，withContext(Dispatcher.Default)将其切到工作线程里发送给Unity。

#### 遇到的问题
Unity方给的aar里的基类Activity适用与绝大多数的普通应用，但是我这里空调app的定位是一个高层级的悬浮窗，我的工程里压根就没有Activity。

这个时候我们用不上他们定义的UnityPlayerActivity，只能使用原生Raw的UnityPlayer，自己管理其创建，销毁，resume和pause。这里需要注意的是，UnityPlayer的创建需要传一个Context上下文，而应用里又没有Activity类型的Context，故只能使用非Activity类型的Context，在实践中发现，这个UnityPlayer的实例必须是我们的应用拿到可用的窗口句柄之后，才能被成功创建，否则就会报错。

所以正确的创建与初始化顺序是先使用WindowManager添加一个xml布局inflate来的ViewGroup，在其onAttachToWindow的方法回调之后，再创建UnityPlayer的实例，并添加到这个ViewGroup的布局中去，调用其resume方法。

```kotlin
    public void initUnity() {
        if (mUnityPlayer == null) {
            LogUtils.i(TAG, "initUnity");
            unityInitView = (LinearLayout) LayoutInflater.from(mContext).inflate(R.layout.layout_unity_init, null);
            unityInitView.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
                @Override
                public void onViewAttachedToWindow(@NonNull View v) {
                    LogUtils.w(TAG, "unityInitView onViewAttachedToWindow");
                    mUnityPlayer = new UnityPlayer(mContext);
                    unityInitView.addView(mUnityPlayer);
                    mUnityPlayer.requestFocus();
                    mUnityPlayer.resume();
                    mUnityPlayer.windowFocusChanged(true);
                }

                @Override
                public void onViewDetachedFromWindow(@NonNull View v) {
                    LogUtils.w(TAG, "unityInitView onViewDetachedFromWindow");
                    // Unityplayer已经成功移除，通知airView将player添加进去
                    airConditionerView.addUnityToAirView();
                }
            });

           mWindowManager.addView(unityInitView, initUnityWindowParams());
        }
    }
```
注意，这样添加的UnityPlayer有一个无法解决的黑屏问题，因为Unity的渲染加载至少都需要4，5秒，期间我们只能在更上层的View里设置静态背景图覆盖上去，等Unity加载完毕，发送ready的回调之后，我们移除掉这个占位的静态图，展示Unity动效的界面。这也是进程隔离的方案的一个很棘手的问题。我的解决方案是在开机的时候往屏幕外添加一个View专门来初始化加载Unity，加载完毕后，再将UnityPlayer给从里面remove掉，重新添加到实际的要展示的窗口中去，这样打开界面的时候可以略去加载的耗时，稍微减少页面僵直的时间。

#### 单进程-URAS(Render As Service)

一个渲染服务 Service 支持多个 Client APP 运行，且每个 Client APP 相互独立、互不干扰、可自更新。

* 仅需一个 Service，多个工程共用同一个 Service，每个工程均正常运行且互不干扰
* 一个新 Client 集成到已运行 Service 中，新 Client 可正常运行和渲染，已运行的 Client 和 Service 均不受干扰。
* 已运行的 Service 和 Client 中，关闭一个 Client，不影响其他 Client 和 Service 的正常运行
* 每个 Client 可通过 OTA 单独更新，更新后可正常运行，且不影响其他 Client 和 Service

Unity Rendering as Service（简称URAS） 的渲染方案是团结引擎特有的，无需在多个安卓应用中集成多个Unity 3D player，而是后台运行，前端应用可直接调用，节省系统资源，更适合多应用动效一镜到底的设计。

#### 相较进程隔离方案的优势
这个方案是在UAAL方案的基础上升级的，所以有一些前期工作是重复的，不作重复的阐述。

它是将要显示的几个Unity引擎都打包到同一个Server服务端去统一管控。其实服务端的apk打包也是拿到Unity提供的服务端AAR打进一个空工程，内部逻辑也隐藏到了AAR中。服务端和客户端的通信采用我们熟知的AIDL接口来实现。而且这个服务端我们需要设置为persistent应用，使其能开机自启，自动执行渲染等工作，其他应用有显示需求可以秒开，并且长时间不显示也不会自己回收资源了，客户端的黑屏问题也可以解决了。

相比于UAAL方案，客户端需要集成的是一个体积很小的Client.aar，对于客户端apk的体积控制是有优势的。

#### 集成与使用方式
我们只需要在gradle里引入这个客户端aar。在gradle sync之后，将远程的UnityView添加到自己的布局中去，配置好display参数(用来给服务端区分是哪个引擎的内容)，并指定服务端的包名。承载的View类型有SurfaceView和TextureView两种，而我的应用界面因为是一个悬浮窗口，设计有进出场的渐隐渐出动效，而SurfaceView不可以线性地设置alpha动画，所以选取TextureView来当作容器。

```xml
<com.unity3d.renderservice.client.TuanjieView
        android:id="@+id/unityview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:tuanjieDisplay="2"
        app:tuanjieServicePkgName="com.tuanjie.renderservice"
        app:tuanjieViewType="TextureView" />
```
剩余的代码逻辑仅仅是服务端Service的启动，添加服务连接的回调，消息回调。由于服务端为若干个Client的公共引擎，所以连resume和pause都不需要处理，因为这两个操作会对所有的客户端都生效。我们只需要确保启动服务，并使用正确的display即可，面板退到后台可以使用setVisbility来控制其显示隐藏。除此之外，我们的通信工具类，UnityMessageHelper还需要实现两个接口，一个服务连接状态接口，一个业务数据的消息回调接口，代码如下：

```kotlin
object UnityMessageHelper : TuanjieRenderService.Callback, SendMessageCallback {

    fun initUnityService() {
        LogUtils.i(TAG, "initUnityService")
        unityRenderService = 
            TuanjieRenderService.init(appContext,TUANJIE_PACKAGENAME).apply {
                enableAutoReconnect = true
                addCallback(this@UnityMessageHelper)
                addSendMessageCallback(this@UnityMessageHelper)
                ensureStarted()
        }
    }


    override fun onServiceConnected() {
        LogUtils.w(TAG, "onUnityRenderServiceConnected")
    }
   
    override fun onServiceDisconnected() {
        LogUtils.w(TAG, "onUnityRenderServiceDisConnected")
        messageScope.launch {
            delay(500L)
            initUnityService()
            unityRenderService.resume()
        }
    }
   
    override fun onServiceStartRenderView(p0: Int) {
        LogUtils.i(TAG, "onServiceStartRenderView")
    }

    override fun onClientRecvMessage(message: String?) = null

    // 服务端的消息回调
    override fun onClientRecvMessageWithNoRet(msg: String?) {
        // 回调消息的解析

    }

}
```
可以说URAS方案由于其统一管控，一对多的特点，在性能和客户端的易集成性方面，是优于UAAL方案的。可以从架构层面上，联动更多的动效使用模块，实现一镜到底的丝滑转场。