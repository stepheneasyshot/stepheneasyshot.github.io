---
layout: post
description: > 
  本文介绍了车载Android里的常用的View控件交互总结
image: 
  path: /assets/img/blog/blogs_aaos_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_aaos_cover.png
    960w:  /assets/img/blog/blogs_aaos_cover.png
    480w:  /assets/img/blog/blogs_aaos_cover.png
accent_image: /assets/img/blog/blogs_aaos_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 车载Android常用View控件交互总结
## 模块名词解释
### 一种常见架构

![blogs_view_car_net](/assets/img/blog/blogs_view_car_net.png)

### VIU
汽车电子系统中的一个重要组成部分，它是一种称为“车辆信息单元”的设备，也被称为“车辆智能单元”，是车辆智能化的核心部件之一。VIU是英文Vehicle Information Unit的缩写，其主要功能是收集车辆的各种信息并进行处理。

VIU由众多不同的传感器、执行器和微控制器组成，它们单独或联合工作，从不同的方面监测和分析车辆的各项数据。传感器可以感知发动机状态、温度、湿度、油压、燃油消耗量等各类参数，执行器则能够控制发动机、座椅、车门等车辆各部分，微控制器则负责控制各种数据从传输到处理。
### S32G
在域控制器中，网关处理器的作用不容忽视，其作为域控制器的中心枢纽，负责安全和功能域（如动力传动、底盘与安全、车身控制、信息娱乐和ADAS等）之间互联并处理异构车载网络中的数据。S32G采用M核+A核多核异构架构，兼顾实时应用以及高算力应用场景，并且具有ASIL D的功能安全等级，集成了低延迟通信引擎LLCE，数据包转发引擎PFE，硬件安全引擎HSE等独立内核，非常适合作为主控制器，整合传统网关，BCM，VCU等多个ECU控制逻辑。

S32G M核基于AUTOSAR CP, 可以处理CAN/LIN信号,系统启动，电源管理，健康管理，车控等对实时性要求较高的应用，以及多种安全策略和功能处理策略。
S32G A核运行Linux或者QNX，满足对于处理多路千兆以太网、大数据收集与分析、整车OTA、数据存储、远程诊断等功能，可部署多种通信协议和相关学习算法。M核和A核之间既可以采用核间通信IPCF，交换延迟要求非常低的数据，也可以通过PFE共享以太网接口，实现高吞吐量数据交换需求。

其内部有如下模块：

1. 整车域控制组件:集成网关、车窗控制、灯光控制等功能模块;
1. 动力域控制组件:集成整车控制器、电池管理系统、热管理系统等功能模块;
1. 底盘域控制组件:集成智能控制悬挂、电子驻车单元等系统功能模块;
1. 车载中央计算机:可协调各个域控单元组件有序工作;
1. 更多应用场景可根据客户实际需求进行功能组件的组合。

可以说其为整车的核心控制器，身兼中央网关信号中转和重要逻辑处理等多个角色。
### TDA4
TDA4是德州仪器推出的一款高性能、超异构的多核SoC，拥有ARM Cortex-R5F、ARM Cortex-A72、C66以及C71内核，可以部署AUTOSAR CP系统、HLOS(Linux或QNX)、图像处理以及深度学习等功能模块，从而满足ADAS对实时性、高运算能力、环境感知及深度学习等方面的需求。

TDA4凭借着出色的运算能力、有竞争性的价格，赢得了越来越多汽车主机厂以及零部件供应商的青睐。

这款智能驾驶处理芯片，计算效率高，工具链成熟，但是算力低，行泊一体将导致其想要完善的能力所带来的要求也极高。
### 8155
相当于手机的高通855芯片，属于高端旗舰级别，事实上诞生于2019年的高通8155其实就是基于855手机芯片“魔改”而来的，当时高通855芯片在业内也算是领先水平，该芯片多应用在各品牌的旗舰级手机上，因此基于855打造的车规级8155性能也不会差。2019年所发布的8155芯片，至今除了特斯拉的AMD主机级锐龙芯片以外，仍然是天花板级别的存在，其稳定性、可靠性也能够经得起时间的考验，这也是为什么如今众多车企都选择高通骁龙8155。业内一般方案为quick unix系统上套安卓虚拟机的方案，以更稳定的qnx系统来作为硬件直接交互的角色，并且仪表显示等重要模块是运行在qnx系统上的应用，而Android系统由于其不稳定性，更适合作为系统和车控设置项、娱乐信息屏幕的承载系统。

