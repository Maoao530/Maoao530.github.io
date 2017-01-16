---
title: Android四大组件
categories:
  - Android基础
tags:
  - Android
  - Activity
  - Service
  - BroadcastReceiver
  - ContentProvider
date: 2016-08-11 09:59:38
---

复习Android四大组件内容。

<!-- more -->

本文会总结介绍Android组件中最为常见用的四大组件：Activity，Service服务,ContentProvider内容提供者，BroadcastReceiver广播接收者。

# 1 Activity

在一个android应用中，一个Activity通常就是一个单独的屏幕，它上面可以显示一些控件也可以监听并处理用户的事件做出响应。Activity之间通过Intent进行通信。

## 1.1 主要函数

![主要函数](/img/archives/activity-main.png)

## 1.2 生命周期

![生命周期](/img/archives/activity-shengming-zhouqi.png)

## 1.3 AndroidManifest.xml文件配置

需要在apk的AndroidManifest配置文件中进行配置：

![配置](/img/archives/activity-androidmanifest.png)

## 1.4 Activity管理

activity在android里面是以栈的形式管理的，处于前台的 Activity 总是在栈的顶端，当前台的 Activity 因为异常或其它原因被销毁时，处于栈第二层的 Activity 将被激活，上浮到栈顶。当新的 Activity 启动入栈时，原 Activity 会被压入到栈的第二层。如下：

![activity管理](/img/archives/activity-stack.png)

## 1.5 通信
1. activity之间通过Intent进行通信，可以将数据放入Bundle中，再将Bundle放入intent中，实现数据通信。
2. 可以通过intent去启动一个activity，方式有显示Intent和隐式Intent。
2.1 显示
   直接指明启动的Activity类：Context.startActivity(new Intent(this,xxx.class))
2.2 隐式
   需要在Activity对应的AndroidManifest.xml中配置对应的Intent-Filter中的action和category，一般默认category属性需要有一个default属性(如果有其它category则不用添加此属性)：`<category android:name=”android.Intent.Category.DEFAULT” />`
这是因为Android把所有传给startActivity()的隐式意图当作他们包含至少一个类别`"android.intent.category.DEFAULT"` 

![intent通信](/img/archives/activity-intent.png)

# 2 Service

Service作为Android四大组件之一，在每一个应用程序中都扮演着非常重要的角色。它主要用于在后台处理一些耗时的逻辑，或者去执行某些需要长期运行的任务。必要的时候我们甚至可以在程序退出的情况下，让Service在后台继续保持运行状态。

## 2.1 生命周期

Service生命周期函数比较简单：

![service生命周期](/img/archives/service-shengming-zhouqi.png)

## 2.2 Local/Remote Service

1. service分为`local service` 和 `remote service`，本地服务的生命周期是和主进程相关的，主进程结束的时候service也结束了，远程服务则为独立的进程和主进程没有关系。

2. local service 使用bindService启动service，用unBindService关闭service

3. remote service 使用startService和stopService启动和关闭Service

4. **通信：**
4.1 Local service ：是运行在主进程的main线程下的，在同一个进程，通信的方式通过返回一个IBinder，然后比如说可以在ui线程中去接收ibinder进行通信，可以吧。
4.2 Remote service：因为是独立进程，所以如果需要和其它进程进行通信，则需要通过aidl进行ipc通信

5. 如果需要比较耗时的操作，可以在service中开一个新的线程进行操作，避免阻塞ui线程产生anr，或者使用IntentService，因为IntentService会开启单独的线程来处理所有的Intent请求

关于Service其他本文不做过多说明，都是一些比较常用的知识。

# 3 BroadcastReceiver

BroadcastReceiver即广播接受者，是一种全局的监听器，可以用来作为不同组件之间的通信，比如说activity和service之间的通信可以借助其实现。

## 3.1 启动方式

Context.sendBroadcast() 或者Context.sendOrderedBroadcast()

## 3.2 接收方式

重写BroadcastReceiver的onReceive()方法

## 3.3 实现方式
BoradcastReceiver实现方式有两种，一是通过代码注册，二是通过Androidmanifest.xml方式配置：

方式一，代码：

```java
IntentFilter   filter  =   new IntentFilter(“xxx.xxx.xxx.xxxAction”);
XxxReceiver   receiver  =  XxxReceiver();
Context.registerReceiver( receiver,filter);
```

