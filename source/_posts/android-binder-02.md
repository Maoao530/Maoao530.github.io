---
title: Binder实用指南（二） - 实战篇
categories:
  - Android进阶
tags:
  - Binder
date: 2016-12-25 21:45:33
---

本章的内容主要说明如何在JavaFramework层和Native层自定义Client-Server组件，并且使用Binder进行通信。

<!-- more -->

# 1 Native Binder

源码目录结构:
> alps/frameworks/native/cmds/NativeBinderDemo/

```
|-NativeBinderDemo
|---ClientDemo.cpp: 客户端程序
|---ServerDemo.cpp：服务端程序
|---IMyService.h：自定义的MyService服务的头文件
|---IMyService.cpp：自定义的MyService服务
|---Android.mk：源码build文件

```

## 1.1 服务端

> alps/frameworks/native/cmds/NativeBinderDemo/ServerDemo.cpp

获取ServiceManager，注册service.myservice服务到ServiceManager，启动服务。

```cpp
#include "IMyService.h"

int main() {
    sp < IServiceManager > sm = defaultServiceManager(); //获取service manager引用
    sm->addService(String16("service.myservice"), new BnMyService()); //注册名为"service.myservice"的服务到service manager
    ProcessState::self()->startThreadPool(); //启动线程池
    IPCThreadState::self()->joinThreadPool(); //把主线程加入线程池
    return 0;
}
```

## 1.2 客户端

> alps/frameworks/native/cmds/NativeBinderDemo/ClientDemo.cpp

获取ServiceManager，拿到service.myservice服务，再进行类型转换成IMyService，最后调用远程方法sayHello()

```cpp
#include "IMyService.h"

int main() {
    sp < IServiceManager > sm = defaultServiceManager(); //获取service manager引用
    sp < IBinder > binder = sm->getService(String16("service.myservice"));//获取名为"service.myservice"的binder接口
    sp<IMyService> cs = interface_cast < IMyService > (binder);//将biner对象转换为强引用类型的IMyService
    cs->sayHello();//利用binder引用调用远程sayHello()方法
    return 0;
}
```

## 1.3 MyService

> alps/frameworks/native/cmds/NativeBinderDemo/IMyService.h

申明IMyService，申明BpMyService（Binder客户端），申明BnMyService（Binder的服务端）。

```cpp
#ifndef MY_SERVICE_DEMO
#define MY_SERVICE_DEMO
#include <stdio.h>
#include <binder/IInterface.h>
#include <binder/Parcel.h>
#include <binder/IBinder.h>
#include <binder/Binder.h>
#include <binder/ProcessState.h>
#include <binder/IPCThreadState.h>
#include <binder/IServiceManager.h>
using namespace android;
namespace android
{
    class IMyService : public IInterface
    {
    public:
        DECLARE_META_INTERFACE(MyService); //使用宏，申明MyService
        virtual void sayHello()=0; //定义方法
    };

        //定义命令字段
    enum
    {
        HELLO = 1,
    };
    
        //申明客户端BpMyService
    class BpMyService: public BpInterface<IMyService> {
    public:
        BpMyService(const sp<IBinder>& impl);
        virtual void sayHello();
    };

        //申明服务端BnMyService
        class BnMyService: public BnInterface<IMyService> {
        public:
            virtual status_t onTransact(uint32_t code, const Parcel& data, Parcel* reply,
                    uint32_t flags = 0);
            virtual void sayHello();
        };
}
#endif
```

> alps/frameworks/native/cmds/NativeBinderDemo/IMyService.cpp

