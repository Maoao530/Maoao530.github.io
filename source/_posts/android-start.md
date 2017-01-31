---
title: Android系统启动流程
categories:
  - Android进阶
tags:
  - Android启动
date: 2017-1-6 21:29:35
top: 10
---

本文介绍android的启动流程。

<!-- more -->

系统启动架构图：

![android-boot-up.png](/img/archives/android-boot-up.png)

# 一、Loader层

**1. Boot ROM: **
上电后，BootRom会被激活，`引导芯片代码`开始从预定义的地方（固化在ROM）开始执行，然后加载引导程序到`RAM`。

**2. Boot Loader引导程序**
Boot Loader是启动Android系统之前的引导程序，引导程序是OEM厂商或者运营商加锁和限制的地方，它是针对特定的主板与芯片的。OEM厂商要么使用很受欢迎的引导程序比如`redboot`、`uboot`、`ARMboot`等或者开发自己的引导程序，它不是Android操作系统的一部分。
Boot Loader主要作用是检查RAM，初始化硬件参数等功能。

**3 Preloader: **
（1）Preloader是MTK平台独有的防止芯片被Hack的一个loader，MTK平台的bootrom会先加载preloader到SRAM中，preloader会先去初始化一些HW组件，比如通信端口(USB/Uart)，外部存储设备(Emmc or Nand)，内存设备(DRAM Calibration)等，最后会Load LK到DRAM中并且run LK(U-boot)。
（2）LK会从外部存储设备load boot image,包括Linux kernel和Ramdisk到DRAM中.最后LK会跳到 Linux Kernel里去执行start kernel.
（3）linux kernel会先完成一些初始化动作，mount 根文件系统和启动第一个用户进程(init 进程)


# 二、kernel层

Android内核与linux内核启动的方式差不多。Kernel的启动流程：

> alps/kernel/init/main.c
> **start_kernel() ==> rest_init() ==> kernel_thread(kernel_init) ==> kernel_init()**

## 2.1 rest_init()

```cpp
static noinline void __ref rest_init(void)
{
    ...
    kernel_thread(kernel_init, NULL, CLONE_FS);
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    ...
    init_idle_bootup_task(current);
    schedule_preempt_disabled();
    ...
    cpu_idle();
}
```

1. rest_init中调用kernel_thread函数启动了2个内核线程，分别是：kernel_init和kthreadd
2. 调用schedule函数开启了内核的调度系统，从此linux系统开始转起来了。
3. rest_init最终调用cpu_idle函数结束了整个内核的启动。
也就是说linux内核最终结束了一个函数cpu_idle。这个函数里面肯定是死循环。
4. 简单来说，linux内核最终的状态是：**有事干的时候去执行有意义的工作（执行各个进程任务），实在没活干的时候就去死循环（实际上死循环也可以看成是一个任务）。**
5. 之前已经启动了内核调度系统，调度系统会负责考评系统中所有的进程，这些进程里面只有有哪个需要被运行，调度系统就会终止cpu_idle死循环进程（空闲进程）转而去执行有意义的干活的进程。这样操作系统就转起来了。

** 0号进程：**
swapper进程(pid=0)：又称为idle进程, 叫空闲进程，由系统自动创建, 运行在内核态。
系统初始化过程Kernel由无到有开创的第一个进程, 也是唯一一个没有通过fork或者kernel_thread产生的进程。
swapper进程用于初始化进程管理、内存管理，加载Display,Camera Driver，Binder Driver等相关工作。

** 1号进程 **
init进程(pid=1)：由0号进程通过kernel_thread创建，在内核空间完成初始化后, 加载init程序, 并最终运行在用户空间，`init进程是所有用户进程的鼻祖`。

** 2号进程 **
kthreadd进程(pid=2)：由0号进程通过kernel_thread创建，是Linux系统的内核进程，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。
kthreadd运行在内核空间, 负责所有内核线程的调度和管理 , `kthreadd进程是所有内核进程的鼻祖`。

## 2.2 kernel_init()

init进程会通过kernel_init创建，这个时候是在内核态，那么怎么一步步走到用户态呢？

> kernel_init->init_post->run_init_process->kernel_execve

```cpp
static int __init kernel_init(void * unused)  
{  
    /*Wait until kthreadd is all set-up.*/  
    wait_for_completion(&kthreadd_done);  
     
    //主要是初始化设备驱动，完成其他驱动程序（直接编译进内核的模块）的初始化。
    do_basic_setup();  

    //挂载根文件系统
    if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)  
        printk(KERN_WARNING "Warning: unable to open an initial console.\n"); 

    //定义init进程 
    if (!ramdisk_execute_command)  
        ramdisk_execute_command = "/init";  

    //最后调用init_post，启动进程负责用户空间的初始化 
    init_post();  
}  
```

