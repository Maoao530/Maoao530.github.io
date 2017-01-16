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





# system_server启动PMS

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


# PMS.main入口

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

# 创建PMS服务 - 细节

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

## PMS_START

BOOT_PROGRESS_PMS_START阶段：

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

我们再来总结PMS_START阶段都做了什么：

**总结阶段一主要工作如下：**
- 构造DisplayMetrics类：描述界面显示，尺寸，分辨率，密度。构造完后并获取默认的信息保存到变量mMetrics中。
- 构造Settings类：这个是Android的全局管理者，用于协助PMS保存所有的安装包信息
- 保存Installer对象
- 获取系统配置信息：SystemConfig构造函数中会通过`readPermissions()`解析指定目录下的所有xml文件,然后把这些信息保存到systemConfig中，涉及的目录有如下：
 + /system/etc/sysconfig
 + /system/etc/permissions
 + /oem/etc/sysconfig
 + /oem/etc/permissions
- 创建名为PackageManager的handler线程，建立PackageHandler消息循环，用于处理外部的安装请求等消息
- 创建data下的各种目录，比如data/app, data/app-private等。
- 创建用户管理服务UserManagerService
- 把systemConfig关于xml中的<library>标签所指的动态库保存到mSharedLibraries
- Settings.readLPw扫描解析packages.xml和packages-backup.xml


补充说明下**debug.separate_processes**这个属性：
这个属性你可以使用强制应用程序组件在自己的进程中运行，有两种方法可以使用这个：

```
// 所有的进程都会受到影响
setprop debug.separate_processes 
// 指定进程受影响
setprop debug.separate_processes“com.google.process.content, com.google.android.samples”  
```

这个属性一般不会用到。


## PMS_SYSTEM_SCAN_START

接下来是BOOT_PROGRESS_PMS_SYSTEM_SCAN_START阶段：

