---
title: 应用程序安装流程
categories:
  - Android进阶
tags:
  - PMS
  - Android
date: 2017-01-18 14:48:01
---

本文介绍APK的安装流程。

<!-- more -->

# 一、安装流程图

APK安装流程，总体可以下图流程，用ProcessOn画的，凑合看：

![package_install](/img/archives/package-install.png)

从上图我们可以看到apk安装到最后都会调用到这个flow：

> PMS.scanPackageTracedLI => PMS.scanPackageLI => PMS.scanPackageDirtyLI

关于这个flow，之前的博客有详细介绍过，本文不再展开 : [https://maoao530.github.io/2017/01/10/packagemanager/](https://maoao530.github.io/2017/01/10/packagemanager/)

后续的博文会根据这张图展开说明。

# 二、APK文件结构

APK(Android Package)，可以看做是一个zip压缩包，可以通过解压缩工具解开，其文件结构如下：

| 目录 or 文件           | 描述           |
| -----------            | :------------  |
| assert                 | 存放的原生资源文件,通过AssetManager类访问   |
| lib                    | native库文件|
| META-INF               | 存放签名信息，用来保证APK包的完整性和系统的安全。系统安装APK时，应用管理器会按照对应算法对包里文件做校验，如果校验结果与META-INF中内容不一致，则不会安装这个APK。|
| res                    | 种资源文件系统会在R.java里面自动生成该资源文件的ID，所以访问这种资源文件比较简单，通过R.XXX.ID即可|
| AndroidManifest.xml    | 每个应用都必须定义和包含，描述应用的名字、版本权限、引用的库文件等信息。apk中的AndroidManifest.xml经过压缩，可以通过AXMLPrinter2工具解开。|
| classes.dex            | 是JAVA源码编译后生成的JAVA字节码文件。但Android使用的dalvik虚拟机与标准的JAVA虚拟机不兼容，dex文件与class文件相比，不论是文件结构还是opcode都不一样。|
| resources.arsc         | 编译后的二进制资源文件。|

# 三、APK安装方法

APK有下面4种安装方法：

| 方法             | 描述 |
| ---              | :--- |
|开机过程中安装    |开机时完成，没有安装界面，如系统应用、其它预置应用|
|adb工具安装       |没有安装界面，adb install/push xxxx.apk|
|第三方应用安装    |通过packageinstaller.apk进行安装，有安装界面，如打开文件管理器并点击sdk卡里APK文件|
|网络下载应用安装  |通过google market应用完成，没有安装界面 |

简单说明下apk安装的基本过程：
- 拷贝目标apk到指定文件目录
- 调用scanPackageLI为apk文件在系统中注册信息

# 四、应用程序安装过程

上述几种安装方法最终都通过PackageManagerService.scanPackageLI完成，总结起来大致有以下三种方式：

- **adb push：**
PackageManagerService的内部类AppDirObserver实现了监听app目录的功能，当把某个APK文件放到app目录下面时，PMS会收到ADD_EVENTS事件
frameworks\base\services\java\com\android\server\pm\PackageManagerService.java

- **adb install：**
安装入口函数为Pm.runInstall
frameworks\base\cmds\pm\src\com\android\commands\pm\Pm.java 

- **网络下载应用安装和第三方应用安装：**
安装入口函数为ApplicationPackageManager.installPackage
 frameworks\base\core\java\android\app\ApplicationPackageManager.java



接下来我们来分别详细说明这些安装流程：

# 五、adb push 

Android 4.4平台，PackageManagerService的内部类AppDirObserver实现了监听app目录的功能，当把某个APK文件放到app目录下面时，PMS会收到ADD_EVENTS事件。
如果是添加事件，则调用scanPackageLI，并使用updatePermissionsLPw授权；如果是删除事件则调用removePackageLI移除该apk的相关信息。最后都要调用writeLPr重新保存相关信息到packages.xml。

关于AppDirObserver具体如何监听的，可以查看：[AppDirObserver](http://blog.csdn.net/new_abc/article/details/12949535)

不过我在android 7.0 sdk里面没有看到这个类，难道7.0把这个功能砍了？手头没有7.0平台，不好验证。

我猜测现在通过adb push apk到data/app或者system/app的apk，如果这个监听的功能砍了，那么应该是会通过reboot重启系统，走PMS.main流程，scanDir-->scanPackageLI去安装apk。

以上待填坑。

# 六、adb install 

adb install 的安装方式，会调用system/core/adb/commandline.cpp中的adb_commandline函数：
```
adb_commandline
    install_app_legacy or install_app 
        pm_command
            send_shell_command
                Pm.runInstall()
```
这个过程会把apk文件copy到data/local/tmp/目录下，然后向shell服务发送pm命令安装apk，最后调用`Pm.runInstall()`方法来安装apk。

## 6.1 pm.runInstall

frameworks\base\cmds\pm\src\com\android\commands\pm\Pm.java 

```java
    private int runInstall() throws RemoteException {
        final InstallParams params = makeInstallParams();
        // 1. 创建session
        final int sessionId = doCreateSession(params.sessionParams,
                params.installerPackageName, params.userId);

        try {
            final String inPath = nextArg();
            if (inPath == null && params.sessionParams.sizeBytes == 0) {
                System.err.println("Error: must either specify a package size or an APK file");
                return 1;
            }
            // 2. 写session
            if (doWriteSession(sessionId, inPath, params.sessionParams.sizeBytes, "base.apk",
                    false /*logSuccess*/) != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            // 3. 提交Session
            if (doCommitSession(sessionId, false /*logSuccess*/)
                    != PackageInstaller.STATUS_SUCCESS) {
                return 1;
            }
            System.out.println("Success");
            return 0;
        } finally {
            try {
                mInstaller.abandonSession(sessionId);
            } catch (Exception ignore) {
            }
        }
    }
```

从上面的代码来看，runInstall主要进行了三件事，即创建session、对session进行写操作，最后提交session。

### 6.1.1 doCreateSession

实际调用的是PackageInstallerService的createSession，这个过程主要是为APK安装做好准备工作，例如权限检查、目的临时文件的创建等， 最终创建出PackageInstallerSession对象。PackageInstallerSession可以看做是”安装APK”这个请求的封装，其中包含了处理这个请求需要的一些信息。 
实际上PackageInstallerSession不仅是分装请求的对象，其自身还是个服务端。

### 6.1.2 doWriteSession

通过PackageInstallerSession将/data/local/tmp的apk拷贝到终端目录内。

### 6.1.3 doCommitSession

doWriteSession结束后，如果没有出现任何错误，那么APK源文件已经copy到目的地址了，doCommitSession最终会调用到PMS.installStage来安装apk，调用流程如下：

`PackageInstallerSession.commit ==> commitLocked(); ==> PMS.installStage()`

PMS.installStage()会调用sendMessage将"INIT_COPY"发送给PackageHandler：

```java
    void installStage(String packageName, File stagedDir, String stagedCid,
            IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
            String installerPackageName, int installerUid, UserHandle user,
            Certificate[][] certificates) {
        if (DEBUG_EPHEMERAL) {
            if ((sessionParams.installFlags & PackageManager.INSTALL_EPHEMERAL) != 0) {
                Slog.d(TAG, "Ephemeral install of " + packageName);
            }
        }
        final VerificationInfo verificationInfo = new VerificationInfo(
                sessionParams.originatingUri, sessionParams.referrerUri,
                sessionParams.originatingUid, installerUid);

        final OriginInfo origin;
        if (stagedDir != null) {
            origin = OriginInfo.fromStagedFile(stagedDir);
        } else {
            origin = OriginInfo.fromStagedContainer(stagedCid);
        }

        final Message msg = mHandler.obtainMessage(INIT_COPY);
        final InstallParams params = new InstallParams(origin, null, observer,
                sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
                verificationInfo, user, sessionParams.abiOverride,
                sessionParams.grantedRuntimePermissions, certificates);
        params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
        msg.obj = params;

        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
                System.identityHashCode(msg.obj));
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(msg.obj));

        mHandler.sendMessage(msg);
    }
```

PackageHandler用于处理apk的安装请求等消息，后面分析。

# 七、ApplicationPackageManager

网络下载应用安装或者通过第三方应用安装，最终都会通过ApplicationPackageManager.installPackage来安装：

```java
public void installPackage(Uri packageURI, PackageInstallObserver observer,
            int flags, String installerPackageName) {
        installCommon(packageURI, observer, flags, installerPackageName, mContext.getUserId());
    }

private void installCommon(Uri packageURI,
        PackageInstallObserver observer, int flags, String installerPackageName,
        int userId) {
    if (!"file".equals(packageURI.getScheme())) {
        throw new UnsupportedOperationException("Only file:// URIs are supported");
    }

    final String originPath = packageURI.getPath();
    try {
        // PMS.installPackageAsUser
        mPM.installPackageAsUser(originPath, observer.getBinder(), flags, installerPackageName,
            userId);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

PMS.installPackageAsUser调用sendMessage将"INIT_COPY"发送给PackageHandler:

```java
    @Override
    public void installPackageAsUser(String originPath, IPackageInstallObserver2 observer,
            int installFlags, String installerPackageName, int userId) {

        ...

        final Message msg = mHandler.obtainMessage(INIT_COPY);
        final VerificationInfo verificationInfo = new VerificationInfo(
                null /*originatingUri*/, null /*referrer*/, -1 /*originatingUid*/, callingUid);
        final InstallParams params = new InstallParams(origin, null /*moveInfo*/, observer,
                installFlags, installerPackageName, null /*volumeUuid*/, verificationInfo, user,
                null /*packageAbiOverride*/, null /*grantedPermissions*/,
                null /*certificates*/);
        params.setTraceMethod("installAsUser").setTraceCookie(System.identityHashCode(params));
        msg.obj = params;
        mHandler.sendMessage(msg);

        ....
    }

```
PackageHandler用于处理apk的安装请求等消息，后面分析。

# 八、PackageHanlder

- PMS.installStage()会调用sendMessage将"INIT_COPY"发送给PackageHandler
- PMS.installPackageAsUser调用sendMessage将”INIT_COPY”发送给PackageHandler

## 8.1 INIT_COPY

PackageHandler用于处理apk的安装请求等消息，在PMS构造函数中有初始化。实际处理消息的函数为doHandleMessage，我们来看看INIT_COPY的处理流程：

```java
class PackageHandler extends Handler {

    ...

    void doHandleMessage(Message msg) {
        switch (msg.what) {
            case INIT_COPY: {
                //这里取出的其实就是InstallParams
                HandlerParams params = (HandlerParams) msg.obj;
                //idx为当前等待处理处理的安装请求的个数
                int idx = mPendingInstalls.size();
                ............

                //初始时，mBound的值为false
                if (!mBound) {
                    ............
                    // If this is the only one pending we might
                    // have to bind to the service again.
                    //连接安装服务
                    if (!connectToService()) {
                        ..................
                    } else {
                        // Once we bind to the service, the first
                        // pending request will be processed.
                        //绑定服务成功后，将新的请求加入到mPendingIntalls中，等待处理
                        mPendingInstalls.add(idx, params);
                    }
                } else {
                    //如果之前已经绑定过服务，同样将新的请求加入到mPendingIntalls中，等待处理
                    mPendingInstalls.add(idx, params);
                    // Already bound to the service. Just make
                    // sure we trigger off processing the first request.
                    if (idx == 0) {
                        //如果是第一个请求，则直接发送事件MCS_BOUND，触发处理流程
                        mHandler.sendEmptyMessage(MCS_BOUND);
                    }
                }
                break;
            }
        }
    }

    ...

}
```

INIT_COPY主要是将新的请求加入到mPendingIntalls中，等待MCS_BOUND阶段处理。

## 8.2 MCS_BOUND

INIT_COPY最后会发送MCS_BOUND消息触发接下来的流程，MCS_BOUND对应的处理流程同样定义于doHandleMessage中：

```java

void doHandleMessage(Message msg) {
    .......
    case MCS_BOUND: {
        ........
        if (msg.obj != null) {
            mContainerService = (IMediaContainerService) msg.obj;
            .......
        }
        if (mContainerService == null) {
            if (!mBound) {
                // Something seriously wrong since we are not bound and we are not
                // waiting for connection. Bail out.
                ............            
            } else {
                Slog.w(TAG, "Waiting to connect to media container service");
            }
        // 请求队列mPendingInstalls不为空
        } else if (mPendingInstalls.size() > 0) {
            HandlerParams params = mPendingInstalls.get(0);
            if (params != null) {
                ........
                //调用参数的startCopy函数处理安装请求
                if (params.startCopy()) {
                    ........
                    // Delete pending install
                    if (mPendingInstalls.size() > 0) {
                        mPendingInstalls.remove(0);
                    }
                    if (mPendingInstalls.size() == 0) {
                        if (mBound) {
                            ..........
                            removeMessages(MCS_UNBIND);
                            Message ubmsg = obtainMessage(MCS_UNBIND);
                            // Unbind after a little delay, to avoid
                            // continual thrashing.
                            sendMessageDelayed(ubmsg, 10000);
                        }
                    } else {
                        // There are more pending requests in queue.
                        // Just post MCS_BOUND message to trigger processing
                        // of next pending install.
                        ......
                        mHandler.sendEmptyMessage(MCS_BOUND);
                    }
                }
                .........
            }
        } else {
            // Should never happen ideally.
            Slog.w(TAG, "Empty queue");
        }
        break;
    }
.......
}
```

这一段代码比较好理解:
- 如果mPendingInstalls不为空，调用`InstallParams.startCopy`函数处理安装请求。
- 接着如果mPendingInstalls不为空，发送MCS_BOUND继续处理下一个，直到队列为空。
- 如果队列为空，则等待一段时间后，发送MCS_UNBIND消息断开与安装服务的绑定。


# 九、startCopy

/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

InstallParams继承HandlerParams，实际调用的是HandlerParams.startCopy:

```java

        final boolean startCopy() {
            boolean res;
            try {
                if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);

                if (++mRetries > MAX_RETRIES) {
                    Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                    mHandler.sendEmptyMessage(MCS_GIVE_UP);
                    handleServiceError();
                    return false;
                } else {
                    // 调用handleStartCopy()处理
                    handleStartCopy();
                    Slog.i(TAG, "Apk copy done");
                    res = true;
                }
            } catch (RemoteException e) {
                if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
                mHandler.sendEmptyMessage(MCS_RECONNECT);
                res = false;
            }
            // 
            handleReturnCode();
            return res;
        }
```

PMS将先后调用handleStartCopy和handleReturnCode来完成主要的工作。

## 9.1 handleStartCopy

handleStartCopy函数在HandleParams抽象类定义，在其子类InstallParams来实现，我们看看与实际安装相关的handleStartCopy函数：

```java
        public void handleStartCopy() throws RemoteException {
            int ret = PackageManager.INSTALL_SUCCEEDED;

            // 决定是安装在手机内还是sdcard中，设置对应标志位
            if (origin.staged) {
                if (origin.file != null) {
                    installFlags |= PackageManager.INSTALL_INTERNAL;
                    installFlags &= ~PackageManager.INSTALL_EXTERNAL;
                } else if (origin.cid != null) {
                    installFlags |= PackageManager.INSTALL_EXTERNAL;
                    installFlags &= ~PackageManager.INSTALL_INTERNAL;
                } else {
                    throw new IllegalStateException("Invalid stage location");
                }
            }

            ...

            // 检查APK的安装位置是否正确
            if (onInt && onSd) {
                // Check if both bits are set.
                Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else if (onSd && ephemeral) {
                Slog.w(TAG,  "Conflicting flags specified for installing ephemeral on external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else {
                ...
            }

            ...
            // createInstallArgs用于创建一个安装参数对象
            final InstallArgs args = createInstallArgs(this);
            
            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                ...
                    // 调用InstallArgs的copyApk函数
                    ret = args.copyApk(mContainerService, true);
                }
            }

            mRet = ret;
        }
```

InstallParams$handleStartCopy()主要功能是获取安装位置信息以及复制apk到指定位置。抽象类InstallArgs中的copyApk负责复制APK文件，具体实现在子类FileInstallArgs和SdInstallArgs里面。 


## 9.2 handleReturnCode

InstallParams$handleReturnCode()中，调用processPendingInstall方法处理安装：

```java
        void handleReturnCode() {
            // If mArgs is null, then MCS couldn't be reached. When it
            // reconnects, it will try again to install. At that point, this
            // will succeed.
            if (mArgs != null) {
                processPendingInstall(mArgs, mRet);
            }
        }
```

## 9.3 processPendingInstall

主要的安装流程都在这个方法里面: PMS.processPendingInstall

```
private void processPendingInstall(final InstallArgs args, final int currentStatus) {
    mHandler.post(new Runnable() {
        public void run() {
            mHandler.removeCallbacks(this);

            // Result object to be returned
            PackageInstalledInfo res = new PackageInstalledInfo();
            res.setReturnCode(currentStatus);
            res.uid = -1;
            res.pkg = null;
            res.removedInfo = null;
            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                //1、预安装，检查包状态，确保环境ok，如果环境不ok，那么会清理拷贝的文件
                args.doPreInstall(res.returnCode);

                synchronized (mInstallLock) {
                    //2、安装，调用installPackageTracedLI进行安装
                    installPackageTracedLI(args, res);
                }

                //3、安装收尾
                args.doPostInstall(res.returnCode, res.uid);
            }

            
            if (!doRestore) {
                .......
                //4、生成一个POST_INSTALL消息给PackageHanlder
                Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0);
                mHandler.sendMessage(msg);
            }
        }
    });
}
```

安装过程放在一个线程里面，处理流程是预安装-安装-安装收尾-发送 POST_INSTALL消息：
- **预安装**：检查当前安装包的状态以及确保SDCARD的挂载，并返回状态信息。在安装前确保安装环境的可靠。
- **安装**：对mInstallLock加锁，表明同时只能有一个安装包进行安装；然后调用installPackageTracedLI完成具体安装操作。
- **安装收尾**： 检查状态，如果安装不成功，删除掉相关目录文件。
- **发送POST_INSTALL消息**：该消息由PackageHandler接收。POST_INSTALL的主要工作其实还是通过广播、回调接口通知系统中的其它组件，有新的Pacakge安装或发生了改变。  

从上面我们可以知道，具体安装apk的函数是`PMS.installPackageTracedLI`。

# 十、installPackageTracedLI

PMS.installPackageTracedLI函数：

```java
    private void installPackageTracedLI(InstallArgs args, PackageInstalledInfo res) {
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installPackage");
            installPackageLI(args, res);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```

# 十一、installPackageLI

继续PMS.installPackageLI：

```java
    private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
        
        // PackageParser对象
        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setDisplayMetrics(mMetrics);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
        final PackageParser.Package pkg;
        try {
            if (DEBUG_INSTALL) Slog.i(TAG, "Start parsing apk: " + installerPackageName);
            // 1.开始解析我们的package
            pkg = pp.parsePackage(tmpPackageFile, parseFlags);
            if (DEBUG_INSTALL) Slog.i(TAG, "Parsing done for apk: " + installerPackageName);
        } catch (PackageParserException e) {
            res.setError("Failed parse during installPackageLI", e);
            return;
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        ...

        //2. 加载证书，获取签名信息
        try {
            // either use what we've been given or parse directly from the APK
            if (args.certificates != null) {
                try {
                    PackageParser.populateCertificates(pkg, args.certificates);
                } catch (PackageParserException e) {
                    PackageParser.collectCertificates(pkg, parseFlags);
                }
            } else {
                PackageParser.collectCertificates(pkg, parseFlags);
            }
        } catch (PackageParserException e) {
            res.setError("Failed collect during installPackageLI", e);
            return;
        }

        ...

        synchronized (mPackages) {
            // 3.检测packages是否存在
            if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
                    ...
                    replace = true;
                    
                } else if (mPackages.containsKey(pkgName)) {
                    ...
                    replace = true;
                    if (DEBUG_INSTALL) Slog.d(TAG, "Replace existing pacakge: " + pkgName);
                }
                ...           
            }
        }    
        ...

        try (PackageFreezer freezer = freezePackageForInstall(pkgName, installFlags,
                "installPackageLI")) {
            if (replace) {
                // 4.更新已经存在的packages
                replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                        installerPackageName, res);
            } else {
                // 5.安装新的packages
                installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                        args.user, installerPackageName, volumeUuid, res);
            }
        }
    ...
    }