## 不同电器架构上层共性
一个车控功能，链路从对应的底层控制器到座舱控制器的网络拓扑随着项目的电器架构而变化，一般分为两种情况：

* 分布式架构，各个控制器彼此独立，使用CAN总线进行通信；
* 集中式架构，一般有一个域控制器作为中心网络中转的角色，各个控制器都通过域控制器进行通信。

不管底层架构如何，当信号到了座舱域的8155或者8295控制器之后，信号的链路就是一致的了，网络层到硬件抽象层，再到系统FW的CarService层，最后通过Binder接口给到应用层。

### 信号的上下行流程
车控信号的上下行流程一般分为以下几个步骤：
1. 用户手动操作控件之后，应用下发对应的setter接口，调用request信号。
1. 座舱域的控制器将信号转发出去；
1. 目的控制器接收之后，做出对应处理，将操作的结果通过另一个广播信号或者setter的同一个信号返回上去
1. 座舱域的控制器将底层的反馈，回调给应用层。

而应用需要做的一般有下面几件事：
1. 界面初始化根据获取的初始值来刷新界面；
1. 点击控件可以下发信号；
1. 操作完毕之后，要根据反馈的信号来刷新页面；
1. 用户离手一段时间后（2s或者3s），需要主动获取一次开关的状态，刷新开关状态。即回弹逻辑，防止实际执行失败，却传达了执行成功的信息。

## 不同UI控件信号处理规范
### Switch开关类
开关一般用于各个设置项，比如氛围灯，智驾功能，蓝牙，网络开关等。

![blogs_view_switch](/assets/img/blog/blogs_view_switch.png)

Switch有切换时下发指令和显示开关状态的需求。

* 下发指令只能通过用户手动点击触发，不可自行发送信号；
* 状态显示有主动获取与被动接收通知之分，主动获取常见的策略是在点击之后若干秒后，去获取当前的功能信号状态，进而刷新开关状态。被动接收通知为长期监听。

底层因某些错误发出置灰信号，或者功能在某种条件下自动打开或关闭，都需要及时地反馈到界面的switch上。注意这种主动和被动的更新常常是冲突的，处理不好会导致开关快速闪动。

变种类switch，严格来说是Button，具有非此即彼，互斥状态的控件。比如可能设计成一个高亮色块，通过不同颜色图标来表示开关此时的状态，往往还需要更改开关的文字描述，例：

![blogs_view_button](/assets/img/blog/blogs_view_button.png)

#### 下游执行快
若座舱域的下游控制器可以做到快速切换，只是信号链路传输慢，可以在用户每次点击开关后，都直接往下发设置信号。单次点击一般没有问题，主要分析快速多次点击这种容易出问题的场景。 **快速点击期间都移除掉信号监听removeCallbacks，也不进行状态的主动获取刷新** 。

在快速点击时期结束，用户手指最后一次点击若干秒后，主动获取一次开关状态，这种情况下，开关状态一般和最后一次点击下发的值是同步的，不会有回弹现象。

并且在主动获取之后，重新加上开关的被动监听刷新机制。即可以支持快速接收并执行指令的，上层芯片可以只管发，处理好自己的UI就行。

#### 下游执行慢
如果该功能是控制器执行速度慢，可以在开关的快速点击期间，只跟随用户操作变动Switch的UI，不往下发setter信号，在最后一次点击结束后，过几百ms，再往下游控制器发送setter信号，快速点击期间，车控信号被动的监听也是移除的，在主动获取状态刷新UI之后，再加上状态监听。

例如下面是500ms防抖的设置方法，两次点击间隔500ms以内，会移除掉上一次的逻辑，只有快速点击完成后，过500ms，才会继续执行逻辑。