接下来定位到`init_post`的函数体，代码如下：

```cpp
static noinline int init_post(void)  
{  
    ……//省略部分代码  
    if (ramdisk_execute_command) {  
    //run_init_process执行后将不再返回  
    run_init_process(ramdisk_execute_command);  
    }  
    if (execute_command) {  
       run_init_process(execute_command);  
    }  
    run_init_process("/sbin/init");  
    run_init_process("/etc/init");  
    run_init_process("/bin/init");  
    run_init_process("/bin/sh");  
}  
```

execute_command 是bootloader传递给内核的参数，一般是/init（即根目录下的init程序），也就是调用文件系统里的init进程。如果找不到就会继续寻找“/sbin/init”、“/etc/init”、“/bin/init”、“/bin/sh”，找到后便执行run_init_process，且不再返回。run_init_process的函数体非常简单，仅仅是对kernel_execve函数的封装，代码如下：

```cpp
static void run_init_process(const char *init_filename)  
{  
  argv_init[0] = init_filename;  
  kernel_execve(init_filename, argv_init, envp_init);  
} 
```
`kernel_execve`是Linux内核中创建用户进程的方法接口，其实现位于
> arch/arm/kernel/sys_arm.c。

那么至此，我们已经对Android Kernel如何引导以及用户空间1号进程（init进程）如何启动做了详细分析。

# 三、Init进程

## 3.1 Init Process

这里的Native层主要说明init进程，当kernel Initialize完成之后，系统会执行第一个用户进程init
，我们可以说它是root进程或者所有进程的父进程。init进程相关的代码：

> /system/core/init/init.cpp：
> /system/core/rootdir/init.rc
> /system/core/init/readme.txt

Init进程启动过程就是代码init.cpp中main函数执行过程，我们来看看它做了什么：

```cpp
int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }

    // Clear the umask.
    umask(0);

    add_environment("PATH", _PATH_DEFPATH);

    bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);

    //挂载tmpfs, devpts, proc, sysfs文件系统
    if (is_first_stage) {
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }

    // We must have some place other than / to create the device nodes for
    // kmsg and null, otherwise we won't be able to remount / read-only
    // later on. Now that tmpfs is mounted on /dev, we can actually talk
    // to the outside world.
    //屏蔽标准的输入输出，即标准的输入输出定向到NULL设备。  
    open_devnull_stdio();
    //kernel log初始化
    klog_init();
    klog_set_level(KLOG_NOTICE_LEVEL);

    NOTICE("init %s started!\n", is_first_stage ? "first stage" : "second stage");

    if (!is_first_stage) {
        // Indicate that booting is in progress to background fw loaders, etc.
        close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
        //创建一块共享的内存空间，用于属性服务
        property_init();

        // If arguments are passed both on the command line and in DT,
        // properties set in DT always have priority over the command-line ones.
        //设置基本属性 
        process_kernel_dt();
        process_kernel_cmdline();

        // Propagate the kernel variables to internal variables
        // used by init as well as the current required properties.
        export_kernel_boot_props();
    }

    // Set up SELinux, including loading the SELinux policy if we're in the kernel domain.
    // 加载SELinux
    selinux_initialize(is_first_stage);

    // If we're in the kernel domain, re-exec init to transition to the init domain now
    // that the SELinux policy has been loaded.
    if (is_first_stage) {
        if (restorecon("/init") == -1) {
            ERROR("restorecon failed: %s\n", strerror(errno));
            security_failure();
        }
        char* path = argv[0];
        char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
        if (execv(path, args) == -1) {
            ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
            security_failure();
        }
    }

    // These directories were necessarily created before initial policy load
    // and therefore need their security context restored to the proper value.
    // This must happen before /dev is populated by ueventd.
    NOTICE("Running restorecon...\n");
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon("/property_contexts");
    restorecon_recursive("/sys");

    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        ERROR("epoll_create1 failed: %s\n", strerror(errno));
        exit(1);
    }
    //初始化子进程退出的信号处理过程
    signal_handler_init();
    //加载/default.prop文件
    property_load_boot_defaults();
    export_oem_lock_status();
    //启动属性服务
    start_property_service();

    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    // 添加解service, on, import 解析器
    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    // 解析init rc文件
    parser.ParseConfig("/init.rc");

    ActionManager& am = ActionManager::GetInstance();

    //执行rc文件中触发器为 on early-init的语句
    am.QueueEventTrigger("early-init");
    //创建wait_for_coldboot_done 动作并添加到action vector和trigger_queue_中
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");

    //执行rc文件中触发器为 on init的语句
    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = property_get("ro.bootmode");
    if (bootmode == "charger") {
        //充电模式下，执行rc文件中触发器为charger的语句
        am.QueueEventTrigger("charger");
    } else {
        //非充电模式下，执行rc文件中触发器为late-init的语句
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    // 创建queue_property_triggers动作并且添加到action vector和trigger_queue_中
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        if (!waiting_for_exec) {
            am.ExecuteOneCommand();
            // 重启一些关键进程
            restart_processes();
        }

        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (am.HasMoreCommands()) {
            timeout = 0;
        }

        bootchart_sample(&timeout);

        epoll_event ev;
        //循环 等待事件发生
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}

```

