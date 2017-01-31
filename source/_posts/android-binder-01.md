---
title: Binder实用指南（一） - 理解篇
categories:
  - Android进阶
tags:
  - Binder
date: 2016-12-21 21:29:35
---

这是关于Android Binder机制的一篇文章，Binder是Android里面非常重要的组成，也是最难理解的一块知识点，学习Binder最好的方法是深入源码阅读，因为Binder相关的知识错综复杂，一般初学者也很容易迷失在源码的汪洋里，本文旨在梳理Binder的架构和流程，并且试着以实用的角度来看待Binder。

<!-- more -->

# 一、为什么需要Binder机制？

Android系统中，每个应用程序是由Android的Activity，Service，Broadcast，ContentProvider这四剑客的中一个或多个组合而成，这四剑客所涉及的多进程间的通信底层都是依赖于Binder IPC机制。例如当进程A中的Activity要向进程B中的Service通信，这便需要依赖于Binder IPC。
如果熟悉Android源码，其实可以知道整个Android系统架构中，也大量采用了Binder机制作为IPC（进程间通信）方案。
Android是在Linux内核的基础上设计的。而在Linux中，已经拥有"管道/消息队列/共享内存/信号量/Socket等等"众多的IPC通信手段；但是，Google为什么单单选择了Binder，可见Binder肯定有自己独特的优势：

## 1.1 Binder能很好的实现C/S架构 

Android系统，很大一部分都是居于Client-Server架构的设计。Client端有什么需求，直接发送给Server端去完成，Server处理完毕再将反馈内容发送给Client。Server端与Client端相对独立，稳定性较好。传统的CS架构只有Socket，但是Socket通信效率相对于其他IPC来说又太低效，而Binder正是基于C/S架构设计的。

## 1.2 Binder传输效率高

Binder只需要进行一次拷贝，把Client端的用户空间的数据即copy_from_user()到内核空间，然后将内核空间的数据映射到Server端的用户空间。
Binder性能上仅仅次于Linux 共享内存的方式，但是共享内存的方式，进程间同步又是一个难题。

## 1.3 Binder安全性极高

Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限。
Client-Server通信过程中，Binder内核会为每个Client进程分配了UID/PID来作为鉴别身份的标示，并且在Binder通信时会根据UID/PID进行有效性检测。
而如果是传统的IPC只能由在数据包当中填入UID/PID，这个并不是一个可靠的方法。