```java
  switch.setOnCheckedChangeListener { buttonView, isChecked ->
            mHandler.removeCallbacks(signalRunnable)
            mHandler.postDelayed(signalRunnable, 500L)
            SignalUtil.sendSignal()
        }
```

即不可快速自由切换的功能的，上层芯片需要过滤点击事件，尽量将一次抖动流程里只发最后一帧信号。

以上两种是用户体验较好的方案，可以支持快速点击，只取最后一次点击使其生效。

#### 加入点击限制
除了防抖，还可以使用点击限制，在点击后的若干时间内，直接使开关除能，不再接收点击事件，在此期间加入置灰或者加载loading的样式提示此开关暂时不可用。

这种纯粹的点击限制，在用户体验上不是特别好，最好加入说明文案等。适合比较复杂的功能，像底层ECU执行时间特别长（2s以上），并且多次频繁下发值有可能导致其出错的场景。

```java
val switchEnableRunnable = Runnable { switch.isEnabled = true }

switch.setOnCheckedChangeListener { buttonView, isChecked ->
    switch.isEnabled = false
    mHandler.removeCallbacks(switchEnableRunnable)
    mHandler.postDelayed(switchEnableRunnable, 2000L)
    SignalUtil.sendSignal()
}
```

### RadioGroup 类控件
这类控件组，往往是同一个功能，走同一个信号接口，有两个以上的待选项，可以选取不同参数的功能。比如 驾驶模式 选择的控件。

![blogs_view_radiogroup](/assets/img/blog/blogs_view_radiogroup.png)

这类控件的处理方式和上述 switch 开关类控件类似。

1. 执行时间长的需要限制点击下发，某段时间内只允许一次点击；
2. 信号链路长的功能，可以不限制点击，只限制回调刷新UI。在快速点击过程中，信号是即点即发，直到快速点击后的若干秒内，UI控件不响应回调数据的变化，在防抖动流程结束后，主动获取一次状态，并重新添加上回调监听更新UI的逻辑。
#### 防止刷新时的循环设置
RadioGroup加入 `checkchanged` 监听，可以监听开关项变化。但是由信号被动刷新时，也会触发这个回调，如果在这个里面设置的信号下发和埋点计算逻辑，就会重复计算。

甚至有时候时间差恰到好处的话，会导致两个开关项之间循环设置，不断跳动。

```java
mRgMainBlowFace.setOnCheckedChangeListener((group, checkedId) -> {
    // 发送set信号
    SignalUtil.sendSignal();
    // 埋点计算
    ReportUtil.report();
});
```

可以对RadioGroup进行封装，对OnCheckedChangeListener加入一个本地变量来保存，加入一个 `updateChecked` 方法替代原来的刷新方案，在这个 `update` 方法里，先把 `checkListener` 给移除掉，再改变选中项的状态，操作完毕再把回调加回去。

```java
public class RadioGroupEx extends RadioGroup {
    private RadioGroup.OnCheckedChangeListener mCheckedChangeListener;

    public RadioGroupEx(Context context) {
        super(context);
    }

    public RadioGroupEx(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public void setOnCheckedChangeListener(@Nullable RadioGroup.OnCheckedChangeListener listener) {
        this.mCheckedChangeListener = listener;
        super.setOnCheckedChangeListener(listener);
    }

    public void updateChecked(int checkedId) {
        super.setOnCheckedChangeListener((RadioGroup.OnCheckedChangeListener)null);
        this.check(checkedId);
        super.setOnCheckedChangeListener(this.mCheckedChangeListener);
    }
}
```

### 持续调节自定义View类
首先重温一下点击事件分发与消耗机制：

当一个点击事件产生后,它的传递过程遵循如下顺序: Activity -> Window -> View，即事件总是先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View。

顶级View接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个View的 `onTouchEvent` 返回false，那么它的父容器的onTouchEvent将会被调用，依此类推。

如果所有的元素都不处理这个事件，那么这个事件将会最终传递给Activity处理，即Activity的 `onTouchEvent` 方法会被调用。

这种长按持续调节的交互方式，需要手动实现控件的ontouch方法，并监听手势滑动轨迹，在 `Action MOVE` 回调方法里实时更新自己的UI，并且持续性地发送信号。