**可以看到，init进程主要以下功能：**

1. 是挂载tmpfs, devpts, proc, sysfs`文件系统`。
2. 是运行init.rc脚本，init将会解析init.rc，并且执行init.rc中的命令。
3. 当一些关键进程死亡时，重启该进程；
4. 提供Android系统的属性服务

## 3.2 文件系统简介

** 1. tmpfs文件系统**

tmpfs是一种虚拟内存文件系统，因此它会将所有的文件存储在虚拟内存中，并且tmpfs下的所有内容均为临时性的内容，如果你将tmpfs文件系统卸载后，那么其下的所有的内容将不复存在。tmpfs是一个独立的文件系统，不是块设备，只要挂接，立即就可以使用。

** 2. devpts文件系统 **  

devpts文件系统为伪终端提供了一个标准接口，它的标准挂接点是/dev/pts。只要pty的主复合设备/dev/ptmx被打开，就会在/dev/pts下动态的创建一个新的pty设备文件。

** 3. proc文件系统 **

proc文件系统是一个非常重要的虚拟文件系统，它可以看作是内核内部数据结构的接口，通过它我们可以获得系统的信息，同时也能够在运行时修改特定的内核参数。

** 4. sysfs文件系统 **

与proc文件系统类似，sysfs文件系统也是一个不占有任何磁盘空间的虚拟文件系统。它通常被挂接在/sys目录下。sysfs文件系统是Linux2.6内核引入的，它把连接在系统上的设备和总线组织成为一个分级的文件，使得它们可以在用户空间存取。


# 四、init rc文件和语法

init rc文件语法是以行尾单位，以空格间隔的语法，以#开始代表注释行。rc文件主要包含`Action`、`Service`、`Command`、`Options`，其中对于Action和Service的名称都是唯一的，对于重复的命名视为无效。

## 4.1 动作Action

Action： 通过trigger，即以 on开头的语句，决定何时执行相应的service。

- **on early-init**; 在初始化早期阶段触发；
- **on init**; 在初始化阶段触发；
- **on late-init**; 在初始化晚期阶段触发；
- **on boot/charger**： 当系统启动/充电时触发，还包含其他情况，此处不一一列举；
- **on property:<key>=<value>**: 当属性值满足条件时触发；

## 4.2 服务Service

服务Service，以service开头，由init进程启动，一般运行于另外一个init的子进程，所以启动service前需要判断对应的可执行文件是否存在。init生成的子进程，定义在rc文件，其中每一个service，**在启动时会通过fork方式生成子进程。**

例如： `service servicemanager /system/bin/servicemanager`代表的是服务名为servicemanager，服务的路径，也就是服务执行操作时运行/system/bin/servicemanager。

## 4.3 命令Command

下面列举常用的命令

- class_start <service_class_name>： 启动属于同一个class的所有服务；
- start <service_name>： 启动指定的服务，若已启动则跳过；
- stop <service_name>： 停止正在运行的服务
- setprop <name> <value>：设置属性值
- mkdir <path>：创建指定目录
- symlink <target> <sym_link>： 创建连接到<target>的<sym_link>符号链接；
- write <path> <string>： 向文件path中写入字符串；
- exec： fork并执行，会阻塞init进程直到程序完毕；
- exprot <name> <name>：设定环境变量；
- loglevel <level>：设置log级别

## 4.4 可选操作Options

Options是Services的可选项，与service配合使用

