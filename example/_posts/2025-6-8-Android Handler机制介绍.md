---
layout: post
description: > 
  本文介绍了Android中特有的Handler消息处理机制
image: 
  path: /assets/img/blog/blogs_handler_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_handler_cover.png
    960w:  /assets/img/blog/blogs_handler_cover.png
    480w:  /assets/img/blog/blogs_handler_cover.png
accent_image: /assets/img/blog/blogs_handler_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# Android Handler机制介绍
Android的Handler机制是Android异步通信的核心机制之一，它主要用于解决在子线程中进行耗时操作后，需要更新UI的问题。由于Android的UI操作是非线程安全的，只能在主线程（UI线程）中进行，因此Handler机制提供了一种安全地从子线程向主线程发送消息并处理消息的方式。

### 核心组件

Handler机制主要由以下四个核心组件组成：

1.  **Message（消息）**:
    * Message是Handler机制中传递的数据单元，可以携带少量数据。
    * 它包含了消息的类型（`what`）、优先级（`arg1`、`arg2`）、对象数据（`obj`）以及一个Bundle数据（`data`）等信息。
    * Message可以由`Handler`发送到`MessageQueue`中。
    * 通常，我们通过`Message.obtain()`或`Handler.obtainMessage()`来获取一个Message对象，以避免频繁创建Message对象造成的性能损耗。

2.  **MessageQueue（消息队列）**:
    * MessageQueue是一个存储Message的队列，它采用**单链表**的数据结构来存储消息。
    * 它负责管理所有由Handler发送的消息，包括插入消息和按照一定的规则（通常是消息的发送时间）取出消息。
    * 每个Looper都对应一个MessageQueue，它会不断地从MessageQueue中取出消息。
    * MessageQueue是一个“先进先出”的队列，但是它也支持延迟消息的插入和取出。

3.  **Looper（消息循环器）**:
    * Looper是一个线程内部的循环器，它负责不断地从`MessageQueue`中取出消息，并将其分发给对应的`Handler`进行处理。
    * 每个线程最多只能有一个Looper。
    * **主线程（UI线程）默认拥有一个Looper**，系统会自动为它创建和启动。
    * 在子线程中，如果需要使用Handler机制，则必须手动为该线程创建并启动Looper，通常通过`Looper.prepare()`和`Looper.loop()`方法实现。
    * `Looper.loop()`方法是一个阻塞的循环，会一直从`MessageQueue`中取消息，直到`MessageQueue`中没有消息或者Looper被退出（通过`Looper.quit()`或`Looper.quitSafely()`）。

4.  **Handler（消息处理器）**:
    * Handler是Looper和MessageQueue的接口。
    * 它主要有两个作用：
        1.  **发送消息**: 可以通过`handler.sendMessage()`、`handler.post()`等方法将Message或Runnable发送到其关联的MessageQueue中。
        2.  **处理消息**: 当Looper从MessageQueue中取出消息后，会将消息回调给对应的Handler的`handleMessage()`方法进行处理。
    * Handler在哪个线程创建，默认就和哪个线程的Looper绑定。因此，如果Handler在主线程创建，那么它发送和处理的消息都会在主线程中执行。

### 工作原理

Handler机制的工作流程可以概括为以下步骤：

1.  **在主线程或子线程中创建Handler**:
    * 当Handler被创建时，它会自动关联当前线程的Looper（如果当前线程有Looper的话）。如果当前线程没有Looper，并且在子线程中直接创建Handler，会抛出异常。
    * 通常在主线程中创建一个Handler，这样这个Handler处理的消息都会在主线程中执行，方便更新UI。
    * 在子线程中，如果需要创建Handler，则需要先调用`Looper.prepare()`为当前线程准备一个Looper，然后创建Handler，最后调用`Looper.loop()`启动消息循环。

2.  **子线程发送消息**:
    * 当子线程需要向主线程发送消息时（例如，后台任务完成，需要更新UI），它可以通过主线程创建的Handler对象，调用`sendMessage()`或`post()`等方法，将一个Message或Runnable发送到Handler关联的MessageQueue中。

3.  **MessageQueue接收并存储消息**:
    * Handler发送的消息会被放入到MessageQueue中，按照其发送的时间（或延迟时间）进行排序。

4.  **Looper循环取出消息**:
    * Looper会不断地从MessageQueue中取出消息。
    * 如果MessageQueue中没有消息，Looper会阻塞等待。
    * 如果MessageQueue中有消息，Looper会取出最先到达（或最先到期）的消息。

5.  **Handler处理消息**:
    * Looper取出消息后，会将消息分发给对应的Handler的`dispatchMessage()`方法。
    * `dispatchMessage()`方法会根据Message的类型，最终调用`handleMessage()`方法或执行Post的Runnable。
    * 由于Handler是在主线程创建的，所以`handleMessage()`方法或执行的Runnable代码会在主线程中执行，从而可以安全地进行UI操作。

