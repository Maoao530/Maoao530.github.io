---
title: Android源码学习指南 - 【置顶】
categories:
  - Android进阶
tags:
  - source
  - android
top:
 - 10
date: 2017-02-06 21:31:21
---

深入学习Android源码，知其然而知其所以然。计划整理一系列源码分析文章。

<!-- more -->


# 一、准备

- Java/C/C++基础，Java Framework和C++ Framework，一部分Lib则采用C。 
- Linux相关知识，Android是基于Linux内核。
- Makefile知识，Android采用make编译，可以看到有很多Android.mk类似的文件。
- Source insight，源码阅读工具神器。
- StarUML，类图工具。
- ProcessOn/Visio，流程图工具。

# 二、入门学习

入门学习建议可以参考如下：

- Android官方培训 [Android training](https://developer.android.com/training/index.html)
- 胡凯发起的 [Android官方培训课程中文版](http://hukai.me/android-training-course-in-chinese/index.html)

# 三、进阶学习

作为一名合格的码农，肯定不仅仅限于使用API文档，因为浮于表面是远远不够的。进阶学习的阶段，需要我们保持一颗好奇的心，深入阅读Android源码，学习优秀的代码风格和设计思想，知其然并且知其所以然。

引用Google的一张框架图：

![android-framework](/img/archives/android-framework.png)

- **Applications层**，和用户直接交互的就是这些应用程序，它们都是用Java开发的。
- **Java Framework层**，这一层大部分用Java语言编写。它是Android平台上Java世界的基石。
- **C++ Framework/Libraries层**，这一层提供动态库（也叫共享库）、Android运行时库、Dalvik虚拟机等。从编程语言上来说，这一层大部分都是用C或C++写的，所以也可以简单地把它看成是Native层。
- **Linux Kernel层**，Android是基于Linux内核，其核心系统服务如安全性、内存管理、进程管理、网路协议以及驱动模型都依赖于Linux内核。

# 四、Android架构

Google提供的四层架构非常经典，如果我们要深入学习这个架构，**最好就是以Android系统启动流程开始学起，然后一步一步展开，牵引学习**。这样不至于我们陷入源码的大海里而没有方向。

这个是之前我画的一张启动流程图：

![android-boot-up.png](/img/archives/android-boot-up.png)

Java和C++通过JNI连接，C/C++通过system call直接调用linux os。

## 4.1 Loader层

**1. Boot ROM: **
上电后，BootRom会被激活，引导芯片代码开始从预定义的地方（固化在ROM）开始执行，然后加载引导程序到RAM。

**2. Boot Loader引导程序**
Boot Loader是启动Android系统之前的引导程序，引导程序是OEM厂商或者运营商加锁和限制的地方，它是针对特定的主板与芯片的。OEM厂商要么使用很受欢迎的引导程序比如redboot、uboot、ARMboot等或者开发自己的引导程序，它不是Android操作系统的一部分。
Boot Loader主要作用是检查RAM，初始化硬件参数等功能。

## 4.2 Kernel层

Kernel的启动流程：

> alps/kernel/init/main.c
> start_kernel() ==> rest_init() ==> kernel_thread(kernel_init) ==> kernel_init()


**0号进程：**
swapper进程(pid=0)：又称为idle进程, 叫空闲进程，由系统自动创建, 运行在内核态。
系统初始化过程Kernel由无到有开创的第一个进程, 也是唯一一个没有通过fork或者kernel_thread产生的进程。
swapper进程用于初始化进程管理、内存管理，加载Display,Camera Driver，Binder Driver等相关工作。

**1号进程 **
init进程(pid=1)：由0号进程通过kernel_thread创建，在内核空间完成初始化后, 加载init程序, 并最终运行在用户空间，init进程是所有用户进程的鼻祖。

**2号进程 **
kthreadd进程(pid=2)：由0号进程通过kernel_thread创建，是Linux系统的内核进程，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。
kthreadd运行在内核空间, 负责所有内核线程的调度和管理 , kthreadd进程是所有内核进程的鼻祖。

## 4.3 Native层

Native层主要是init一号进程，并且由其孵化出来的一系列daemon进程，还有一些列native service。

- init进程会孵化出ueventd、logd、healthd、installd、adbd、lmkd等用户守护进程；
- init进程还启动servicemanager、bootanim、mediaserver等重要本地服务
- init进程孵化出Zygote进程，Zygote进程是Android系统的第一个Java进程，Zygote是所有Java进程的父进程。

## 4.4 Java层

- Zygote是第一个Java进程，并且是所有java进程的父进程，由init进程解析init.rc文件后fork生成。
- System Server进程，是由Zygote进程fork而来，System Server是Zygote孵化的第一个进程。System Server负责启动和管理整个Java framework，包含ActivityManager，PackageManager，WindowManager等服务。

## 4.5 Application层

Zygote进程孵化出的第一个App进程是Launcher，Zygote进程还会创建Browser，Phone，Email等App进程，每个App至少运行在一个进程上。所有的App进程都是由Zygote进程fork生成的。


# 五、学习计划

了解了大致的框架体系之后，接下来就是给自己列一个学习计划。博主不定期总结学习内容到博客上面来，与大家一起分享。博客会以Android N源码为主来分析，尽量每个知识点做到多画图，多总结，少贴大段源码，避免犯困。

## 5.1 四大组件

Android应用的四大组件Activity，Service，Broadcast Receiver， Content Provider。

- 四大组件基础知识
 - [Android四大组件](http://maoao530.github.io/2016/08/11/android-base/)
- Android组件 - Activity
- Android组件 - Service
- Android组件 - Broadcast Receiver
- Android组件 - Content Provider

## 5.2 消息处理机制

- Android消息处理机制 Looper、Handler、Message介绍
 - [Android消息机制](http://maoao530.github.io/2016/12/11/android-looper-handler-message/)

## 5.3 IPC通信

- Binder机制
 - [Binder实用指南（一） - 理解篇](http://maoao530.github.io/2016/12/21/android-binder-01/)
 - [Binder实用指南（二） - 实战篇](http://maoao530.github.io/2016/12/25/android-binder-02/)
- Socket通信

## 5.4 系统启动

- Android系统启动总结，包含如何启动`init进程`，如何启动`zygote进程`和`system_server进程`：
 - [Android系统启动流程](http://maoao530.github.io/2017/01/06/android-start/)
- init进程相关包含init rc语法
- Zygote进程相关知识
- system_server进程
- DVM的启动

## 5.4 系统服务

- Android系统服务 - ActivityManagerService
 - AMS启动流程
 - AMS的Activity调度
- Android系统服务 - PackageManagerService
 - [PackageManagerService启动流程](http://maoao530.github.io/2017/01/10/packagemanager/)
 - [应用程序安装流程](http://maoao530.github.io/2017/01/18/package-install/)
 - 应用程序卸载流程
 - Installd守护进程
- Android系统服务 - WindowManagerService
 - Surface View原理
- Android系统服务- SurfaceFlinger推图
- Input系统
 - InputReader介绍
 - InputDispatcher介绍
- Android系统服务 - PowerManagerService
- Android系统服务 - UserManagerService
- MediaServer
 - AudioFlinger - 处理上层AudioTrack创建的音频
 - MediaPlayerService服务：StageFrightPlayer本地播放、NuPlayer在线播放

## 5.5 安全机制

- [Android系统build阶段签名机制](http://maoao530.github.io/2017/01/31/android-build-sign/)
- [APK签名机制](http://maoao530.github.io/2017/01/31/apk-sign/)
- APK逆向 - smali注入

## 5.6 问题分析

- ANR问题原理和分析
- Crash/Exception问题分析
 - Java Exception
 - Native Exception
 - Kernel Panic
- LowMemoryKiller

## 5.7 工具篇

- Android.mk介绍
- [理解和使用systrace](http://maoao530.github.io/2017/02/06/systrace/)

## 5.8 其它

- [Android智能指针](http://maoao530.github.io/2016/12/10/cpp-ref-count/)

# 六、写在最后

由于博主水平和经历有限，肯定还有很多知识点遗漏，这一份计划大纲会在后续的学习过程中不断补充完善和改正，博主打算整理一系列源码分析的文章，如果有理解或者表达错误的地方，还请读者海涵并且指正。

可以通过给我留言或者邮件：maoao530@foxmail.com，感激。