```cpp
#include "IMyService.h"
namespace android
{
    IMPLEMENT_META_INTERFACE(MyService, "android.demo.IMyService"); //使用宏，完成MyService定义
   
  //客户端
    BpMyService::BpMyService(const sp<IBinder>& impl) :
            BpInterface<IMyService>(impl) {
    }
    
    // 实现客户端sayHello方法
    void BpMyService::sayHello() {
        printf("BpMyService::sayHello\n");
        Parcel data, reply;
        data.writeInterfaceToken(IMyService::getInterfaceDescriptor());
        remote()->transact(HELLO, data, &reply);
        printf("get num from BnMyService: %d\n", reply.readInt32());
    }
    
    //服务端，接收远程消息，处理onTransact方法
    status_t BnMyService::onTransact(uint_t code, const Parcel& data,
            Parcel* reply, uint32_t flags) {
        switch (code) {
        case HELLO: {    //收到HELLO命令的处理流程
            printf("BnMyService:: got the client hello\n");
            CHECK_INTERFACE(IMyService, data, reply);
            sayHello();
            reply->writeInt32(2015);
            return NO_ERROR;
        }
            break;
        default:
            break;
        }
        return NO_ERROR;
    }
    
    // 实现服务端sayHello方法
    void BnMyService::sayHello() {
        printf("BnMyService::sayHello\n");
    };
}
```

## 1.4 Android.mk
```
LOCAL_PATH := $(call my-dir)
 

include $(CLEAR_VARS)
LOCAL_SHARED_LIBRARIES := \
    libcutils \
    libutils \
    libbinder       
LOCAL_MODULE    := ServerDemo
LOCAL_SRC_FILES := \
    IMyService.cpp \
    ServerDemo.cpp
   
LOCAL_MODULE_TAGS := optional
include $(BUILD_EXECUTABLE)
  

include $(CLEAR_VARS)
LOCAL_SHARED_LIBRARIES := \
    libcutils \
    libutils \
    libbinder
LOCAL_MODULE    := ClientDemo
LOCAL_SRC_FILES := \
    IMyService.cpp \
    ClientDemo.cpp
LOCAL_MODULE_TAGS := optional
include $(BUILD_EXECUTABLE)
```

## 1.5 原理图

![native_binder_demo.jpg](/img/archives/native_binder_demo.jpg)

## 1.6 运行

**编译：**
mm alps/frameworks/native/cmds/NativeBinderDemo目录，然后到alps/out/target/product/{Project}/system/bin/会生成`ClientDemo`和`ServerDemo`

**执行：**
```
adb remount
adb push ServerDemo /system/bin
adb push ClientDemo /system/bin
```
开两个窗口分别执行下面两个命令便可以看到结果了：
`adb shell system/bin/ServerDemo` ， `adb shell system/bin/ClientDemo`


# 2 JavaFramework Binder

源码目录结构:
> alps/frameworks/base/cmds/FrameworkBinderDemo/

```
|-Server端
|---ServerDemo.java：可执行程序
|---IMyService.java: 定义IMyService接口
|---MyService.java：定义MyService
|-Client端
|---ClientDemo.java：可执行程序
|---IMyService.java: 与Server端完全一致
|---MyServiceProxy.java：定义MyServiceProxy
```

## 2.1 Server端

**(1)ServerDemo.java**

可执行程序

```java
public class ServerDemo {
    public static void main(String[] args) {
        System.out.println("MyService Start");
        //准备Looper循环执行
        Looper.prepareMainLooper();
        //设置为前台优先级
        android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
        //注册服务
        ServiceManager.addService("MyService", new MyService());
        Looper.loop();
    }
}
```

**(2)IMyService.java**

定义sayHello()方法，DESCRIPTOR属性

```java
public interface IMyService extends IInterface {
    static final java.lang.String DESCRIPTOR = "com.yuanhh.frameworkBinder.MyServer";
    public void sayHello(String str) throws RemoteException ;
    static final int TRANSACTION_say = android.os.IBinder.FIRST_CALL_TRANSACTION;
}
```
**(3)MyService.java**