```java
long startTime = SystemClock.uptimeMillis();
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
        startTime);

final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;
//该集合中存放的是已经优化或者不需要优先的文件
final ArraySet<String> alreadyDexOpted = new ArraySet<String>();

final String bootClassPath = System.getenv("BOOTCLASSPATH");
final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");

//将环境变量BOOTCLASSPATH所执行的文件加入alreadyDexOpted
if (bootClassPath != null) {
    String[] bootClassPathElements = splitString(bootClassPath, ':');
    for (String element : bootClassPathElements) {
        alreadyDexOpted.add(element);
    }
}

//将环境变量SYSTEMSERVERCLASSPATH所执行的文件加入alreadyDexOpted
if (systemServerClassPath != null) {
    String[] systemServerClassPathElements = splitString(systemServerClassPath, ':');
    for (String element : systemServerClassPathElements) {
        alreadyDexOpted.add(element);
    }
}
...

//此处共享库是由SystemConfig实例化过程赋值的
if (mSharedLibraries.size() > 0) {
    for (String dexCodeInstructionSet : dexCodeInstructionSets) {
        for (SharedLibraryEntry libEntry : mSharedLibraries.values()) {
            final String lib = libEntry.path;
            ...
            int dexoptNeeded = DexFile.getDexOptNeeded(lib, dexCodeInstructionSet,
                    "speed", false);
            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                alreadyDexOpted.add(lib);
                //执行dexopt操作
                mInstaller.dexopt(lib, Process.SYSTEM_UID, dexCodeInstructionSet,
                        dexoptNeeded, DEXOPT_PUBLIC /*dexFlags*/);
            }
        }
    }
}

//此处frameworkDir目录为/system/framework
File frameworkDir = new File(Environment.getRootDirectory(), "framework");

//添加以下两个文件添加到已优化集合
alreadyDexOpted.add(frameworkDir.getPath() + "/framework-res.apk");
alreadyDexOpted.add(frameworkDir.getPath() + "/core-libart.jar");

String[] frameworkFiles = frameworkDir.list();
if (frameworkFiles != null) {
    for (String dexCodeInstructionSet : dexCodeInstructionSets) {
        for (int i=0; i<frameworkFiles.length; i++) {
            File libPath = new File(frameworkDir, frameworkFiles[i]);
            String path = libPath.getPath();
            //跳过已优化集合中的文件
            if (alreadyDexOpted.contains(path)) {
                continue;
            }
            //跳过后缀不为apk和jar的文件
            if (!path.endsWith(".apk") && !path.endsWith(".jar")) {
                continue;
            }

            int dexoptNeeded = DexFile.getDexOptNeeded(path, dexCodeInstructionSet,
                    "speed", false);
            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                //执行dexopt操作【见小节2.2.1】
                mInstaller.dexopt(path, Process.SYSTEM_UID, dexCodeInstructionSet,
                        dexoptNeeded, DEXOPT_PUBLIC /*dexFlags*/);
            }

        }
    }
}

final VersionInfo ver = mSettings.getInternalVersion();
mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);
mPromoteSystemApps = mIsUpgrade && ver.sdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1;

if (mPromoteSystemApps) {
    Iterator<PackageSetting> pkgSettingIter = mSettings.mPackages.values().iterator();
    while (pkgSettingIter.hasNext()) {
        PackageSetting ps = pkgSettingIter.next();
        if (isSystemApp(ps)) {
            mExistingSystemPackages.add(ps.name);
        }
    }
}

//收集供应商包名：/vendor/overlay
File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
scanDirLI(vendorOverlayDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

//收集包名：/system/framework
scanDirLI(frameworkDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED,
        scanFlags | SCAN_NO_DEX, 0);

//收集私有的系统包名：/system/priv-app
final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

//收集一般地系统包名：/system/app
final File systemAppDir = new File(Environment.getRootDirectory(), "app");
scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

//收集私有供应商包名：/vendor/priv-app
final File privilegedVendorAppDir = new File(Environment.getVendorDirectory(), "priv-app");
scanDirLI(privilegedVendorAppDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

//收集所有的供应商包名：/vendor/app
File vendorAppDir = new File(Environment.getVendorDirectory(), "app");
vendorAppDir = vendorAppDir.getCanonicalFile();
scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

//收集所有OEM包名：/oem/app
final File oemAppDir = new File(Environment.getOemDirectory(), "app");
scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

//移除文件
mInstaller.moveFiles();

//删除不在存在的系统包
final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();
if (!mOnlyCore) {
    Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
    while (psit.hasNext()) {
        PackageSetting ps = psit.next();

        if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
            continue;
        }

        final PackageParser.Package scannedPkg = mPackages.get(ps.name);
        if (scannedPkg != null) {
            if (mSettings.isDisabledSystemPackageLPr(ps.name)) {
                removePackageLI(ps, true);
                mExpectingBetter.put(ps.name, ps.codePath);
            }
            continue;
        }

        if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
            psit.remove();
            removeDataDirsLI(null, ps.name);
        } else {
            final PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(ps.name);
            if (disabledPs.codePath == null || !disabledPs.codePath.exists()) {
                possiblyDeletedUpdatedSystemApps.add(ps.name);
            }
        }
    }
}

//清理所有安装不完整的包
ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
for(int i = 0; i < deletePkgsList.size(); i++) {
    cleanupInstallFailedPackage(deletePkgsList.get(i));
}
//删除临时文件
deleteTempPackageFiles();

//移除不相干包中的所有共享userID
mSettings.pruneSharedUsersLPw();
```

PMS_SYSTEM_SCAN_START阶段主要做了如下工作：

- 首先将BOOTCLASSPATH，SYSTEMSERVERCLASSPATH这两个环境变量下的路径加入到不需要dex优化集合alreadyDexOpted中
 + SYSTEMSERVERCLASSPATH：主要包括/system/framework目录下services.jar，ethernet-service.jar，wifi-service.jar这3个文件。
 + BOOTCLASSPATH：该环境变量内容较多，不同ROM可能有所不同，常见内容包含/system/framework目录下的framework.jar，ext.jar，core-libart.jar，telephony-common.jar，ims-common.jar，core-junit.jar等文件。
- 获取共享库mSharedLibraries，判断是否需要dex优化，如果需要则进行dex优化，并加入到alreadyDexOpted列表中
- 添加framework-res.apk、core-libart.jar两个文件添加到已优化集合alreadyDexOpted中
- 将framework目录下，其他的apk或者jar，进行dex优化并加入已优化集合alreadyDexOpted中
- scanDirLI(): 扫描指定目录下的apk文件，最终调用PackageParser.parseBaseApk来完成AndroidManifest.xml文件的解析，生成Application, activity,service,broadcast, provider等信息
- 删除系统不存在的包 removePackageLI
- 清理安装失败的包 cleanupInstallFailedPackage
- 删除临时文件 deleteTempPackageFiles
- 移除不相干包中的所有共享userID

## PMS_DATA_SCAN_START

BOOT_PROGRESS_PMS_DATA_SCAN_START阶段：