知乎上有一位答主讲得很好，可以看看:
> **[为什么 Android 要采用 Binder 作为 IPC 机制?](https://www.zhihu.com/question/39440766)**

# 二、Binder原理

![binder-ipc](/img/archives/binder-01.jpg)

1. Binder采用Client-Server架构，包含Client、Server、ServiceManager、Binder驱动四个组件。
2. **应用程序都运行在用户空间，每个应用程序都有它自己独立的内存空间**；若不同的应用程序之间涉及到通信，需要通过内核进行中转，因为需要用到内核的`copy_from_user()`和`copy_to_user()`等函数
3. Server进程要先注册Service到ServiceManager，Client进程使用某Server的Service前，须先向ServiceManager中获取相应的Service，然后使用Service。

# 三、Binder驱动层
<center>
![binder-driver](/img/archives/binder-02.png)
</center>

当用户空间调用open()方法，最终会调用binder驱动的binder_open()方法；mmap()/ioctl()方法也是同理，从用户态进入内核态，都依赖于系统调用过程。

## 3.1 binder_init

注册misc设备，指定相应文件操作的方法。

## 3.2 binder_open

创建binder_proc对象，并把当前进程等信息保存到binder_proc对象，该对象管理IPC所需的各种信息并拥有其他结构体的根结构体；再把binder_proc对象保存到文件指针filp，以及把binder_proc加入到全局链表binder_procs。

## 3.3 binder_mmap

在内核虚拟地址空间，申请一块与用户虚拟内存相同大小的内存；然后再申请1个page大小的物理内存，再将同一块物理内存分别映射到内核虚拟地址空间和用户虚拟内存空间，从而实现了用户空间的Buffer和内核空间的Buffer同步操作的功能。

## 3.4 binder_ioctl

负责在两个进程间收发IPC数据和IPC reply数据。调用流程比如：

```c
//step 1:
binder_write_read bwr;
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) 
    // step 2:
    binder_ioctl(filp, BINDER_WRITE_READ, &bwr)
        // step 3:
        binder_ioctl_write_read(filp, BINDER_WRITE_READ, &bwr, thread)
            // step 4:
            copy_from_user(&bwr, ubuf, sizeof(bwr))
            binder_thread_write(proc, thread, bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
            binder_thread_read(proc, thread, bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
            copy_to_user(...)
```

**binder_thread_write()**：处理Binder请求码，以”BC_“开头，简称BC码，用于从IPC层传递到Binder Driver层；
**binder_thread_read()**：生成Binder响应码，以”BR_“开头，简称BR码，用于从Binder Driver层传递到IPC层；

![binder-04](/img/archives/binder-04.jpg)

# 四、Binder通信流程

例如当名为`BatteryStatsService`的Client向`ServiceManager`注册服务的过程中，IPC层的数据组成为：
**Handle=0，RPC代码为ADD_SERVICE_TRANSACTION，RPC数据为BatteryStatsService，Binder协议为BC_TRANSACTION。**
整个流程图大致如下：
<center>
![binder-05](/img/archives/binder-05.png)
</center>

handle为0正是指向ServiceManager。

# 五、启动ServiceManager

ServiceManager启动时序图：

![servicemanager_create](/img/archives/create_servicemanager.jpg)

1. 打开binder驱动，并调用mmap()方法分配128k的内存映射空间：binder_open();
2. 通知binder驱动使其成为守护进程：binder_become_context_manager()；
3. 验证selinux权限，判断进程是否有权注册或查看指定服务；
4. 进入循环状态，等待Client端的请求：binder_loop()。

# 六、获取ServiceManager

获取Service Manager是通过defaultServiceManager()方法来完成，当进程注册服务(addService)或 获取服务(getService)的过程之前，都需要先调用defaultServiceManager()方法来获取gDefaultServiceManager对象。

![get_servicemanager.jpg](/img/archives/get_servicemanager.jpg)

1. 获取ProcessState对象，在其构造函数中调用open_driver函数打开Binder驱动，并将句柄保存到mDriverFD；
2. 调用gProcess->getContextObject函数来获得一个句柄值为0的Binder引用，即BpBinder；
3. 通过`interface_cast`构造一个BpServiceManager对象，所以gDefaultServiceManager最终为`new BpServiceManager(new BpBinder(0))`。

# 七、addService

以Native层的服务以media服务为例，注册MediaPlayerService的时序图如下：

![add-servive](/img/archives/add-service.jpg)

1. defaultServiceManager()返回的是BpServiceManager，会调用`BpServiceManager.addService`方法
2. `addService()`通过remote()中保存的BpBinder调用到IPCThreadState的transact方法；
3. IPCThreadState::transact会调用writeTransactionData()传输数据传输数据，然后和驱动交互，驱动把请求转发给ServiceManager执行真正的注册服务；
4. 得到驱动的返回后，调用BBinder，最终调用到BnMediaPlayerService的onTransact方法；
5. 开启两个线程不断和Binder进行交互，获取Client请求。

获取服务的流程基本也是差不多的，不再累述。

# 八、Binder架构

binder在framework层，采用JNI技术来调用native(C/C++)层的binder架构，从而为上层应用程序提供服务。 我们知道native层中，binder是C/S架构，分为Bn端(Server)和Bp端(Client)。对于java层在命名与架构上非常相近，同样实现了一套IPC通信架构。

![java-binder](/img/archives/java_binder.jpg)

1.BinderProxy类代码Client端,Binder类代表Server端
2.framework层的Binder逻辑是建立在Native层架构基础之上的，核心逻辑都是交予Native层方法来处理

**比如addService流程：**
1.java层通过getIServiceManager获得ServiceManagerProxy对象，通过该对象的BinderProxy，最终会调用BpBinder对象，由BpBinder来完成通信。
2.Binder驱动将Client端的请求转发给BBinder的transact方法，然后由其子类JavaBBinder调用。后者会调用指定Service的方法，并返回给驱动。

# 九、Binder类图

## 9.1 Native Binder类图

![binder_native_class.jpg](/img/archives/binder_native_class.jpg)

## 9.2 Framework Binder类图

![class_ServiceManager.jpg](/img/archives/class_ServiceManager.jpg)

# 十、Binder其他

介绍一些Binder其他比较重要的点，方便理清Binder的一些疑问。比如Binder实体和引用，比如ProcessState和IPCThreadState，比如数据结构怎么传递等。

## 10.1 Binder中各个角色的关系

![binder_relationship.jpg](/img/archives/binder_relationship.jpg)

**1. Binder实体 : binder_node**

Binder实体，是各个Server以及ServiceManager在内核中的存在形式。
Binder实体实际上是内核中 **`binder_node`** 结构体的对象，它的作用是在内核中保存Server和ServiceManager的信息(例如，Binder实体中保存了Server对象在用户空间的地址)。简言之，Binder实体是Server在Binder驱动中的存在形式，内核通过Binder实体可以找到用户空间的Server对象。
在上图中，Server和ServiceManager在Binder驱动中都对应的存在一个Binder实体。

**2. Binder引用 : binder_ref**

所谓Binder引用，实际上是内核中**`binder_ref`**结构体的对象，它的作用是在表示"Binder实体"的引用。换句话说，每一个Binder引用都是某一个Binder实体的引用，通过Binder引用可以在内核中找到它对应的Binder实体。
如果将Server看作是Binder实体的话，那么Client就好比Binder引用。Client要和Server通信，它就是通过保存一个Server对象的Binder引用，再通过该Binder引用在内核中找到对应的Binder实体，进而找到Server对象，然后将通信内容发送给Server对象。
Binder实体和Binder引用都是内核(即Binder驱动)中的数据结构。每一个Server在内核中就表现为一个Binder实体，而每一个Client则表现为一个Binder引用。这样，每个Binder引用都对应一个Binder实体，而每个Binder实体则可以多个Binder引用。

**3. 远程服务**

Server都是以服务的形式注册到ServiceManager中进行管理的。如果将Server本身看作是"本地服务"的话，那么Client中的"远程服务"就是本地服务的代理。如果你对代理模式比较熟悉的话，就很容易理解了，远程服务就是本地服务的一个代理，通过该远程服务Client就能和Server进行通信。

## 10.2 进程和线程的关系

![binder_proc_relation.png](/img/archives/binder-proc-relation.png)

** 图解：**
1.Binder驱动通过`binder_procs`链表记录所有创建的`binder_proc`结构体，binder驱动层的每一个binder_proc结构体都与用户空间的一个用于binder通信的进程一一对应。
2.每个进程有且只有一个`ProcessState`对象，这是通过单例模式来保证的。
3.每个进程中可以有很多个线程，每个线程对应一个`IPCThreadState`对象，IPCThreadState对象也是单例模式，即一个线程对应一个IPCThreadState对象，在Binder驱动层也有与之相对应的结构，那就是`Binder_thread`结构体。在binder_proc结构体中通过成员变量`rb_root threads`，来记录当前进程内所有的binder_thread。

**Binder线程池：**
每个Server进程在启动时会创建一个binder线程池，并向其中注册一个Binder线程；之后Server进程也可以向binder线程池注册新的线程，或者Binder驱动在探测到没有空闲binder线程时会主动向Server进程注册新的的binder线程。对于一个Server进程有一个最大Binder线程数限制，默认为16个binder线程，例如Android的system_server进程就存在16个线程。对于所有Client端进程的binder请求都是交由Server端进程的binder线程来处理的。

## 10.3 Binder数据传输

![binder_data.jpg](/img/archives/binder_data.jpg)

当Client向Server发送请求时，Client会将数据打包成上述格式，然后通过ioctl()发送给Binder驱动。

1. 用户空间的进程调用**ioctl(fd,BINDER_WRITE_READ,&bwr)**时传递给Binder驱动的信息。fd是Binder驱动的文件句柄，BINDER_WRITE_READ是ioctl()的一个标识，而bwr是传递的数据，`write_buffer`是请求数据的内容，而`write_consumed`是用来记录请求数据中已经被Binder驱动处理过的数据的大小。
2. ioctl会走到**binder_thread_write**和**binder_thread_read**。这层的数据是"事务指令"+"binder_transaction_data结构体"组成的。data是保存事务中具体数据的内存地址。具体调用流程可以参考**#3.4章节**
3. 这层是有效数据。如果该请求是传递给ServiceManager进行处理的，则有效数据是：消息头+"Server的相关信息"。消息头是用来进行有效性检查的，而"Server的相关信息"则是请求要处理的信息。


# 十一、源码目录

从上之下, 整个Binder架构所涉及的总共有以下5个目录:
```
/framework/base/core/java/               (Java)
/framework/base/core/jni/                (JNI)
/framework/native/libs/binder            (Native)
/framework/native/cmds/servicemanager/   (Native)
/kernel/drivers/staging/android          (Driver)
```

## 11.1 Java framework
```
/framework/base/core/java/android/os/  
    - IInterface.java
    - IBinder.java
    - Parcel.java
    - IServiceManager.java
    - ServiceManager.java
    - ServiceManagerNative.java
    - Binder.java  
    
/framework/base/core/jni/    
    - android_os_Parcel.cpp
    - AndroidRuntime.cpp
    - android_util_Binder.cpp (核心类)
```

## 11.2 Native framework

```
/framework/native/libs/binder         
    - IServiceManager.cpp
    - BpBinder.cpp
    - Binder.cpp
    - IPCThreadState.cpp (核心类)
    - ProcessState.cpp  (核心类)

/framework/native/include/binder/ 
    - IServiceManager.h
    - IInterface.h

/framework/native/cmds/servicemanager/ 
    - service_manager.c
    - binder.c
```

## 11.3 Kernel

```
/kernel/drivers/staging/android/
    - binder.c
    - uapi/binder.h
```