```java
public class MyService extends Binder implements IMyService{

    public MyService() {
        this.attachInterface(this, DESCRIPTOR);
    }

    @Override
    public IBinder asBinder() {
        return this;
    }

    /** 将MyService转换为IMyService接口 **/
    public static com.yuanhh.frameworkBinder.IMyService asInterface(
            android.os.IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iInterface = obj.queryLocalInterface(DESCRIPTOR);
        if (((iInterface != null)&&(iInterface instanceof com.yuanhh.frameworkBinder.IMyService))){
            return ((com.yuanhh.frameworkBinder.IMyService) iInterface);
        }
        return null;
    }

    /**  服务端，接收远程消息，处理onTransact方法  **/
    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case INTERFACE_TRANSACTION: {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_say: {
            data.enforceInterface(DESCRIPTOR);
            String str = data.readString();
            sayHello(str);
            reply.writeNoException();
            return true;
        }}
        return super.onTransact(code, data, reply, flags);
    }

    /** 自定义sayHello()方法   **/
    @Override
    public void sayHello(String str) {
        System.out.println("MyService:: Hello, " + str);
    }
}
```

**(4) Android.mk**

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := ServerDemo 
LOCAL_MODULE_TAGS := optional
include $(BUILD_JAVA_LIBRARY)


include $(CLEAR_VARS)
LOCAL_MODULE := ServerDemo
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_PATH := $(TARGET_OUT)/bin
LOCAL_MODULE_CLASS := UTILITY_EXECUTABLES
LOCAL_SRC_FILES := ServerDemo
include $(BUILD_PREBUILT)
```

**(5) ServerDemo**

```
base=/system
export CLASSPATH=$base/framework/ServerDemo.jar
exec app_process $base/bin com.shun.frameworkBinder.ServerDemo "$@"
```
## 2.2 Client端

**(1)ClientDemo.java**

可执行程序

```java
public class ClientDemo {

    public static void main(String[] args) throws RemoteException {
        System.out.println("Client start");
        IBinder binder = ServiceManager.getService("MyService"); //获取名为"MyService"的服务
        IMyService myService = new MyServiceProxy(binder); //创建MyServiceProxy对象
        myService.sayHello("binder"); //通过MyServiceProxy对象调用接口的方法
        System.out.println("Client end");
    }
}
```

**(2)IMyService.java**

与Server端的IMyService是一致，基本都是拷贝一份过来。

**(3)MyServiceProxy.java**

```java
public class MyServiceProxy implements IMyService {
    private android.os.IBinder mRemote;  //代表BpBinder

    public MyServiceProxy(android.os.IBinder remote) {
        mRemote = remote;
    }

    public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
    }

    /** 自定义的sayHello()方法   **/
    @Override
    public void sayHello(String str) throws RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeString(str);
            mRemote.transact(TRANSACTION_say, _data, _reply, 0);
            _reply.readException();
        } finally {
            _reply.recycle();
            _data.recycle();
        }
    }

    @Override
    public IBinder asBinder() {
        return mRemote;
    }
}
```

** (4) Android.mk**

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := ClientDemo 
LOCAL_MODULE_TAGS := optional
include $(BUILD_JAVA_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := ClientDemo
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_PATH := $(TARGET_OUT)/bin
LOCAL_MODULE_CLASS := UTILITY_EXECUTABLES
LOCAL_SRC_FILES := ClientDemo
include $(BUILD_PREBUILT)
```

** (5) ClientDemo **

```
base=/system
export CLASSPATH=$base/framework/ClientDemo.jar
exec app_process $base/bin com.shun.frameworkBinder.ClientDemo "$@"
```
## 2.3 原理图

![MyServer_framework_binder.jpg](/img/archives/MyServer_framework_binder.jpg)

## 2.4 运行

**编译：**

mm alps/frameworks/base/cmds/FrameworkBinderDemo/目录，然后：
alps/out/target/product/{Project}/system/bin/        生成ClientDemo和ServerDemo 
alps/out/target/product/{Project}/system/framework/  生成ClientDemo.jar和ServerDemo.jar。

**执行：**

```
adb remount
adb push ServerDemo /system/bin
adb push ClientDemo /system/bin
adb push ServerDemo.jar /system/framework/
adb push ClientDemo.jar /system/framework/
```

开两个窗口分别执行下面两个命令便可以看到结果了：
adb shell system/bin/ServerDemo ， adb shell system/bin/ClientDemo