```java
if (!mOnlyCore) {
      EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
              SystemClock.uptimeMillis());
      scanDirLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
      scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
              scanFlags | SCAN_REQUIRE_KNOWN, 0);
      /**
       * Remove disable package settings for any updated system
       * apps that were removed via an OTA. If they're not a
       * previously-updated app, remove them completely.
       * Otherwise, just revoke their system-level permissions.
       */
      for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
          PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
          mSettings.removeDisabledSystemPackageLPw(deletedAppName);
          String msg;
          if (deletedPkg == null) {
              msg = "Updated system package " + deletedAppName
                      + " no longer exists; wiping its data";
              removeDataDirsLI(null, deletedAppName);
          } else {
              msg = "Updated system app + " + deletedAppName
                      + " no longer present; removing system privileges for "
                      + deletedAppName;
              deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;
              PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
              deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;
          }
          logCriticalInfo(Log.WARN, msg);
      }
      /**
       * Make sure all system apps that we expected to appear on
       * the userdata partition actually showed up. If they never
       * appeared, crawl back and revive the system version.
       */
      for (int i = 0; i < mExpectingBetter.size(); i++) {
          final String packageName = mExpectingBetter.keyAt(i);
          if (!mPackages.containsKey(packageName)) {
              final File scanFile = mExpectingBetter.valueAt(i);
              logCriticalInfo(Log.WARN, "Expected better " + packageName
                      + " but never showed up; reverting to system");
              final int reparseFlags;
              if (FileUtils.contains(privilegedAppDir, scanFile)) {
                  reparseFlags = PackageParser.PARSE_IS_SYSTEM
                          | PackageParser.PARSE_IS_SYSTEM_DIR
                          | PackageParser.PARSE_IS_PRIVILEGED;
              } else if (FileUtils.contains(systemAppDir, scanFile)) {
                  reparseFlags = PackageParser.PARSE_IS_SYSTEM
                          | PackageParser.PARSE_IS_SYSTEM_DIR;
              } else if (FileUtils.contains(vendorAppDir, scanFile)) {
                  reparseFlags = PackageParser.PARSE_IS_SYSTEM
                          | PackageParser.PARSE_IS_SYSTEM_DIR;
              } else if (FileUtils.contains(oemAppDir, scanFile)) {
                  reparseFlags = PackageParser.PARSE_IS_SYSTEM
                          | PackageParser.PARSE_IS_SYSTEM_DIR;
              } else {
                  Slog.e(TAG, "Ignoring unexpected fallback path " + scanFile);
                  continue;
              }
              mSettings.enableSystemPackageLPw(packageName);
              try {
                  scanPackageLI(scanFile, reparseFlags, scanFlags, 0, null);
              } catch (PackageManagerException e) {
                  Slog.e(TAG, "Failed to parse original system package: "
                          + e.getMessage());
              }
          }
      }
  }
  mExpectingBetter.clear();
  // Now that we know all of the shared libraries, update all clients to have
  // the correct library paths.
  updateAllSharedLibrariesLPw();
  for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
      // NOTE: We ignore potential failures here during a system scan (like
      // the rest of the commands above) because there's precious little we
      // can do about it. A settings error is reported, though.
      adjustCpuAbisForSharedUserLPw(setting.packages, null /* scanned package */,
              false /* force dexopt */, false /* defer dexopt */);
  }
  // Now that we know all the packages we are keeping,
  // read and update their last usage times.
  mPackageUsage.readLP();
```

- 当mOnlyCore = false时，则scanDirLI()还会收集如下目录中的apk
 + /data/app
 + /data/app-private


## PMS_SCAN_END

BOOT_PROGRESS_PMS_SCAN_END阶段：

```java
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());
            Slog.i(TAG, "Time to scan packages: "
                    + ((SystemClock.uptimeMillis()-startTime)/1000f)
                    + " seconds");

            // 当sdk版本不一致时，需要更新权限
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
            ver.sdkVersion = mSdkVersion;

            //当这是ota后的首次启动，正常启动则需要清除目录的缓存代码
            if (!onlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(this, user.id);
                    applyFactoryDefaultBrowserLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            //当权限和其他默认项都完成更新，则清理相关信息
            final int storageFlags;
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
            reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL, UserHandle.USER_SYSTEM,
                    storageFlags);

            //当这是ota后的首次启动，正常启动则需要清除目录的缓存代码
            if (mIsUpgrade && !onlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        // No apps are running this early, so no need to freeze
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            checkDefaultBrowser();

            //当权限和其他默认项都完成更新，则清理相关信息
            mExistingSystemPackages.clear();
            mPromoteSystemApps = false;

            // All the changes are done during package scanning.
            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            //信息写回packages.xml文件
            mSettings.writeLPr();
```

