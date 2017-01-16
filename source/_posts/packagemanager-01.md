---
title: PackageManagerService启动流程
categories:
  - Android进阶
tags:
  - PMS
  - PackageManager
  - PackageManagerService
date: 2017-01-10 22:04:59
---

本文基于Android 7.0介绍PackageManager。

<!-- more -->

相关源码:
```
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
frameworks/base/services/core/java/com/android/server/pm/PackageInstallerService.java
frameworks/base/services/core/java/com/android/server/pm/Settings.java
frameworks/base/services/core/java/com/android/server/pm/Installer.java
frameworks/base/services/core/java/com/android/server/SystemConfig

frameworks/base/core/java/android/content/pm/PackageManager.java
frameworks/base/core/java/android/content/pm/IPackageManager.aidl
frameworks/base/core/java/android/content/pm/PackageParser.java

frameworks/base/core/java/com/android/internal/os/InstallerConnection.java
frameworks/base/cmds/pm/src/com/android/commands/pm/Pm.java

```





# PMS启动入口 - system_server

Android的所有Java服务都是通过`system_server`进程启动的，并且驻留在`system_server`进程中。SystemServer进程在启动时，通过创建一个`ServerThread`线程来启动所有服务，现在先来看看Android服务中`PackageManagerService`服务启动过程。

> /frameworks/base/services/java/com/android/server/SystemServer.java

## startBootstrapServices()

system_server的**`startBootstrapServices()`**函数会启动一些引导服务，该方法所创建的服务：
- ActivityManagerService, 
- PowerManagerService, 
- LightsService, 
- DisplayManagerService， 
- `PackageManagerService`，
- UserManagerService， 
- SensorService服务。

其中我们需要的PackageManagerService就在这里启动，我们来看看startBootstrapServices()是如何启动PMS的：

```java
private void startBootstrapServices() {

        //启动installer服务
        Installer installer = mSystemServiceManager.startService(Installer.class);

        // We need the default display before we can initialize the package manager.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
        
        //处于加密状态则仅仅解析核心应用
        // Only run "core" apps if we're encrypting the device.
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        // 创建PMS对象 - 启动入口
        traceBeginAndSlog("StartPackageManagerService");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        // 是否首次启动
        mFirstBoot = mPackageManagerService.isFirstBoot();
        
        // 获取PackageManager
        mPackageManager = mSystemContext.getPackageManager();
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

}
```

## startOtherServices()

另外，system_server的`startOtherServices()`方法会启动其他服务，这个函数也会对PMS作一些操作：

```java
private void startOtherServices() {
    ...
    //启动MountService，后续PackageManager会需要使用
    mSystemServiceManager.startService(MOUNT_SERVICE_CLASS);
    //
    mPackageManagerService.performBootDexOpt();
    ...  

    //
    mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
    ...
    //
    mPackageManagerService.systemReady();
}
```

整个system_server进程启动过程，涉及PMS服务的主要几个动作如下，接下来分别讲解每个过程

- PMS.main()
- PMS.performBootDexOpt()
- PMS.systemReady()


# PMS.main()

PackageManagerService.main过程主要是创建PMS服务，并注册到ServiceManager大管家：

```java
public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        // 创建PMS对象
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        // Disable any carrier apps. We do this very early in boot to prevent the apps from being
        // disabled after already being started.
        CarrierAppUtils.disableCarrierAppsUntilPrivileged(context.getOpPackageName(), m,
                UserHandle.USER_SYSTEM);
        
        // 添加到ServiceManager
        ServiceManager.addService("package", m);
        return m;
}
```

## 创建PMS对象

> new PackageManagerService(context, installer, factoryTest, onlyCore);

创建PMS对象的过程，就是执行PMS的构造函数，PMS构造函数比较长，我们把这个过程分成几个阶段：

- BOOT_PROGRESS_PMS_START,
- BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
- BOOT_PROGRESS_PMS_DATA_SCAN_START,
- BOOT_PROGRESS_PMS_SCAN_END,
- BOOT_PROGRESS_PMS_READY,

PMS构造函数里面，在每个阶段开始的时候，都会往`Eventlog`里面打Tag，比如像这样：

```java
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START, SystemClock.uptimeMillis());
```

接下来分别说说这几个阶段。

### PMS_START

为了方便分析，把PMS_START分成两个阶段，阶段一：

