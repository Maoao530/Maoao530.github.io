---
title: 理解和使用systrace
categories:
  - Android进阶
tags:
  - systrace
date: 2017-02-06 11:25:51
---

理解和使用systrace。

<!-- more -->


# 一、介绍systrace

systrace是Android4.1版本之后推出的，对系统Performance分析的工具。

systrace的功能包括跟踪系统的I/O操作、内核工作队列、CPU负载以及Android各个子系统的运行状况等。在Android平台中，它主要由3部分组成：

- **内核部分：**Systrace利用了Linux Kernel中的`ftrace`功能。所以，如果要使用systrace的话，必须开启kernel中和ftrace相关的模块。
- **数据采集部分**：Android定义了一个Trace类。应用程序可利用该类把统计信息输出给ftrace。同时，Android还有一个`atrace`程序，它可以从ftrace中读取统计信息然后交给数据分析工具来处理。
- **数据分析工具**：Android提供一个`systrace.py`（python脚本文件，位于Android SDK目录`/sdk/platform-tools/systrace`中，其内部将调用atrace程序）用来配置数据采集的方式（如采集数据的标签、输出文件名等）和收集ftrace统计数据并生成一个结果网页文件供用户查看。

# 二、抓取systrace

有三种方式抓取systrace：

## 2.1 systrace.py工具

命令行的用法是：

```
python systrace.py [options] [category1] [category2] ... [categoryN]
```

需要装python，最好是2.7版本，避免出现问题，示例如下：

```
cd android-sdk/platform-tools/systrace
python systrace.py --time=10 -o mynewtrace.html sched gfx view wm
```

### 2.1.1 options

其中options可取值：

| options           | 描述           |
| -----------       | :------------  |
|-o  < FILE >  | 输出的目标文件|
|-t N, –time=N |  执行时间，默认5s|
|-b N, –buf-size=N  | buffer大小（单位kB),用于限制trace总大小，默认无上限|
|-k < KFUNCS >，–ktrace=< KFUNCS > |   追踪kernel函数，用逗号分隔|
|-a < APP_NAME >,–app=< APP_NAME >  | 追踪应用包名，用逗号分隔|
|–from-file=< FROM_FILE > | 从文件中创建互动的systrace|
|-e < DEVICE_SERIAL >,–serial=< DEVICE_SERIAL > | 指定设备|
|-l, –list-categories  |  列举可用的tags|

### 2.1.2 category

其中category可取值：

|category |   描述 |
| -----------       | :------------  |
|gfx |Graphics|
|input|   Input|
|view  |  View System|
|webview |WebView|
|wm  |Window Manager|
|am  |Activity Manager|
|sm  |Sync Manager|
|audio |  Audio|
|video  | Video|
|camera  |Camera|
|hal |Hardware Modules|
|app |Application|
|res |Resource Loading|
|dalvik | Dalvik VM|
|rs  |RenderScript|
|bionic | Bionic C Library|
|power  | Power Management|
|sched   |CPU Scheduling|
|irq IRQ |Events|
|freq    |CPU Frequency|
|idle    |CPU Idle|
|disk    |Disk I/O|
|mmc |eMMC commands|
|load |   CPU Load|
|sync  |  Synchronization|
|workq  | Kernel Workqueues|
|memreclaim|  Kernel Memory Reclaim|
|regulators | Voltage and Current Regulators|

## 2.2 Device Monitor（DDMS）

可以使用Eclipse或者Android Studio集成开发工具，切换到DDMS，点击devices，点击Systrace按钮：
<center>
![ddms-capture-systrace.png](/img/archives/ddms-capture-systrace.png)
</center>