- 当sdk版本不一致时，需要更新权限
- 当这是ota后的首次启动，正常启动则需要清除目录的缓存代码
- 当权限和其他默认项都完成更新，则清理相关信息
- 信息写回packages.xml文件


## PMS_READY

BOOT_PROGRESS_PMS_READY阶段：

```java
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());

            if (!mOnlyCore) {
                mRequiredVerifierPackage = getRequiredButNotReallyRequiredVerifierLPr();
                mRequiredInstallerPackage = getRequiredInstallerLPr();
                mRequiredUninstallerPackage = getRequiredUninstallerLPr();
                mIntentFilterVerifierComponent = getIntentFilterVerifierComponentNameLPr();
                mIntentFilterVerifier = new IntentVerifierProxy(mContext,
                        mIntentFilterVerifierComponent);
                mServicesSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SERVICES);
                mSharedSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SHARED);
            } else {
                mRequiredVerifierPackage = null;
                mRequiredInstallerPackage = null;
                mRequiredUninstallerPackage = null;
                mIntentFilterVerifierComponent = null;
                mIntentFilterVerifier = null;
                mServicesSystemSharedLibraryPackageName = null;
                mSharedSystemSharedLibraryPackageName = null;
            }

            mInstallerService = new PackageInstallerService(context, this);

            final ComponentName ephemeralResolverComponent = getEphemeralResolverLPr();
            final ComponentName ephemeralInstallerComponent = getEphemeralInstallerLPr();
            // both the installer and resolver must be present to enable ephemeral
            if (ephemeralInstallerComponent != null && ephemeralResolverComponent != null) {
                if (DEBUG_EPHEMERAL) {
                    Slog.i(TAG, "Ephemeral activated; resolver: " + ephemeralResolverComponent
                            + " installer:" + ephemeralInstallerComponent);
                }
                mEphemeralResolverComponent = ephemeralResolverComponent;
                mEphemeralInstallerComponent = ephemeralInstallerComponent;
                setUpEphemeralInstallerActivityLP(mEphemeralInstallerComponent);
                mEphemeralResolverConnection =
                        new EphemeralResolverConnection(mContext, mEphemeralResolverComponent);
            } else {
                if (DEBUG_EPHEMERAL) {
                    final String missingComponent =
                            (ephemeralResolverComponent == null)
                            ? (ephemeralInstallerComponent == null)
                                    ? "resolver and installer"
                                    : "resolver"
                            : "installer";
                    Slog.i(TAG, "Ephemeral deactivated; missing " + missingComponent);
                }
                mEphemeralResolverComponent = null;
                mEphemeralInstallerComponent = null;
                mEphemeralResolverConnection = null;
            }

            mEphemeralApplicationRegistry = new EphemeralApplicationRegistry(this);
        } // synchronized (mPackages)
        } // synchronized (mInstallLock)

        // Now after opening every single application zip, make sure they
        // are all flushed.  Not really needed, but keeps things nice and
        // tidy.
        Runtime.getRuntime().gc();

        // The initial scanning above does many calls into installd while
        // holding the mPackages lock, but we're mostly interested in yelling
        // once we have a booted system.
        mInstaller.setWarnIfHeld(mPackages);

        // Expose private service for system components to use.
        LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
```

- 初始化PackageInstallerService
- GC回收下内存

# 创建PMS服务 - 总结

PMS初始化过程，分为5个阶段：

**1. PMS_START阶段：**
 - 创建Settings对象；
 - 将6类shareUserId到mSettings；
 - 初始化SystemConfig；
 - 创建名为“PackageManager”的handler线程mHandlerThread;
 - 创建UserManagerService多用户管理服务；
 - 通过解析4大目录中的xmL文件构造共享mSharedLibraries；

**2. PMS_SYSTEM_SCAN_START阶段：**
 - mSharedLibraries共享库中的文件执行dexopt操作；
 - system/framework目录中满足条件的apk或jar文件执行dexopt操作；
 - 扫描系统apk;

**3. PMS_DATA_SCAN_START阶段：**
 - 扫描/data/app目录下的apk;
 - 扫描/data/app-private目录下的apk;

**4. PMS_SCAN_END阶段：**
 - 将上述信息写回/data/system/packages.xml;

**5. PMS_READY阶段：**
 - 创建服务PackageInstallerService；


 # 开机时间分析

