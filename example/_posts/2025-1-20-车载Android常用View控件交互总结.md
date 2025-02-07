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
## 前言


## 不同UI控件信号处理规范
### Switch开关类
开关一般指各个设置项，比如氛围灯，能量回收，智驾功能开关等。Switch都有下发指令和显示状态的需求。下发指令只能通过用户手动点击触发，不可自行发送信号；状态显示有主动获取与被动接收通知之分，主动获取常见的策略是在点击之后若干秒后，去获取当前的功能信号状态，进而刷新开关状态。被动接收通知为长期监听，底层因错误发出置灰信号，或者功能在某种条件下自动打开或关闭，都需要及时地反馈到开关上。注意这种主动和被动的更新常常是冲突的，处理不好会导致开关快速闪动。
变种类switch：具有两种非此即彼状态的控件，比如一个高亮色块，通过不同颜色图标来表示开关此时的状态，往往还需要更改开关的文字描述，例：
链路耗时长
若8155下游控制器可以做到快速切换，只是信号链路传输慢，可以在用户每次点击开关都直接往下发设置信号，快速点击期间都removeCallbacks，不进行状态的主动获取，也移除开关的被动监听刷新。在用户手指最后一次点击若干秒后，主动获取一次开关状态，这种情况下，开关状态一般和最后一次点击下发的值是同步的，不会有回弹现象。并且在主动获取之后，重新加上开关的被动监听刷新机制。即可以支持快速接收并执行指令的，上层芯片可以只管发，处理好自己的UI就行。
执行时间长
如果该功能是控制器执行速度慢，可以再快速点击期间，只变动Switch的UI，不往下发值，在最后一次点击的若干秒后，再往下游控制器发送信号，并且延时获取状态，期间被动的监听也是移除的，在主动获取状态刷新UI之后，再加上状态监听。例如下面是500ms防抖的设置方法，两次点击间隔500ms以内，只有快速点击完成后500ms，才会继续执行逻辑。即不可快速自由切换的功能的，上层芯片需要过滤点击事件，尽量将一次抖动流程里只发最后一帧信号。
mHandler.removeCallbacks(signalRunnable);
mHandler.postDelayed(signalRunnable, 500L);以上两种是用户体验较好的方案，可以支持快速点击，只取最后一次点击使其生效。
加入点击限制
除了防抖，还可以使用点击限制，在点击后的若干时间内，直接使开关除能，不再接收点击事件，在此期间加入置灰或者加载的样式提示此开关不可用。这种纯粹的点击限制，在用户体验上不是特别好，最好加入说明文案等。适合较复杂功能，底层ECU执行时间特别长（2s以上），并且多次频繁下发值有可能导致其出错的场景。

```java
  switch.setOnCheckedChangeListener { buttonView, isChecked ->
            switch.isEnabled = false
            mHandler.postDelayed({ switch.isEnabled = true }, 2000L)
            SignalUtil.sendSignal()
        }
```

### RadioButton类多选控件
这类控件往往是同一个功能，走同一个信号接口，有两个以上的待选项，可以选取不同参数的功能。比如下面这个灯光时长选择的控件。
这类控件的处理方式和上述switch类似。
1. 执行时间长的需要限制点击下发，某段时间内只允许一次点击；
2. 信号链路长的功能，可以不限制点击，只限制回调刷新UI。在快速点击过程中，信号是即点即发，在点击时知道点击后的若干秒内，UI控件不响应回调数据的变化，在抖动流程结束后，主动获取一次状态，并重新添加上回调监听更新UI的逻辑。
### 持续调节自定义View类
首先了解：
点击事件分发与消耗机制
当一个点击事件产生后,它的传递过程遵循如下顺序:Activity -> Window -> View,即事件总是先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View。顶级View接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个View的onTouchEvent返回false，那么它的父容器的onTouchEvent将会被调用，依此类推。如果所有的元素都不处理这个事件，那么这个事件将会最终传递给Activity处理，即Activity的onTouchEvent方法会被调用。

这种长按持续调节的交互方式，需要实现控件的ontouch方法，并监听手势滑动轨迹，在回调方法里实时更新自己的UI，并且持续性地发送信号。
插入，ontouch方法和onTouchEvent方法：
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

```java
mCushionTouch.setOnTouchListener((view, motionEvent) -> {
            if (motionEvent.getAction() == MotionEvent.ACTION_DOWN) {
            
            }
            if (motionEvent.getAction() == MotionEvent.ACTION_MOVE) {
             // DO YOUR WORK
             // UPDATE UI  &  SEND SIGNAL
            return true;
            }
);
```

这种控件，一般用在座椅位置和空调风向的调节上，需要其既能响应用户手动滑动，来更新界面UI样式，又可以根据底层反馈更新。
在手动调节时，同样的，为了避免界面显示错乱，需要在touch调节时移除回调更新UI的逻辑，以用户手动调节的位置为最高优先级，调节完成后若干时间后，获取状态更新UI，再重新添加上回调去监听更新的逻辑。注意界面首次调节的起始点必须是当前的位置，不可出现跳动现象。
### 滑动条SeekBar类
这一类控件常被用来作为进度条展示，也具有手动调节的功能。一般是亮度，音量，充放电电量等具有一定调节范围的设置项。它有三个回调方法，分别是onProgressChanged，onStartTrackingTouch，  onStopTrackingTouch，代表调节时，按下时，抬起时。其中onProgressChanged的回调相当之快，除非有动态变化显示的需求，否则不要在这里处理逻辑，或者在这里的逻辑加上防抖限制，一定时间内只调用一次。曾经在这里调用埋点方法，利用系统服务处理网络上传请求，导致系统崩溃黑屏。后续改到停止调节时上传，一次touch操作只会传一次。
这类调节条的更新逻辑与其他控件类似，在下发方案上主要分两类：
一、 需求上实时调节的，比如氛围灯颜色，音量，亮度，在跟手时硬件即响应变化，用户体验比较好，这种需要在onProgressChanged回调里进行信号的发送。
二、不需要实时调节的，是那种无法直观观察到变化的设置项，比如车辆充放电截止电量，能动回收百分比，各种灵敏度等，这就推荐在手指抬起或者按下时调用一次，不处理变化中的逻辑。
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
所以为了丝滑调节，可以设置默认步长1，采用整除回乘的算法来将区间数据处理成需要的数据，比如32整除5为6，再乘5就是30。
至于快速调节过程中可能出现的回调闪烁问题，则采用防抖或者节流算法来减少频次，再控制一下回调刷新UI的逻辑，即可实现体验最优的seekbar滑动调节信号收发。


