---
title: Android消息机制
categories:
  - Android进阶
tags:
  - Looper
  - Handler
  - Message
date: 2016-12-11 14:17:21
---

本文介绍Android的消息机制。

<!-- more -->

# 1 引言

假设现在我们有这样的需求，点一下图中的button，然后去获取一些数据（假设这个步骤是一个耗时的操作），然后获取完后将得到的数据返回显示到屏幕上。
为了避免产生`ANR(Application Not Response)`问题，通常我们会在新的线程去做耗时的操作，然后在UI线程里面更新组件，所以Handler就是类似这样子一个机制。

![android-looper-handler-message-01.png](/img/archives/android-looper-handler-message-01.png)

那么我们会怎么去实现呢？可以参考如下：

```java
public class TestDriverActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        tv = (TextView)findViewById(R.id.tv);
        btn = (Button)findViewById(R.id.btn);

        // 接收并处理消息
        mHandler = new Handler(){
            @Override
            public void handleMessage(Message msg){
                if (message.what == 0x11){
                    Toast.makeText(getApplicationContext(), "mHandler handleMessage" );
                    tv.setText("mHandler is receive message")
                }
                
            }
        };

        // 监听
        btn.setOnClickListener(new View.onClickListener(){
            @Override
            public void onClick(View v){
                MyThread t = new MyThread(mHandler);
                t.start();
            }
        });
    }
}


class MyThread extends Thread{
    Handler handler;
    public MyThread(Handler handler) {
        super();
        this.handler = handler;
    }
    @Override
    public void run(){
        super.run();
        mHandler.sendEmptyMessage(0x11);  //发送消息
    }
}

```
这只是一种实现的方式，如果在`子线程`而不是ui线程去初始化handler，则需要初始化`handler`前调用`Looper.prepare()`，初始化结束后调用`Looper.loop()`。

# 2 相关概念

学习Android的消息处理机制，有几个概念（类）必须了解：

1. **Message**
消息，理解为线程间通讯的数据单元。例如后台线程在处理数据完毕后需要更新UI，则可发送一条包含更新信息的Message给UI线程。
2. **Message Queue**
消息队列，用来存放通过Handler发布的消息，按照先进先出执行。
3. **Handler**
Handler是Message的主要处理者，负责将Message添加到消息队列以及对消息队列中的Message进行处理。
4. **Looper**
循环器，扮演Message Queue和Handler之间桥梁的角色，循环取出Message Queue里面的Message，并交付给相应的Handler进行处理。
5. **Thread**
UI thread 通常就是main thread，而Android启动程序时会替它建立一个Message Queue。
每一个线程里可含有一个Looper对象以及一个MessageQueue数据结构。在你的应用程序里，可以定义Handler的子类别来接收Looper所送出的消息。


# 3 Looper

Looper被设计用来使一个普通线程变成Looper线程。所谓Looper线程就是循环工作的线程。在程序开发中（尤其是GUI开发中），我们经常会需要一个线程不断循环，一旦有新任务则执行，执行完继续等待下一个任务，这就是Looper线程。使用Looper类创建Looper线程很简单：

```java
public class LooperThread extends Thread {
    @Override
    public void run() {
        // 将当前线程初始化为Looper线程
        Looper.prepare();
        
        // ...其他处理，如实例化handler
        
        // 开始循环处理消息队列
        Looper.loop();
    }
} 
```

## 3.1 Looper.prepare()

当执行了`Looper.prepare()`后，当前线程就会升级为Looper线程：

![android-looper-prepare.png](/img/archives/android-looper-prepare.png)

- 一个Thread只能有一个Looper对象
- 线程中有一个Looper对象，它的内部维护了一个消息队列MessageQueue

## 3.2 Looper.loop()

当调用loop方法后，Looper线程就开始真正工作了，它不断从自己的MQ中取出队头的消息(也叫任务)执行。

![android-looper-loop.png](/img/archives/android-looper-loop.png)

那么，我们如何往MQ上添加消息呢？下面有请Handler

# 4 Handler

Handler扮演了往MQ上添加消息和处理消息的角色（只处理由自己发出的消息），即往MQ上添加消息的时候执行sendMessage，并在loop到自己的时候处理消息执行handleMessage，整个过程是异步的。

**Handler创建时会关联一个looper，默认关联当前线程的looper。**

```java
public class LooperThread extends Thread {
    private Handler handler1;
    private Handler handler2;
                                                              
    @Override
    public void run() {
        // 将当前线程初始化为Looper线程
        Looper.prepare();
        
        // 实例化两个handler
        handler1 = new Handler(); 
        handler2 = new Handler();
        
        // 开始循环处理消息队列
        Looper.loop();
    }
} 
```    

加入Handler后结构图如下：

![android-looper-handler-2.png](/img/archives/android-looper-handler-2.png)

**一个线程可以有多个Handler，但是只能有一个Looper。**

# 4.1 Handler发送消息和处理消息

大致流程：
1. mHandler.sendMessage()发送消息到MQ
2. Looper.loop()将message不断从MQ从取出来交给handler处理
3. mHandler.handleMessage()处理消息

![android-handler-send-handle-msg.PNG](/img/archives/android-handler-send-handle-msg.PNG)

# 5 回顾

那么回到一开始我们举的例子，在非UI线程去做耗时的操作，然后完成后在UI线程更新UI信息。那么这种case下，我们的结构图是这样的：

![android-ui-thread-handler.PNG](/img/archives/android-ui-thread-handler.PNG)


至此，本文介绍的内容已经完成，本文内容大部分非原创，更多的是基于其他博客的和自己理解的总结，好记性不如烂笔头。如果需要了解源码的同学，可以继续深入阅读研究，包括Java层Looper，Handler，Message，MessageQueue的源码和Native层Looper，NativeMessageQueue的源码实现。