```java
        // 输出event log
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());

        /** M: Mtprof tool @{ */
        //mMTPROFDisable = "1".equals(SystemProperties.get("ro.mtprof.disable"));
        mMTPROFDisable = false;
        addBootEvent("Android:PackageManagerService_Start");
        /** @} */

        if (mSdkVersion <= 0) {
            Slog.w(TAG, "**** ro.build.version.sdk not set!");
        }

        mContext = context;
        mFactoryTest = factoryTest;
        mOnlyCore = onlyCore;

        // DisplayMetrics是一个描述界面显示，尺寸，分辨率，密度的类。
        mMetrics = new DisplayMetrics();

        // Settings是Android的全局管理者，用于协助PMS保存所有的安装包信息
        mSettings = new Settings(mPackages);
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);

        // 获取debug.separate_processes属性
        // 如果设置了这个属性，那么会强制应用程序组件在自己的进程中运行。
        // 一般情况下不会设置这个属性
        String separateProcesses = SystemProperties.get("debug.separate_processes");
        if (separateProcesses != null && separateProcesses.length() > 0) {
            // 所有process都设置这个属性
            if ("*".equals(separateProcesses)) {
                mDefParseFlags = PackageParser.PARSE_IGNORE_PROCESSES;
                mSeparateProcesses = null;
                Slog.w(TAG, "Running with debug.separate_processes: * (ALL)");
            }
            // 个别的process设置这个属性 
            else {
                mDefParseFlags = 0;
                mSeparateProcesses = separateProcesses.split(",");
                Slog.w(TAG, "Running with debug.separate_processes: "
                        + separateProcesses);
            }
        } else { // 不设置这个属性,一般情况下会走这
            mDefParseFlags = 0;
            mSeparateProcesses = null;
        }

        // 保存Installer对象
        mInstaller = installer;

        // //用于dex优化
        mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
                "*dexopt*");
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());

        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());

        // 获取默认的显示信息，保存到mMetrics
        getDefaultDisplayMetrics(context, mMetrics);

        // 获取系统配置信息
        SystemConfig systemConfig = SystemConfig.getInstance();
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
```

说明下**debug.separate_processes**这个属性：
这个属性你可以使用强制应用程序组件在自己的进程中运行。 一般不会用到，有两种方法可以使用这个：

```
// 所有的进程都会受到影响
setprop debug.separate_processes 
// 指定进程受影响
setprop debug.separate_processes“com.google.process.content, com.google.android.samples”  
```

**总结阶段一主要工作如下：**

- 构造`DisplayMetrics`类：描述界面显示，尺寸，分辨率，密度。构造完后并获取默认的信息保存到变量mMetrics中。
- 构造`Settings`类：这个是Android的全局管理者，用于协助PMS保存所有的安装包信息
- 保存`Installer`对象
- 获取系统配置信息：SystemConfig构造函数中会通过`readPermissions()`解析指定目录下的所有xml文件,然后把这些信息保存到systemConfig中，解析XML是通过XmlPullParser解析的，涉及的目录有如下：
 + /system/etc/sysconfig
 + /system/etc/permissions
 + /oem/etc/sysconfig
 + /oem/etc/permissions

接下来，我们看后半部，后续代码都是同时持有同步锁`mPackages`和`mInstallLock`的过程中执行的，我们把这个定义为**阶段二**：

```java
        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {

            //创建名为“PackageManager”的handler线程
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            
            // 建立PackageHandler消息循环，用于处理外部的安装请求等消息
            // 比如如adb install、packageinstaller安装APK时
            mHandler = new PackageHandler(mHandlerThread.getLooper());
            mProcessLoggingHandler = new ProcessLoggingHandler();
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);

            // 创建各种目录
            File dataDir = Environment.getDataDirectory();
            mAppInstallDir = new File(dataDir, "app");
            mAppLib32InstallDir = new File(dataDir, "app-lib");
            mEphemeralInstallDir = new File(dataDir, "app-ephemeral");
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");

            // 创建用户管理服务
            sUserManager = new UserManagerService(context, this, mPackages);

            // Propagate permission configuration in to package manager.
            ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                    = systemConfig.getPermissions();
            for (int i=0; i<permConfig.size(); i++) {
                SystemConfig.PermissionEntry perm = permConfig.valueAt(i);
                BasePermission bp = mSettings.mPermissions.get(perm.name);
                if (bp == null) {
                    bp = new BasePermission(perm.name, "android", BasePermission.TYPE_BUILTIN);
                    mSettings.mPermissions.put(perm.name, bp);
                }
                if (perm.gids != null) {
                    bp.setGids(perm.gids, perm.perUser);
                }
            }

            // 获取共享库
            ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
            for (int i=0; i<libConfig.size(); i++) {
                mSharedLibraries.put(libConfig.keyAt(i),
                        new SharedLibraryEntry(libConfig.valueAt(i), null));
            }

            mFoundPolicyFile = SELinuxMMAC.readInstallPolicy();

            // 解析packages.xml和packages-backup.xml
            mRestoredSettings = mSettings.readLPw(sUserManager.getUsers(false));

            ...
            }
```

我们再来总结阶段二都做了什么：

- 创建名为“PackageManager”的handler线程，建立PackageHandler消息循环，用于处理外部的安装请求等消息
- 创建data下的各种目录，比如data/app, data/app-private等。
- 创建用户管理服务
- 扫描解析packages.xml和packages-backup.xml