补充说明：
- Destionation file ：trace输出的文件路径
- Trace duration : 配置抓取systrace的时间，通常设置5秒，并在5秒内重现问题，时间太短会导致问题重现时没有被抓到，时间太长会导致Java Heap不够而无法保存，因此在能抓到问题点的情况下，时间越小越好。
- Trace Buffer Size : Buffer Size是存储systrace的size，同样的，太小会导致信息丢失，时间太长会导致Java Heap不够而无法保存，建议20480。
- Enable Application Traces from :如果用户有自己在应用程序中加入自己的systrace log:
`Trace.beginSection("newInstance");
Trace.endSection();`
那么此处必须选择这个应用对应的进程名字，否则新加的systrace log不会被抓到。

# 三、自定义systrace

有时候为了debug方便，那么我们需要自己在apk或者framework层添加trace信息：

## 3.1 app层

app可以使用：

```
import android.os.Trace;
Trace.beginSection(String sectionName)
Trace.EndSection()
```

然后通过`python systrace.py --app=sectionName` 指定apk，或者通过ddms选择指定apk，抓取systrace分析。

## 3.2 Java framework层

Java Framework可以使用：

```
import android.os.Trace;
Trace.traceBegin(long traceTag, String methodName)
Trace.traceEnd(long traceTag)
```

抓取systrace分析。

## 3.3 Native framework层

Native Framework可以使用：最好在函数开头声明定义

```
#include <cutils/trace.h>
ATRACE_CALL()
```

抓取systrace分析。

# 四、分析systrace
Google Chrome浏览器可以打开systrace，如果打不开，可以通过`chrome://tracing/`，然后load systrace。

以分析UI Performance为例：

## 4.1 Frame

每个应用都有一行专门显示frame，每一帧就显示为圆圈，正常绘制是1秒60帧，大约一帧16.6毫秒，在这个值以下是正常颜色绿色，如果超过它就会变成红色、黄色。非绿色的都说明有问题。这时需要通过’w’键放大那一帧，然后按‘m’键高亮，进一步分析问题。

对于Android 5.0(API level 21)或者更高的设备，该问题主要聚焦在`UI Thread`和`Render Thread`这两个线程当中。对于更早的版本，则所有工作在`UI Thread`。

![systrace-fenxi.png](/img/archives/systrace-fenxi.png)

## 4.2 Alerts

Systrace能自动分析trace中的事件，并能自动高亮性能问题作为一个Alerts，建议调试人员下一步该怎么做。

比如对于丢帧是，点击黄色或红色的Frames圆点便会有相关的提示信息；另外，在systrace的最右上方，有一个Alerts tab可以展开，这里记录着所有的的警告提示信息。


# 五、快捷操作

## 5.1 导航操作

|导航操作   | 作用|
| ---- | : ---- |
|w  | 放大，[+shift]速度更快|
|s  | 缩小，[+shift]速度更快|
|a  | 左移，[+shift]速度更快|
|d  | 右移，[+shift]速度更快|

## 5.2 快捷操作

|常用操作 |   作用|
| ---- | : ---- |
|f|   放大当前选定区域|
|m|   标记当前选定区域|
|v| 高亮VSync|
|g|   切换是否显示60hz的网格线|
|0|  恢复trace到初始态，这里是数字0而非字母o|

|一般操作|    作用|
| ---- | : ---- |
|h |  切换是否显示详情|
|/ |  搜索关键字|
|enter|   显示搜索结果，可通过← →定位搜索结果|
|`|   显示/隐藏脚本控制台|
|? |  显示帮助功能|

对于脚本控制台，除了能当做记事本的功能，目前还不清楚有啥功能，或许还在开发中。

## 5.3 模式切换
- Select mode: 双击已选定区能将所有相同的块高亮选中；（对应数字1）
- Pan mode: 拖动平移视图（对应数字2）
- Zoom mode:通过上/下拖动鼠标来实现放大/缩小功能；（对应数字3）
- Timing mode:拖动来创建或移除时间窗口线。（对应数字4）

可通过按数字1~4，用于切换鼠标模式； 另外，按住alt键，再滚动鼠标滚轮能实现放大/缩小功能。

**Reference:**

- https://developer.android.com/studio/profile/systrace-commandline.html
- https://developer.android.com/studio/profile/systrace.html
- http://gityuan.com/2016/01/17/systrace/