```

这个函数过程比较长，主要做了几件事：

- PackageParser$parsePackage，主要是解析APK的AndroidManifest.xml，将每个标签对应的信息添加到Package的相关列表中，如将<application>下的<activity>信息添加到Package的activities列表等。 
- 加载apk证书，获取签名信息
- 检查目前安装的APK是否在系统中已存在: 
 - 已存在，则调用`replacePackageLIF`进行替换安装。 
 - 不存在，否则调用`installNewPackageLIF`进行安装。


## 11.1 replacePackageLIF

如果需要替换的是系统APP，则调用Settings$disableSystemPackageLPw来disable旧的APK；如果替换的是非系统APP，则调用deletePackageLI删除旧的APK。

因为这个过程实在太差，没有必要贴出来一一分析，我来简化一下flow，有兴趣的读者可以深入跟进：

```
replacePackageLIF
    replaceSystemPackageLIF  // 系统 pkg
        removePackageLI
        disableSystemPackageLPw
        clearAppDataLIF
        scanPackageTracedLI  //安装apk
            scanPackageLI
                scanPackageDirtyLI  
        updateSettingsLI
        updatePermissionsLPw
        mSettings.writeLPr();

    replaceNonSystemPackageLIF  // 非系统 pkg
        deletePackageLIF
        clearAppDataLIF
        clearAppProfilesLIF
        scanPackageTracedLI    // 安装apk
            scanPackageLI
                scanPackageDirtyLI  
        updateSettingsLI
        updatePermissionsLPw
        mSettings.writeLPr();