- disabled: 不随class自动启动，只有根据service名才启动；
- oneshot: service退出后不再重启；
- user/group： 设置执行服务的用户/用户组，默认都是root；
- class：设置所属的类名，当所属类启动/退出时，服务也启动/停止，默认为default；
- onrestart:当服务重启时执行相应命令；
- socket: 创建名为/dev/socket/<name>的socket
- critical: 在规定时间内该service不断重启，则系统会重启并进入恢复模式

**default:* 意味着disabled=false，oneshot=false，critical=false。

所有的Service里面只有servicemanager ，zygote ，surfaceflinger这3个service有onrestart关键字来触发其他service启动过程。

# 五、Daemon守护进程

init.rc会启动一些daemon进程，比如ueventd, adbd, servicemanager, vold, netd, debuggerd等。

```  
service ueventd /sbin/ueventd  
    class core  
    critical  
  
service console /system/bin/sh  
    class core  
    console  
    disabled  
    user shell  
    group log  
  
service adbd /sbin/adbd  
    class core  
    disabled  
      
service servicemanager /system/bin/servicemanager  
    class core  
    user system  
    group system  
    critical  
    onrestart restart zygote  
    onrestart restart media  
    onrestart restart surfaceflinger  
    onrestart restart drm  
  
service vold /system/bin/vold  
    class core  
    socket vold stream 0660 root mount  
    ioprio be 2  
  
service netd /system/bin/netd  
    class main  
    socket netd stream 0660 root system  
    socket dnsproxyd stream 0660 root inet  
    socket mdns stream 0660 root system  
  
service debuggerd /system/bin/debuggerd  
    class main  
```

# 六、ServiceManager

ServiceManager也是守护进程，它是android的服务大管家，是一个很重要的服务：

```
//servicemanager可触发healthd、zygote、media、surfaceflinger、drm重启
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm
```

`service servicemanager /system/bin/servicemanager`表示服务名为servicemanager，服务运行的时候会执行`/system/bin/servicemanager`。

# 七、Zygote 

Zygote是第一个Java进程，并且**是所有java进程的父进程**。在init.zygote32.rc文件中，zygote服务定义如下：

```cpp
//zygote可触发media、netd重启
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

**Zygote入口和相关类：**

> /frameworks/base/cmds/app_process/App_main.cpp （内含AppRuntime类）
> /frameworks/base/core/jni/AndroidRuntime.cpp
> /frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
> /frameworks/base/core/java/com/android/internal/os/Zygote.java
> /frameworks/base/core/java/android/net/LocalServerSocket.java
> /system/core/libutils/Threads.cpp

解释下第一行参数：

- zygote : 服务名
- systen/bin/app_process : zygote所对应的可执行文件是，通过调用pid =fork()创建子进程
- 剩下的四个参数是zygote进程的`启动参数`，其中最后一个参数`--start-system-server`是表示zygote进程启动完成后，马上将system进程启动起来。

zygote启动过程调用堆栈：

```
App_main.main()
    AndroidRuntime.start()
        StartVM()
        StartReg()
        ZygoteInit.main()
            ResgisterZygoteSocket()
            preload()
            startSystemServer()
            runSelectLoop()
```

zygote进程的主要工作如下：

1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
3. 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5. preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，用于提高ap启动效率；
6. zygote完毕大部分工作，接下来再通过startSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。
7. zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。


# 八、system_server

上面提到Zygote启动过程中会调用startSystemServer()，可知startSystemServer()函数是system_server启动流程的起点， system_server相关类函数如下：


> /frameworks/base/core/java/android/app/ActivityThread.java
> /frameworks/base/core/java/android/app/LoadedApk.java
> /frameworks/base/core/java/android/app/ContextImpl.java
> /frameworks/base/core/java/com/android/server/LocalServices.java
> /frameworks/base/services/java/com/android/server/SystemServer.java
> /frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
> /frameworks/base/services/core/java/com/android/server/ServiceThread.java
> /frameworks/base/services/core/java/com/android/server/pm/Installer.java
> /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java


**启动流程图如下：**

![system_server.png](/img/archives/system_server.jpg)

进入到SystemServer.main后，调用堆栈如下：
```
SystemServer.main
    SystemServer.run
        createSystemContext
            ActivityThread.systemMain
                ActivityThread.attach
                    LoadedApk.makeApplication
            ActivityThread.getSystemContext
                ContextImpl.createSystemContext
        startBootstrapServices();//启动引导服务
        startCoreServices();     // 启动核心服务
        startOtherServices();    // 启动其他服务
        Looper.loop();