插入，ontouch方法和onTouchEvent方法：

```
boolean onTouch(View v, MotionVent event)
触摸事件发送到视图时调用（v:视图，event:触摸事件）
返回true:事件被完全消耗（即，从down事件开始，触发move，up所有的事件）
返回fasle:事件未被完全消耗（即，只会消耗掉down事件）

boolean onTouchEvent(MotionEvent event)
触摸屏幕时调用
返回值，同上

注意：
1、onTouch优先级比onTouchEvent高
2、如果button设置了onTouchListener监听，onTouch方法返回了true，就不会调用这个button的Click事件 
```

下面是一个复写 `OnTouchListener` 的例子：

```java
mCushionTouch.setOnTouchListener((view, motionEvent) -> {
    if (motionEvent.getAction() == MotionEvent.ACTION_DOWN) {

    }
    if (motionEvent.getAction() == MotionEvent.ACTION_MOVE) {
        // DO YOUR WORK
        // UPDATE UI  &  SEND SIGNAL
        return true;
    }
});
```

这种控件，在车控领域，一般用在座椅位置和空调风向的调节上，需要其既能响应用户手动滑动，来更新界面UI样式，又可以根据底层反馈更新。

在手动调节时，同样的，为了避免界面显示错乱，需要在touch调节时移除回调更新UI的逻辑，以用户手动调节的位置为最高优先级，调节完成后若干时间后，获取状态更新UI，再重新添加上回调去监听更新的逻辑。注意界面首次调节的起始点必须是当前的位置，不可出现跳动现象。
### 滑动条SeekBar类
这一类控件常被用来作为 **进度条** 展示，也具有 **手动调节** 的功能。

![blogs_view_seekbar](/assets/img/blog/blogs_view_seekbar.png)

一般是亮度，音量，充放电电量等具有一定调节范围的设置项。它有三个回调方法，分别是onProgressChanged，onStartTrackingTouch，onStopTrackingTouch，代表调节时，按下时，抬起时。

其中 `onProgressChanged` 的回调相当之快，除非有动态变化显示的需求，否则不建议在这里处理逻辑，或者在这里的逻辑加上防抖限制，一定时间内只调用一次。曾经我在这里调用埋点方法，利用系统服务处理网络上传请求，导致系统崩溃黑屏。后续改到停止调节时上传，一次touch操作只会传一次。

这类调节条的更新逻辑与其他控件类似，在下发方案上主要分两类：
1. 需求上实时调节的，比如氛围灯颜色，音量，亮度，在跟手时硬件即响应变化，用户体验比较好，这种需要在onProgressChanged回调里进行信号的发送。
2. 不需要实时调节的，是那种无法直观观察到变化的设置项，比如车辆充放电截止电量，能动回收百分比，各种灵敏度等，这就推荐在手指抬起或者按下时调用一次，不处理变化中的逻辑。

以上两种方案有一个共同的更新逻辑，就是在快速调节（滑动or快速点击）中，不响应底层数据反馈，避免进度条乱跳，在设置后一段时间内，主动获取状态，并重新加上数据监听被动更新。

```java
signalSeekbar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
    @Override
    public void onProgressChanged(SeekBar seekBar, int i, boolean b) {

    }

    @Override
    public void onStartTrackingTouch(SeekBar seekBar) {

    }

    @Override
    public void onStopTrackingTouch(SeekBar seekBar) {

    }
}); 
```

### 有步长的Seekbar体验优化
比如设置某个信号，底层只能接受像5，15，25等5的倍数的信号值。而将seekbar的步长设置为5，在滑动时会有明显的卡顿感。

![blogs_view_seekbar_step](/assets/img/blog/blogs_view_seekbar_step.png)

所以为了丝滑调节，可以设置默认步长1，采用整除回乘的算法来将区间数据处理成需要的数据，比如32整除5为6，再乘5就是30。

至于快速调节过程中可能出现的回调闪烁问题，则采用防抖或者节流算法来减少频次，再控制一下回调刷新UI的逻辑，即可实现体验最优的seekbar滑动调节信号收发。