```

不管是更新系统还是非系统apk，都会先清除之前的packages信息，然后通过scanPackageTracedLI去安装apk，安装完后更新permissions和setting，最后通过writeLPr更新packages.xml。

关于scanPackageTracedLI和Settings.writeLPr();我有在上一篇blog讲过，可以回去看看。

## 11.2 installNewPackageLIF

PMS.installNewPackageLIF用于安装新的apk：

```java
private void installNewPackageLIF(PackageParser.Package pkg, final int policyFlags,
            int scanFlags, UserHandle user, String installerPackageName, String volumeUuid,
            PackageInstalledInfo res) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installNewPackage");

        // Remember this for later, in case we need to rollback this install
        String pkgName = pkg.packageName;

        if (DEBUG_INSTALL) Slog.d(TAG, "installNewPackageLI: " + pkg);

        // package已经存在
        synchronized(mPackages) {
            if (mSettings.mRenamedPackages.containsKey(pkgName)) {
                // A package with the same name is already installed, though
                // it has been renamed to an older name.  The package we
                // are trying to install should be installed as an update to
                // the existing one, but that has not been requested, so bail.
                res.setError(INSTALL_FAILED_ALREADY_EXISTS, "Attempt to re-install " + pkgName
                        + " without first uninstalling package running as "
                        + mSettings.mRenamedPackages.get(pkgName));
                return;
            }
            if (mPackages.containsKey(pkgName)) {
                // Don't allow installation over an existing package with the same name.
                res.setError(INSTALL_FAILED_ALREADY_EXISTS, "Attempt to re-install " + pkgName
                        + " without first uninstalling.");
                return;
            }
        }

        try {
            // 1. 安装apk
            PackageParser.Package newPackage = scanPackageTracedLI(pkg, policyFlags, scanFlags,
                    System.currentTimeMillis(), user);

            // 2. 更新setting
            updateSettingsLI(newPackage, installerPackageName, null, res, user);

            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                prepareAppDataAfterInstallLIF(newPackage);

            } else {
                // Remove package from internal structures, but keep around any
                // data that might have already existed
                deletePackageLIF(pkgName, UserHandle.ALL, false, null,
                        PackageManager.DELETE_KEEP_DATA, res.removedInfo, true, null);
            }
        } catch (PackageManagerException e) {
            res.setError("Package couldn't be installed in " + pkg.codePath, e);
        }

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```

installNewPackageLIF会调用scanPackageTracedLI去安装apk，最终会调用scanPackageLI->scanPackageDirtyLI实际去安装apk。

由于之前有描述过，便不再叙述。















