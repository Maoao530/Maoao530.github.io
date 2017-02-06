---
title: Android源码学习指南 - 【置顶】
categories:
  - Android进阶
tags:
  - source
  - android
date: 2017-02-06 21:31:21
---

Android源码的正确打开方式。

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

- [Android training](https://developer.android.com/training/index.html)
- 胡凯发起的 [Android官方培训课程中文版](http://hukai.me/android-training-course-in-chinese/index.html)

# 三、进阶学习

作为一名合格的码农，肯定不仅仅限于使用API文档，因为浮于表面是远远不够的。进阶学习的阶段，需要我们保持一颗好奇的心，深入阅读Android源码，学习优秀的代码风格和设计思想，知其然并且知其所以然。

引用Google的一张框架图：

![android-framework](/img/archives/android-framework.png)

- Applications层了，和用户直接交互的就是这些应用程序，它们都是用Java开发的。
- Java Framework层，这一层大部分用Java语言编写。它是Android平台上Java世界的基石。
- C++ Framework/Libraries层，这一层提供动态库（也叫共享库）、Android运行时库、Dalvik虚拟机等。从编程语言上来说，这一层大部分都是用C或C++写的，所以也可以简单地把它看成是Native层。
- Linux内核层，Android是基于Linux内核，其核心系统服务如安全性、内存管理、进程管理、网路协议以及驱动模型都依赖于Linux内核。

http://wiki.jikexueyuan.com/project/deep-android-v1/prepare.html