```

有兴趣的读者可以跟着follow源码一步步展开，由于篇幅所限，只做总结归纳，system_server最主要的工作就是启动系统服务。 通过startBootstrapServices(), startCoreServices(), startOtherServices()3个方法。

## 8.1 startBootstrapServices();

[ SystemServer.java ] 启动引导服务

```java
private void startBootstrapServices() {
    //阻塞等待与installd建立socket通道
    Installer installer = mSystemServiceManager.startService(Installer.class);

    //启动服务ActivityManagerService
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);

    //启动服务PowerManagerService
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

    //初始化power management
    mActivityManagerService.initPowerManagement();

    //启动服务LightsService
    mSystemServiceManager.startService(LightsService.class);

    //启动服务DisplayManagerService
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    //Phase100: 在初始化package manager之前，需要默认的显示.
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

    //当设备正在加密时，仅运行核心
    String cryptState = SystemProperties.get("vold.decrypt");
    if (ENCRYPTING_STATE.equals(cryptState)) {
        mOnlyCore = true;
    } else if (ENCRYPTED_STATE.equals(cryptState)) {
        mOnlyCore = true;
    }

    //启动服务PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();

    //启动服务UserManagerService，新建目录/data/user/
    ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

    AttributeCache.init(mSystemContext);

    //设置AMS
    mActivityManagerService.setSystemProcess();

    //启动传感器服务
    startSensorService();
}
```

该方法所创建的服务：
`ActivityManagerService`, `PowerManagerService`, `LightsService`, `DisplayManagerService`， `PackageManagerService`， `UserManagerService`， `SensorService`服务。

## 8.2 startCoreServices();     

[ SystemServer.java ]  启动核心服务

```java
private void startCoreServices() {
    //启动服务BatteryService，用于统计电池电量，需要LightService.
    mSystemServiceManager.startService(BatteryService.class);

    //启动服务UsageStatsService，用于统计应用使用情况
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));

    mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

    //启动服务WebViewUpdateService
    mSystemServiceManager.startService(WebViewUpdateService.class);
}
```

启动服务`BatteryService`，`UsageStatsService`，`WebViewUpdateService`。

## 8.3 startOtherServices()

[ SystemServer.java ]  启动其他服务

该方法比较长，有近千行代码，逻辑很简单，主要是启动一系列的服务:

```java
private void startOtherServices() {
        ...
        SystemConfig.getInstance();
        mContentResolver = context.getContentResolver(); // resolver
        ...
        mActivityManagerService.installSystemProviders(); //provider
        mSystemServiceManager.startService(AlarmManagerService.class); // alarm
        // watchdog
        watchdog.init(context, mActivityManagerService); 
        inputManager = new InputManagerService(context); // input
        wm = WindowManagerService.main(...); // window
        inputManager.start();  //启动input
        mDisplayManagerService.windowManagerAndInputReady();
        ...
        mSystemServiceManager.startService(MOUNT_SERVICE_CLASS); // mount
        mPackageManagerService.performBootDexOpt();  // dexopt操作
        ActivityManagerNative.getDefault().showBootMessage(...); //显示启动界面
        ...
        statusBar = new StatusBarManagerService(context, wm); //statusBar
        //dropbox
        ServiceManager.addService(Context.DROPBOX_SERVICE,
                    new DropBoxManagerService(context, new File("/data/system/dropbox")));
         mSystemServiceManager.startService(JobSchedulerService.class); //JobScheduler
         lockSettings.systemReady(); //lockSettings

        //phase480 和phase500
        mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
        mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
        ...
        // 准备好window, power, package, display服务
        wm.systemReady();
        mPowerManagerService.systemReady(...);
        mPackageManagerService.systemReady();
        mDisplayManagerService.systemReady(...);
        //见下面分析
        mActivityManagerService.systemReady(new Runnable() {...});
```

其中AMS.systemReady()的大致过程如下:

```java
public final class ActivityManagerService extends ActivityManagerNative
    implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        
    public void systemReady(final Runnable goingCallback) {
        ... //update相关
        mSystemReady = true;
        
        //杀掉所有非persistent进程
        removeProcessLocked(proc, true, false, "system update done");
        mProcessesReady = true; 

        goingCallback.run();  //[见小节1.6.2]
        
        addAppLocked(info, false, null); //启动所有的persistent进程
        mBooting = true; 
        
        //启动home
        startHomeActivityLocked(mCurrentUserId, "systemReady"); 
        //恢复栈顶的Activity
        mStackSupervisor.resumeTopActivitiesLocked();
    }
}
```

ActivityManagerService的systemReady方法，在该方法里会启动系统界面以及Home程序。