### 适用场景

Handler机制主要用于以下场景：

* **子线程更新UI**: 这是Handler最主要的用途。在子线程中执行耗时操作（如网络请求、数据库操作），完成后通过Handler将结果发送到主线程，由主线程更新UI。
* **线程间通信**: 不同的线程之间可以通过Handler发送消息进行通信。
* **延迟执行任务**: 使用`postDelayed()`或`sendMessageDelayed()`可以实现延迟执行某个任务。
* **定时执行任务**: 结合`postDelayed()`或`sendMessageDelayed()`，可以在`handleMessage()`中再次发送延迟消息，实现定时任务。

### 示例代码

**主线程创建Handler，子线程发送消息更新UI：**

```java
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    private TextView mTextView;
    private Handler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTextView = findViewById(R.id.my_text_view);

        // 1. 在主线程创建Handler，它会关联主线程的Looper
        mHandler = new Handler(Looper.getMainLooper()) {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                // 5. 在主线程中处理消息，更新UI
                if (msg.what == 1) {
                    String data = (String) msg.obj;
                    mTextView.setText("Received from sub-thread: " + data);
                }
            }
        };

        // 启动一个子线程执行耗时操作
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000); // 模拟耗时操作
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                // 2. 子线程创建Message并发送给主线程的Handler
                Message message = Message.obtain();
                message.what = 1; // 消息标识
                message.obj = "Hello from background thread!"; // 携带数据
                mHandler.sendMessage(message); // 将消息发送到主线程的MessageQueue

                // 也可以使用post方法发送Runnable
                // mHandler.post(new Runnable() {
                //     @Override
                //     public void run() {
                //         mTextView.setText("Updated via post from background thread!");
                //     }
                // });
            }
        }).start();
    }
}
```

**在子线程中创建Looper和Handler：**

```java
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.util.Log;
import androidx.appcompat.app.AppCompatActivity;

public class SubThreadHandlerActivity extends AppCompatActivity {

    private static final String TAG = "SubThreadHandlerActivity";
    private Handler subThreadHandler; // 子线程的Handler

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main); // 假设还是用这个布局

        // 启动一个新的线程，并在该线程中创建Looper和Handler
        new Thread(new Runnable() {
            @Override
            public void run() {
                // 1. 在子线程中准备Looper
                Looper.prepare();

                // 2. 在子线程中创建Handler，它会关联当前线程的Looper
                subThreadHandler = new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                        if (msg.what == 2) {
                            String data = (String) msg.obj;
                            Log.d(TAG, "SubThread received message: " + data + " on thread: " + Thread.currentThread().getName());
                            // 在这里可以进行一些非UI操作
                        }
                    }
                };

                // 3. 启动子线程的消息循环
                Looper.loop();
                Log.d(TAG, "SubThread Looper stopped."); // 当Looper.quit()被调用时会执行
            }
        }, "MyWorkerThread").start();

        // 延时发送消息到子线程的Handler
        new Handler(Looper.getMainLooper()).postDelayed(new Runnable() {
            @Override
            public void run() {
                if (subThreadHandler != null) {
                    Message message = Message.obtain();
                    message.what = 2;
                    message.obj = "Message from Main Thread to Sub Thread";
                    subThreadHandler.sendMessage(message);
                }
            }
        }, 5000); // 5秒后发送
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 在Activity销毁时，如果子线程的Looper还在运行，应该将其退出
        if (subThreadHandler != null && subThreadHandler.getLooper() != null) {
            subThreadHandler.getLooper().quitSafely();
        }
    }
}
```

### 注意事项

* **内存泄露**: 在匿名内部类或非静态内部类中创建Handler时，如果Handler持有Activity的引用，且消息队列中存在未处理的消息，可能导致Activity无法被垃圾回收，从而引起内存泄露。解决办法通常是使用`静态内部类 + WeakReference` 来持有Activity引用，或者在Activity的`onDestroy()`中移除所有未处理的消息。
* **Looper的退出**: 对于子线程中的Looper，在不再需要时应调用`Looper.quit()`或`Looper.quitSafely()`来终止消息循环，释放资源。
* **UI线程阻塞**: 避免在主线程的`handleMessage()`方法中执行耗时操作，否则会导致UI卡顿，影响用户体验。
* **HandlerThread**: Android提供了`HandlerThread`类，它是一个带有Looper的Thread，简化了在子线程中使用Handler的步骤。

通过理解Handler机制的各个组件及其工作流程，开发者可以更好地在Android应用中实现异步操作和线程间通信，保证UI的流畅性。