adb shell cat /proc/bootprof/
```
 C:\Users\shun>adb shell cat /proc/bootprof
----------------------------------------
0           BOOT PROF (unit:msec)
----------------------------------------
      1655        : preloader
      1001        : lk
       493        : lk->Kernel
----------------------------------------
      5156.702307 : Kernel_init_done
      5171.629538 : SElinux start.
     10739.699692 : SElinux end.
     11496.788538 : INIT: on init start
     11878.325615 : INIT:eMMC:Mount_START
     12777.653384 : INIT:eMMC:Mount_END
     12780.874230 : INIT:PROTECT:Mount_START
     12938.042615 : INIT:PROTECT:Mount_END
     14029.370538 : INIT: eng build setting
     15215.342538 : BOOT_Animation:START
     16618.475076 : Zygote:Preload Start
     20691.658230 : Zygote:Preload 2775 classes in 4062ms
     23061.424153 : Zygote:Preload 274 obtain resources in 2334ms
     23110.519076 : Zygote:Preload 31 resources in 47ms
     23240.816000 : Zygote:Preload End
     23720.832000 : Android:SysServerInit_START
     24448.175153 : Android:PackageManagerService_Start
     24747.363153 : Android:PMS_scan_START
     24817.216000 : Android:PMS_scan_done:/custom/framework
     24947.104384 : Android:PMS_scan_done:/system/framework
     25131.265384 : Android:PMS_scan_done:/system/priv-app
     25533.440461 : Android:PMS_scan_done:/system/app
     25540.237769 : Android:PMS_scan_done:/system/vendor/app
     25542.379538 : Android:PMS_scan_done:/system/vendor/operator/app
     25544.285615 : Android:PMS_scan_done:/custom/app
     25551.297076 : Android:PMS_scan_data_start
     25967.971076 : Android:PMS_scan_data_done:/data/app
     25969.811230 : Android:PMS_scan_data_done:/data/app-private
     25971.862692 : Android:PMS_scan_END
     26224.410076 : Android:PMS_READY
     30108.635538 : AP_Init:[service]:[com.mediatek.security]:[com.mediatek.security/.service.PermControlService]:pid:738
     30548.305692 : AP_Init:[broadcast]:[com.android.settings]:[com.android.settings/.widget.SettingsAppWidgetProvider]:pid:769
     31341.380923 : AP_Init:[broadcast]:[com.mediatek.engineermode]:[com.mediatek.engineermode/.wifi.WifiStateReceiver]:pid:806
     31563.917923 : AP_Init:[broadcast]:[com.tvguo.app]:[com.tvguo.app/.content.TvguoStateReceiver]:pid:829
     31708.206000 : AP_Init:[service]:[com.android.systemui]:[com.android.systemui/.ImageWallpaper]:pid:847
     31796.008076 : AP_Init:[service]:[com.android.inputmethod.latin]:[com.android.inputmethod.latin/.LatinIME]:pid:860
     31987.434923 : AP_Init:[added application]:[com.mediatek.dongle]:[com.mediatek.dongle]:pid:878:(PersistAP)
     32051.271692 : AP_Init:[added application]:[com.mediatek.bluetooth]:[com.mediatek.bluetooth]:pid:891:(PersistAP)
     32142.389846 : AP_Init:[activity]:[com.android.launcher3]:[com.android.launcher3/.Launcher]:pid:906
     32170.530846 : Android:SysServerInit_END
     32309.788000 : AP_Init:[service]:[com.android.printspooler]:[com.android.printspooler/.PrintSpoolerService]:pid:923
     33967.604076 : AP_Init:[broadcast]:[com.android.contacts]:[com.android.contacts/com.mediatek.contacts.simcontact.BootCmpReceiver]:pid:972
     34752.970538 : AP_Init:[content provider]:[android.process.acore]:[com.android.providers.contacts/.ContactsProvider2]:pid:1028
     35486.120000 : BOOT_Animation:END
----------------------------------------
```

我们可以从上面的信息看到PMS在开机的时候做的动作和时间分布：（因为手上只有kk的平台，所以信息不太对应）

```
     24747.363153 : Android:PMS_scan_START
     24817.216000 : Android:PMS_scan_done:/custom/framework
     24947.104384 : Android:PMS_scan_done:/system/framework
     25131.265384 : Android:PMS_scan_done:/system/priv-app
     25533.440461 : Android:PMS_scan_done:/system/app
     25540.237769 : Android:PMS_scan_done:/system/vendor/app
     25542.379538 : Android:PMS_scan_done:/system/vendor/operator/app
     25544.285615 : Android:PMS_scan_done:/custom/app
     25551.297076 : Android:PMS_scan_data_start
     25967.971076 : Android:PMS_scan_data_done:/data/app
     25969.811230 : Android:PMS_scan_data_done:/data/app-private
     25971.862692 : Android:PMS_scan_END
     26224.410076 : Android:PMS_READY
```

一般app装的越多，那么开机时间就会越长。