方式二，Androidmanifest.xml
```java
< receiver android:name = ".XxxReceiver" >
    < intent-filter  android:priority=”1000”>         
        < action android:name = " xxx.xxx.xxx.xxxAction" />
    </ intent-filter >
</ receiver >
```

每次广播事件发生后，系统会创建对应的BroadcastReceiver实例，并且去触发onReceive方法，这个方法执行完毕后，BroadcastReceiver实例会被销毁，因为BroadcastReceiver生命周期短，超过10s会产生anr对话框，所以不要在onReceive里面做一些耗时的操作，如果需要耗时操作，可以用Intent开启一个Service来完成而不是开启一个新的子线程去完成，因为可能子线程还没结束，BroadcastReceiver就已经结束退出了。

## 3.4 普通广播和有序广播

1. 普通广播：
普通广播是完全异步的，即发送一个广播，所有注册了action相同的BroadcastReceiver都能同时接收到，传递的效率比较高。

2. 有序广播：
顾名思义，需要设置广播的顺序，设置的方式有两种，一种是在代码中：IntentFilter.setPriority(Int  order)设置，一种是在AndroidManifest.xml中设置< intent-filter  android:priority=”1000”>，order值范围为[-1000, 1000]，1000为最高优先级，也就是说当你发送了一个广播，优先级高的BroadcastReceiver会先接收，然后传递给下一个优先级低的BroadcastReceiver，层层传递，然后还可以在优先级高的终止广播的往下传递，或者可以向下一个广播传递新的数据信息，具体方式如下
优先级高的广播发送信息：
```java
Bundle bundle = new Bundle();
bundle.putString(“info”,”优先级高的就是爽！！”);
setResultExtras(bundle);
// 甚至可以取消广播继续传递
// abortBroadcast()
优先级低的接收信息：
Bundle bundle = getResultExtras(true);
String info = bundle.getString(“info”);
```

# 4 ContentProvider

ContentProvider即内容提供者，简单来讲它的作用就是将一个app的数据提供给其它app进行操作，比如增删改查等。
一个典型的例子，比如说我们会经常遇到有应用软件需要读取手机的联系人，这时候就需要联系人应用的ContentProvider功能提供数据的crud操作。
ContentProvider的功能是提供数据的增删改查，而其它应用想访问ContentProvider的crud操作，则需要通过ContentResolver的增删改查功能实现，通过uri作为媒介，比如其它应用想去读取联系人，则模型如下：

![ContentProvider-Model](/img/archives/contentprovider-model.png)

## 4.1 URI

uri由三部分组成，协议+主机名+路径：

- **scheme：** ContentProvider（内容提供者）的scheme已经由Android所规定为：`content://`。  
- **主机名（或Authority）：** 用于唯一标识这个ContentProvider，外部调用者可以根据这个标识来找到它。
- **路径（path）：**可以用来表示我们要操作的数据，路径的构建应根据业务而定，如下：要操作contact表中id为10的记录，可以构建这样的路径
: `/contact/10`

一个uri例子：
```
content://com.shun.provider.myapp/contact/2
```

如果要把一个字符串转换成Uri，可以使用Uri类中的parse()方法，如下：

```java
Uri uri = Uri.parse("content://com.changcheng.provider.contactprovider/contact")
```

## 4.2 实现ContentProvider

实现一个ContentProvider有两个步骤：

1. 开发一个ContentProvider的子类，然后去实现它的`query`，`delete`，`update`，`insert`方法，然后还要使用`UriMatcher`对uri进行匹配过滤
2. 在androidmanifest.xml中配置它，需要指定android:authorities方法（唯一主机名）
```
 <provider android:name="MyProvider" android:authorities=" com.shun.provider.myapp " /> 
```
## 4.3 ContentResolver

ContentResolver的使用则比较简单了：
```java
Context.getContentResolver().query(…)
Context.getContentResolver().delete(…)
Context.getContentResolver().update(…)
Context.getContentResolver().insert(…)
```
当然，上面这些个crud方法返回值肯定不一样，具体参考google官网

## 4.4 ContentObserver

ContentObserver即内容观察者，可以监听ContentProvider数据的改变，实现一个内容观察者步骤如下：

1. 实现一个ContentObserver的子类，实现onChange()方法，每当有数据改变时会触发此方法
2. 注册监听器，比如监听短信信息：
```java
Uri smsUri = Uri.parse("content://sms");  
getContentResolver().registerContentObserver(smsUri, true, new smsContentObserver()); 
```
