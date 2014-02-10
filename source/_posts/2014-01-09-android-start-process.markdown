---
layout: post
title: "Android 启动流程"
date: 2014-01-09 14:23
comments: true
categories:
- android
- startup
keywords: android,startup，Zygote,init,system server，Android 启动流程
description: android startup process
---

保持你的好奇心和创造力。

<!--more-->

早就好奇， Android 启动的流程是怎样的，正好有时间好好看一下。

在追溯代码之前，先让我们提出几个问题，Andorid 启动过程中需要完成什么任务？然后，站在设计者的角度，想一想，如果是你来设计，你会怎么做？

1. 系统服务什么形式存在？如何启动？
2. 系统服务如何为应用提供服务？
3. 应用如何被启动，应用之间如何交互？
4. 应用如何安装/升级？
5. 权限如何控制？

暂时想到这些问题，没有细节到具体的模块(如 Telephony,Multimedia 等).下面，来揭开 Android 启动之谜.


## Power 键  到 User Space

按下 Power 键的时候，系统加电，触发 bootloader (代码见 bootable/),bootloader 载入内存，bootloader 引导 kernel, kernel 启动，kernel 启动用户空间的 init 进程。

这部分还不是很熟悉，这里不做详解。:-)

## Init 进程

init 进程是内核启动后调用的第一个用户空间的程序，从 init 开始，系统开始进入用户空间。

init 是一个二进制文件，代码在: **system/core/init/init.c**

打开 init.c （kitkat)，找到 `main` 函数,来看看 init 主要做了哪些工作（...表示省略部分代码)：

```c system/core/init/init.c
int main(int argc, char **argv)
{
...
    /*清除 umask*/
    umask(0);

    /*创建目录，挂载文件系统*/
    mkdir("/dev", 0755);
    mkdir("/proc", 0755);
    mkdir("/sys", 0755);

    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    mount("proc", "/proc", "proc", 0, NULL);
    mount("sysfs", "/sys", "sysfs", 0, NULL);

    /*猜猜这句是干什么？很有意思 哈哈！*/
    close(open("/dev/.booting", O_WRONLY | O_CREAT, 0000));

    /*打开 /dev/null 并将 0,1,2 重定向到 /dev/null*/
    open_devnull_stdio();

    klog_init();
    /*初始化 property 详见[android properties][101]*/
    property_init();
    get_hardware_name(hardware, &revision);

    process_kernel_cmdline();

    /*selinux 相关*/
    union selinux_callback cb;
    cb.func_log = klog_write;
    selinux_set_callback(SELINUX_CB_LOG, cb);

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    selinux_initialize();

    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon_recursive("/sys");

    /*load 默认的properties*/
    if (!is_charger)
        property_load_boot_defaults();

    /*init 的主要工作之一:解析 init.rc */
    INFO("reading config file\n");
    init_parse_config_file("/init.rc");

    /*action_for_each_trigger 和 queue_builtin_action 将 action/command 都是加入到 command 队列里面，等待合适的时机执行*/
    action_for_each_trigger("early-init", action_add_queue_tail);

    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    queue_builtin_action(keychord_init_action, "keychord_init");
    queue_builtin_action(console_init_action, "console_init");


    /* execute all the boot actions to get us started */
    action_for_each_trigger("init", action_add_queue_tail);

    if (!is_charger) {
        action_for_each_trigger("early-fs", action_add_queue_tail);
        action_for_each_trigger("fs", action_add_queue_tail);
        action_for_each_trigger("post-fs", action_add_queue_tail);
        action_for_each_trigger("post-fs-data", action_add_queue_tail);
    } 

    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    queue_builtin_action(property_service_init_action, "property_service_init");
    queue_builtin_action(signal_init_action, "signal_init");
    queue_builtin_action(check_startup_action, "check_startup");

    if (is_charger) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("early-boot", action_add_queue_tail);
        action_for_each_trigger("boot", action_add_queue_tail);
    }

    /* run all property triggers based on current state of the properties */
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");


#if BOOTCHART
    queue_builtin_action(bootchart_init_action, "bootchart_init");
#endif

    /*解析完 init.rc 之后到此的这一段，好像都是在想队列里面添加 action/command*/

    /*什么时候来执行呢？下面！*/

    /*init 最后进入了一个无限循环，不会主动退出！*/
    for(;;) {
        int nr, i, timeout = -1;

        /*执行command 队列里面的一条命令!*/
        execute_one_command();
        restart_processes();

        /*添加property fd (socket） 到 ufds */
        if (!property_set_fd_init && get_property_set_fd() > 0) {
            ufds[fd_count].fd = get_property_set_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            property_set_fd_init = 1;
        }

        /*添加 signal fd 到 ufds*/
        if (!signal_fd_init && get_signal_fd() > 0) {
            ufds[fd_count].fd = get_signal_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            signal_fd_init = 1;
        }

        /*添加 keychord ?fd 到 ufds*/
        if (!keychord_fd_init && get_keychord_fd() > 0) {
            ufds[fd_count].fd = get_keychord_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            keychord_fd_init = 1;
        }

        /*计算进程restart 时间*/
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (!action_queue_empty() || cur_action)
            timeout = 0;

#if BOOTCHART
        if (bootchart_count > 0) {
            if (timeout < 0 || timeout > BOOTCHART_POLLING_MS)
                timeout = BOOTCHART_POLLING_MS;
            if (bootchart_step() < 0 || --bootchart_count == 0) {
                bootchart_finish();
                bootchart_count = 0;
            }
        }
#endif

        /*poll出 ufds 的事件，等待 timeout 时间*/
        nr = poll(ufds, fd_count, timeout);
        if (nr <= 0)
            continue;
       /*处理 ufds 事件!(property set , signal,keychord?)*/
        for (i = 0; i < fd_count; i++) {
            if (ufds[i].revents == POLLIN) {
                if (ufds[i].fd == get_property_set_fd())
                    handle_property_set_fd();
                else if (ufds[i].fd == get_keychord_fd())
                    handle_keychord();
                else if (ufds[i].fd == get_signal_fd())
                    handle_signal();
            }
        }

    }
}

```

代码看下来，可以看出来，init 的主要工作就是:

1. 解析 init.rc 文件并执行相应的 action/commands.
2. 作为一个 Daemon 进程处理 property_set,keychord(key combo） 和 signal 事件。

init.rc 的解析过程略过，详情请见 system/core/init/init_parser.c


下面来看看 init.rc 里面究竟干了些什么？

```sh system/core/rootdir/init.rc
#导入其他 rc 文件
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc  #这里保留了导入特定平台相关 rc 的接口,这里可能导入很多东西！
import /init.trace.rc

#early-init 要处理的事情
on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_adj -16 

    # Set the security context for the init process.
    # This should occur before anything else (e.g. ueventd) is started.
    setcon u:r:init:s0

    start ueventd

# create mountpoints
    mkdir /mnt 0775 root system

#init 要处理的事情
on init

sysclktz 0

loglevel 3

# Backward compatibility
    symlink /system/etc /etc
    symlink /sys/kernel/debug /d

# Right now vendor lives on the same filesystem as system,
# but someday that may change.
    symlink /system/vendor /vendor

#...省略部分代码

on post-fs
    # once everything is setup, no need to modify /
    mount rootfs rootfs / ro remount
    # mount shared so changes propagate into child namespaces
    mount rootfs rootfs / shared rec
    mount tmpfs tmpfs /mnt/secure private rec
#...省略部分代码

on post-fs-data
    # We chown/chmod /data again so because mount is run as root + defaults
    chown system system /data
    chmod 0771 /data
    # We restorecon /data in case the userdata partition has been reset.
    restorecon /data
#...省略部分代码

on boot
# basic network init
    ifup lo
    hostname localhost
    domainname localdomain
#...省略部分代码

#这两句重要！！
    class_start core
    class_start main

on nonencrypted
    class_start late_start

on charger
    class_start charger

on property:vold.decrypt=trigger_reset_main
    class_reset main
#...省略部分代码

service console /system/bin/sh
    class core
    console
    disabled
    user shell
    group log

#...省略部分代码

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
#...省略部分代码
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

虽然 ini.rc 里面这么多内容，但是你仔细观察就会发现，这里面只有两种模式：

- 第一种 trigger 触发
```
on trigger
   ...
```
- 第二种 service 
```
service ...
   ...
```

`on xxxx` 这种是基于某个事件触发， init.rc 里面列举的有

- on early-init
- on init
- on post-fs
- on boot
- on nonencrypted
- on charger
- on property:key=value

你一定会奇怪，这些 trigger 是如何定义的，在什么时间点触发？ 其实这些 trigger 没有定义，触发的顺序由 init.c 里面添加它们到 action queue 的顺序决定。

```c  system/core/init/init.c
action_for_each_trigger("early-init", action_add_queue_tail);
...
action_for_each_trigger("init", action_add_queue_tail);
action_for_each_trigger("early-fs", action_add_queue_tail);
...
```

在 init.c 中以合适的顺序添加到 action queue 里面之后，在 init.c 最后依次从 action queue 中取出这些 action,顺序执行。

```c system/core/init/init.c
    for(;;) {
        int nr, i, timeout = -1;

        execute_one_command();  //这里取出 action 来执行。
        ...
    }
```

`on property` 例外，这个是在 set property 之后，查询有没有相关的 action，如果有的话添加到 action queue , 等待取出来执行。


那么 service 是如何启动的呢？service 并没有像 trigger 一样的方式进入 action queue.仔细观察一下 service 里面的 option 就会发现，每个 service 都有一个 class.

```
service ueventd /sbin/ueventd
    class core

service servicemanager /system/bin/servicemanager
    class core

service healthd-charger /sbin/healthd -n
    class charger

service netd /system/bin/netd
    class main

service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
...
```

这个 class 指的就是 service 的类别， service 不是像 trigger 一样，通过名字来查找，而是通过类别来一起启动的。

在 `on boot`，可以看到:

```
on boot
...
    class_start core
    class_start main
```

`core` 和 `main` 类别的 service 在 `on boot` 的时候被一起启动了.


通过上面的分析，可以看得出来，虽然 init.rc 里面的内容很多，但还是很容易理解的：

- on xxx 是在 init.c 里面通过 xxx 关键字进入 action 队列并顺序执行的。
- service xxx 是以 `class xxx` 分类的，一类在一起进入队列并执行的， core 和 main 类别的 service 是 on boot 的时候一起执行的。


OK, 看得出来，其实那些 service 并没有什么特别的，都是 init 启动的而已。

那么，Android 是如何从 Native 切换到 Java 世界的呢？这依赖于 init.rc 里面启动的一个重要 service  -- zygote

zygote 是在 init.rc 中被定义为一个 service ：

```bash
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

name = `zygote`
path = `/system/bin/app_process`
arguments = `-Xzygote /system/bin --zygote --start-system-server`

zygote 这个 service 在启动的时候实际上是执行了 /system/bin/app_process 这个二进制文件 , 这个文件的源码在:

`frameworks/base/cmds/app_process/app_main.cpp` 

```c frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
...
//-Xzygote 传给 Vm
    int i = runtime.addVmArguments(argc, argv);
...
//其他参数解析
    while (i < argc) {
        const char* arg = argv[i++];
        if (!parentDir) {
            parentDir = arg;
        } else if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = "zygote";
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName = arg + 12;
        } else {
            className = arg; 
            break;
        }
    }
...
// call runtime.start 并 传了两个参数
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");
    }
    ...
}
```


在最后调用了 `runtime` 的 start 方法并传入了 `com.android.internal.os.ZygoteInit` 和 `start-system-server` 两个参数。

`runtime` 是 `AppRuntime` 的实例， `start` 是继承自基类 `AndroidRuntime` 的方法。

```cpp frameworks/core/jni/AndroidRuntime.cpp
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const char* options)
{
...
     if (startVm(&mJavaVM, &env) != 0) {
        return;
     }
...
    onVmCreated(env);
...
    strArray = env->NewObjectArray(2, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
    optionsStr = env->NewStringUTF(options);
    env->SetObjectArrayElement(strArray, 1, optionsStr);
...
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
     }

}
```

调用 `start` 的时候传递了两个参数 `com.android.internal.os.ZygoteInit` , `start-system-server`,对应 `start` 的两个形参 `className` 和 `options`。在 `start` 里面，可以看到，先是 启动了 VM , 然后将两个参数放到 VM 的 env 里面，在然后，找到 `com.android.internal.os.ZygoteInit` 并调用其  `main` 方法 !

哈哈！ 终于开启了 java 世界！ 来看看 `ZygoteInit` 干了些什么?

```java frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static void main(String argv[]) {
...
            registerZygoteSocket();
...
            if (argv[1].equals("start-system-server")) {
                startSystemServer();
            } else if (!argv[1].equals("")) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }
...
            runSelectLoop();
...
}
```

`ZygoteInit` 的 `main` 里面做了三件事 :

1.  注册 zygote socket 
2.  启动 system server 
3.  无限循环监听 zygote socket 消息。

在 init 启动 zygote 的时候创建了一个 socket - `/dev/socket/zygote` 

```bash
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
...
```

`registerZygoteSocket` 就是用这个 socket 的 fd 创建了一个 Server socket , 用来接收 client 发来的消息。


然后调用，`startSystemServer()` 来启动 system server 进程。
```java ZygoteInit.java
    /**
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer()
            throws MethodAndArgsCaller, RuntimeException {
...
        /* Hardcoded command line to start the system server */
        String args[] = { 
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };
...
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
...
        /* For child process */
        if (pid == 0) {
            handleSystemServerProcess(parsedArgs);
        }

        return true;
```

这一步，调用 `Zygote.forkSystemServer` , `forkSystemServer` 又调用了 `nativenativeForkSystemServer`

```java  libcore/dalvik/src/main/java/dalvik/system/Zygote.java
    public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        postFork();
        return pid;
    }
```

`nativenativeForkSystemServer` 在 dalvik 里面 

```cpp dalvik/vm/native/dalvik_system_Zygote.cpp
static void Dalvik_dalvik_system_Zygote_forkSystemServer(
        const u4* args, JValue* pResult)
{
    pid_t pid;
    pid = forkAndSpecializeCommon(args, true);

    /* The zygote process checks whether the child process has died or not. */
    if (pid > 0) {
        int status;

        ALOGI("System server process %d has been created", pid);
        gDvm.systemServerPid = pid;
        /*如果 system server 没启动成功，zygote 会自杀!*/
        if (waitpid(pid, &status, WNOHANG) == pid) {
            ALOGE("System server process %d has died. Restarting Zygote!", pid);
            kill(getpid(), SIGKILL);
        }   
    }   
    RETURN_INT(pid);
}
```

来看看 `forkAndSpecializeCommon`:

```
static pid_t forkAndSpecializeCommon(const u4* args, bool isSystemServer)
{
...
    setSignalHandler();
...
    pid = fork();
...
    if (pid == 0) {
        unsetSignalHandler();
    }
...
}
```
这里 fork 之前，注册了 signal 处理函数 `setSignalHandler`, 子进程里面又取消了signal 处理函数，说明这个是为父进程 zygote 注册的，来看看里面干了些啥：

```
static void sigchldHandler(int s)
{
...
        /*
         * If the just-crashed process is the system_server, bring down zygote
         * so that it is restarted by init and system server will be restarted
         * from there.
         */
        if (pid == gDvm.systemServerPid) {
            ALOG(LOG_INFO, ZYGOTE_LOG_TAG,
                "Exit zygote because system server (%d) has terminated",
                (int) pid);
            kill(getpid(), SIGKILL);
        }
...
}
```

这里如果 system server 挂了， zygote 又要自杀!!


OK , 假设正常执行，继续回到 `startSystemServer`, 下一步， system server 和 zygote 分道扬镳了终于, system server 调用 `handleSystemServerProcess`, zygote 则调用 `runSelectLoop()`, 先看 zygote `runSelectLoop`:

```
    /** 
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     *
     * @throws MethodAndArgsCaller in a child process when a main() should
     * be executed.
     */
    private static void runSelectLoop() throws MethodAndArgsCaller {
        ....
        while (true) {
            ...
            } else if (index == 0) {
                ZygoteConnection newPeer = acceptCommandPeer();
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                boolean done;
                done = peers.get(index).runOnce();
            }
        }
    }
```

`acceptCommandPeer()` 实际是在阻塞式的等待 client 来发起连接。

```
    /**
     * Waits for and accepts a single command connection. Throws
     * RuntimeException on failure.
     */
    private static ZygoteConnection acceptCommandPeer() {
        try {
            return new ZygoteConnection(sServerSocket.accept());
        } catch (IOException ex) {
            throw new RuntimeException(
                    "IOException during accept()", ex);
        }
    }
```

`runOnce()` 则是 client 建立连接之后，读取 client 发过来的数据，并 fork 出新的进程。

```
    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
            ...
            args = readArgumentList();
            ...
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
            parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
            parsedArgs.niceName);
            ...
    }
```

回过头来看 system server `handleSystemServerProcess`:

```java ZygoteInit.java
 /** 
 * Finish remaining work for the newly forked system server process.
 */
 private static void handleSystemServerProcess( ZygoteConnection.Arguments parsedArgs) throws ZygoteInit.MethodAndArgsCaller {
    ...
    RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs);
    ...
 }
```

调用了 `RuntimeInit.zygoteInit`:

```java RuntimeInit.java
    public static final void zygoteInit(int targetSdkVersion, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
        
        //重定向 log
        redirectLogStreams();
        
        //设置 timezone ,useragent 等等
        commonInit();
        //app_main.cpp 的 onZygoteInit, 初始化 Binder thread pool.
        nativeZygoteInit();
       
        //
        applicationInit(targetSdkVersion, argv);
    } 
```

继续 `applicationInit`,传入的 `startClass` 参数正是 "com.android.server.SystemServer" , 所以这里是掉到了 `SystemServer` 的 main 方法。

```java RuntimeInit.java
    private static void applicationInit(int targetSdkVersion, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {
        ...
        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs);
    }
```

终于到了 SystemServer 里面来了 :
```java SystemServer.java
    public static void main(String[] args) {
        ...
        System.loadLibrary("android_servers");
        // Initialize native services.
        nativeInit();
        // This used to be its own separate thread, but now it is
        // just the loop we run on the main thread.
        ServerThread thr = new ServerThread();
        thr.initAndLoop();
    }
```

主要干了两件事：

1. nativeInit
2. ServerThread 启动


nativeInit 好像没干啥嘛

```cpp services/jni/com_android_server_SystemServer.cpp
static void android_server_SystemServer_nativeInit(JNIEnv* env, jobject clazz) {
    char propBuf[PROPERTY_VALUE_MAX];
    property_get("system_init.startsensorservice", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        // Start the sensor service
        SensorService::instantiate();
    }   
}
```

ServerThread 里面内容可就多喽 ！

```java SystemServer.java
    public void initAndLoop() {
        ...
        Looper.prepareMainLooper();

        //为 window manager 创建一个 handler.
        HandlerThread wmHandlerThread = new HandlerThread("WindowManager");
        wmHandlerThread.start();
        Handler wmHandler = new Handler(wmHandlerThread.getLooper());
        wmHandler.post(new Runnable() {
            @Override
            public void run() {
                android.os.Process.setThreadPriority(
                        android.os.Process.THREAD_PRIORITY_DISPLAY);
                android.os.Process.setCanSelfBackground(false);
            }
        });

        // 启动 installd , Power Manager Activity Manager service.
        boolean onlyCore = false;
        boolean firstBoot = false;
        try {
            installer = new Installer();
            installer.ping();

            power = new PowerManagerService();
            ServiceManager.addService(Context.POWER_SERVICE, power);

            context = ActivityManagerService.main(factoryTest);
        }


        boolean disableStorage = SystemProperties.getBoolean("config.disable_storage", false);
        boolean disableMedia = SystemProperties.getBoolean("config.disable_media", false);
        boolean disableBluetooth = SystemProperties.getBoolean("config.disable_bluetooth", false);
        boolean disableTelephony = SystemProperties.getBoolean("config.disable_telephony", false);
        boolean disableLocation = SystemProperties.getBoolean("config.disable_location", false);
        boolean disableSystemUI = SystemProperties.getBoolean("config.disable_systemui", false);
        boolean disableNonCoreServices = SystemProperties.getBoolean("config.disable_noncore", false);
        boolean disableNetwork = SystemProperties.getBoolean("config.disable_network", false);

        
        //各种 service add 

        try {
            display = new DisplayManagerService(context, wmHandler);
            ServiceManager.addService(Context.DISPLAY_SERVICE, display, true);

            telephonyRegistry = new TelephonyRegistry(context);
            ServiceManager.addService("telephony.registry", telephonyRegistry);

            if (android.telephony.MSimTelephonyManager.getDefault().isMultiSimEnabled()) {
                msimTelephonyRegistry = new MSimTelephonyRegistry(context);
                ServiceManager.addService("telephony.msim.registry", msimTelephonyRegistry);
            }

            ServiceManager.addService("scheduling_policy", new SchedulingPolicyService());

            String cryptState = SystemProperties.get("vold.decrypt");
            if (ENCRYPTING_STATE.equals(cryptState)) {
                onlyCore = true;
            } else if (ENCRYPTED_STATE.equals(cryptState)) {
                onlyCore = true;
            }

            pm = PackageManagerService.main(context, installer,
                    factoryTest != SystemServer.FACTORY_TEST_OFF,
                    onlyCore);

            ActivityManagerService.setSystemProcess();

            ServiceManager.addService("entropy", new EntropyMixer(context));

            ServiceManager.addService(Context.USER_SERVICE,
                    UserManagerService.getInstance());

            mContentResolver = context.getContentResolver();

                accountManager = new AccountManagerService(context);
                ServiceManager.addService(Context.ACCOUNT_SERVICE, accountManager);

            contentService = ContentService.main(context,
                    factoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL);

            ActivityManagerService.installSystemProviders();

            lights = new LightsService(context);

            battery = new BatteryService(context, lights);
            ServiceManager.addService("battery", battery);

            vibrator = new VibratorService(context);
            ServiceManager.addService("vibrator", vibrator);

            consumerIr = new ConsumerIrService(context); 
            ServiceManager.addService(Context.CONSUMER_IR_SERVICE, consumerIr);

            power.init(context, lights, ActivityManagerService.self(), battery,
                    BatteryStatsService.getService(),
                    ActivityManagerService.self().getAppOpsService(), display);

            alarm = new AlarmManagerService(context);
            ServiceManager.addService(Context.ALARM_SERVICE, alarm);

            Watchdog.getInstance().init(context, battery, power, alarm,
                    ActivityManagerService.self());
            Watchdog.getInstance().addThread(wmHandler, "WindowManager thread");

            inputManager = new InputManagerService(context, wmHandler);

            wm = WindowManagerService.main(context, power, display, inputManager,
                    wmHandler, factoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL,
                    !firstBoot, onlyCore);
            ServiceManager.addService(Context.WINDOW_SERVICE, wm);
            ServiceManager.addService(Context.INPUT_SERVICE, inputManager);

            inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
            inputManager.start();

            display.setWindowManager(wm);
            display.setInputManager(inputManager);


                    imm = new InputMethodManagerService(context, wm);
                    ServiceManager.addService(Context.INPUT_METHOD_SERVICE, imm);

                    ServiceManager.addService(Context.ACCESSIBILITY_SERVICE,
                            new AccessibilityManagerService(context));

            wm.displayReady();

            pm.performBootDexOpt();

                    mountService = new MountService(context);
                    ServiceManager.addService("mount", mountService);

            if (!disableNonCoreServices) {
                    lockSettings = new LockSettingsService(context);
                    ServiceManager.addService("lock_settings", lockSettings);

                    devicePolicy = new DevicePolicyManagerService(context);
                    ServiceManager.addService(Context.DEVICE_POLICY_SERVICE, devicePolicy);
            }

            if (!disableSystemUI) {
                    statusBar = new StatusBarManagerService(context, wm);
                    ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);
            }


            if (!disableNonCoreServices) {
                    ServiceManager.addService(Context.CLIPBOARD_SERVICE,
                            new ClipboardService(context));
            }

            if (!disableNetwork) {
                    networkManagement = NetworkManagementService.create(context);
                    ServiceManager.addService(Context.NETWORKMANAGEMENT_SERVICE, networkManagement);
            }

            if (!disableNonCoreServices) {
                    tsms = new TextServicesManagerService(context);
                    ServiceManager.addService(Context.TEXT_SERVICES_MANAGER_SERVICE, tsms);
            }

            if (!disableNetwork) {
                    networkStats = new NetworkStatsService(context, networkManagement, alarm);
                    ServiceManager.addService(Context.NETWORK_STATS_SERVICE, networkStats);

                    networkPolicy = new NetworkPolicyManagerService(
                            context, ActivityManagerService.self(), power,
                            networkStats, networkManagement);
                    ServiceManager.addService(Context.NETWORK_POLICY_SERVICE, networkPolicy);

                    wifiP2p = new WifiP2pService(context);
                    ServiceManager.addService(Context.WIFI_P2P_SERVICE, wifiP2p);

                    wifi = new WifiService(context);
                    ServiceManager.addService(Context.WIFI_SERVICE, wifi);

                    serviceDiscovery = NsdService.create(context);
                    ServiceManager.addService(
                            Context.NSD_SERVICE, serviceDiscovery);
            }

            if (!disableNonCoreServices) {
                    ServiceManager.addService(Context.UPDATE_LOCK_SERVICE,
                            new UpdateLockService(context));
            }


            /*   
             * MountService has a few dependencies: Notification Manager and
             * AppWidget Provider. Make sure MountService is completely started
             * first before continuing.
             */
            if (mountService != null && !onlyCore) {
                mountService.waitForAsecScan();
            }

            try {
                if (accountManager != null)
                    accountManager.systemReady();
            }

            try {
                if (contentService != null)
                    contentService.systemReady();
            }

            try {
                notification = new NotificationManagerService(context, statusBar, lights);
                ServiceManager.addService(Context.NOTIFICATION_SERVICE, notification);
                networkPolicy.bindNotificationManager(notification);
            }

            try {
                ServiceManager.addService(DeviceStorageMonitorService.SERVICE,
                        new DeviceStorageMonitorService(context));
            }

            if (!disableLocation) {
                    location = new LocationManagerService(context);
                    ServiceManager.addService(Context.LOCATION_SERVICE, location);

                    countryDetector = new CountryDetectorService(context);
                    ServiceManager.addService(Context.COUNTRY_DETECTOR, countryDetector);
            }

            if (!disableNonCoreServices) {
                try {
                    Slog.i(TAG, "Search Service");
                    ServiceManager.addService(Context.SEARCH_SERVICE,
                            new SearchManagerService(context));
                }
            }

            try {
                ServiceManager.addService(Context.DROPBOX_SERVICE,
                        new DropBoxManagerService(context, new File("/data/system/dropbox")));
            }

            if (!disableNonCoreServices && context.getResources().getBoolean(
                        R.bool.config_enableWallpaperService)) {
                try {
                    if (!headless) {
                        wallpaper = new WallpaperManagerService(context);
                        ServiceManager.addService(Context.WALLPAPER_SERVICE, wallpaper);
                    }
                }
            }

            if (!disableMedia && !"0".equals(SystemProperties.get("system_init.startaudioservice"))) {
                try {
                    ServiceManager.addService(Context.AUDIO_SERVICE, new AudioService(context));
                }
            }

            if (!disableNonCoreServices) {
                try {
                    // Listen for dock station changes
                    dock = new DockObserver(context);
                }
            }

            if (!disableMedia) {
                try {
                    // Listen for wired headset changes
                    inputManager.setWiredAccessoryCallbacks(
                            new WiredAccessoryManager(context, inputManager));
                } catch (Throwable e) {
                    reportWtf("starting WiredAccessoryManager", e);
                }
            }


            if (!disableNonCoreServices) {
                try {
                    // Manage USB host and device support
                    usb = new UsbService(context);
                    ServiceManager.addService(Context.USB_SERVICE, usb);
                }

                try {
                    // Serial port support
                    serial = new SerialService(context);
                    ServiceManager.addService(Context.SERIAL_SERVICE, serial);
                }
            }

            try {
                twilight = new TwilightService(context);
            }

            try {
                // Listen for UI mode changes
                uiMode = new UiModeManagerService(context, twilight);
            }

            if (!disableNonCoreServices) {
                try {
                    ServiceManager.addService(Context.BACKUP_SERVICE,
                            new BackupManagerService(context));
                }

                try {
                    appWidget = new AppWidgetService(context);
                    ServiceManager.addService(Context.APPWIDGET_SERVICE, appWidget);
                }

                try {
                    recognition = new RecognitionManagerService(context);
                }
            }

            try {
                ServiceManager.addService("diskstats", new DiskStatsService(context));
            }

            try {
                ServiceManager.addService("samplingprofiler",
                            new SamplingProfilerService(context));
            }

            if (!disableNetwork) {
                try {
                    networkTimeUpdater = new NetworkTimeUpdateService(context);
                }
            }

            if (!disableMedia) {
                try {
                    commonTimeMgmtService = new CommonTimeManagementService(context);
                    ServiceManager.addService("commontime_management", commonTimeMgmtService);
                }
            }

            if (!disableNetwork) {
                try {
                    CertBlacklister blacklister = new CertBlacklister(context);
                }
            }

            if (!disableNonCoreServices &&
                context.getResources().getBoolean(R.bool.config_dreamsSupported)) {
                try {
                    // Dreams (interactive idle-time views, a/k/a screen savers)
                    dreamy = new DreamManagerService(context, wmHandler);
                    ServiceManager.addService(DreamService.DREAM_SERVICE, dreamy);
                }
            }

            if (!disableNonCoreServices) {
                try {
                    atlas = new AssetAtlasService(context);
                    ServiceManager.addService(AssetAtlasService.ASSET_ATLAS_SERVICE, atlas);
                }
            }


            try {
                new IdleMaintenanceService(context, battery);
            }

            try {
                printManager = new PrintManagerService(context);
                ServiceManager.addService(Context.PRINT_SERVICE, printManager);
            }

            if (!disableNonCoreServices) {
                try {
                    mediaRouter = new MediaRouterService(context);
                    ServiceManager.addService(Context.MEDIA_ROUTER_SERVICE, mediaRouter);
                }
            }
        //safe mode 相关
        final boolean safeMode = wm.detectSafeMode();
        if (safeMode) {
            ActivityManagerService.self().enterSafeMode();
            // Post the safe mode state in the Zygote class
            Zygote.systemInSafeMode = true;
            // Disable the JIT for the system_server process
            VMRuntime.getRuntime().disableJitCompilation();
        } else {
            // Enable the JIT for the system_server process
            VMRuntime.getRuntime().startJitCompilation();
        }


        //service system ready
        vibrator.systemReady();
        lockSettings.systemReady();
        devicePolicy.systemReady();
        notification.systemReady();
        wm.systemReady();
        if (safeMode) {
            ActivityManagerService.self().showSafeModeOverlay();
        }
        power.systemReady(twilight, dreamy);
        pm.systemReady();
        display.systemReady(safeMode, onlyCore);

        //接下来是 ActivityManagerService 的 systemReady
        ActivityManagerService.self().systemReady(new Runnable() {
            public void run() {

                    ActivityManagerService.self().startObservingNativeCrashes();
                    startSystemUi(contextF);
                    if (mountServiceF != null) mountServiceF.systemReady();
                    if (batteryF != null) batteryF.systemReady();
                    if (networkManagementF != null) networkManagementF.systemReady();
                    if (networkStatsF != null) networkStatsF.systemReady();
                    if (networkPolicyF != null) networkPolicyF.systemReady();
                    if (connectivityF != null) connectivityF.systemReady();
                    if (dockF != null) dockF.systemReady();
                    if (usbF != null) usbF.systemReady();
                    if (twilightF != null) twilightF.systemReady();
                    if (uiModeF != null) uiModeF.systemReady();
                    if (recognitionF != null) recognitionF.systemReady();
                Watchdog.getInstance().start();

                // It is now okay to let the various system services start their
                // third party code...

                    if (appWidgetF != null) appWidgetF.systemRunning(safeMode);
                    if (wallpaperF != null) wallpaperF.systemRunning();
                    if (immF != null) immF.systemRunning(statusBarF);
                    if (locationF != null) locationF.systemRunning();
                    if (countryDetectorF != null) countryDetectorF.systemRunning();
                    if (networkTimeUpdaterF != null) networkTimeUpdaterF.systemRunning();
                    if (commonTimeMgmtServiceF != null) commonTimeMgmtServiceF.systemRunning();
                        textServiceManagerServiceF.systemRunning();
                    if (dreamyF != null) dreamyF.systemRunning();
                    if (atlasF != null) atlasF.systemRunning();
                    if (inputManagerF != null) inputManagerF.systemRunning();
                    if (telephonyRegistryF != null) telephonyRegistryF.systemRunning();
                    if (msimTelephonyRegistryF != null) msimTelephonyRegistryF.systemRunning();
                    if (printManagerF != null) printManagerF.systemRuning();
                    if (mediaRouterF != null) mediaRouterF.systemRunning();
       
        //进入消息循环
        Looper.loop();
    }

```

system server 启动了，来看下 Launcher 是怎么启动的。system server 的最后一段调用了 ActivityManagerService 的 systemReady() 回调。

ActivityManagerService 的 systemReady()：

```java ActivityManagerService.java
    public void systemReady(final Runnable goingCallback) {
        ...
            mStackSupervisor.resumeTopActivitiesLocked();
        ...
    }

```

->mStackSupervisor.resumeTopActivitiesLocked

```java ActivityStackSupervisor.java
    boolean resumeTopActivitiesLocked() {
        return resumeTopActivitiesLocked(null, null, null);
    }    

    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = getFocusedStack();
        }    
        boolean result = false;
        for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = mStacks.get(stackNdx);
            if (isFrontStack(stack)) {
                if (stack == targetStack) {
                    result = stack.resumeTopActivityLocked(target, targetOptions);
                } else {
                    stack.resumeTopActivityLocked(null);
                }    
            }    
        }    
        return result;
    } 
```

->stack.resumeTopActivityLocked

```java ActivityStack.java
    final boolean resumeTopActivityLocked(ActivityRecord prev) {
        return resumeTopActivityLocked(prev, null);
    }

    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (ActivityManagerService.DEBUG_LOCKSCREEN) mService.logLockScreen("");
        ...
        if (next == null) {
            // There are no more activities!  Let's just start up the
            // Launcher...
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: No more activities go home");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return mStackSupervisor.resumeHomeActivity(prev);
        }   
```

->mStackSupervisor.resumeHomeActivity

```java ActivityStackSupervisor.java
    boolean resumeHomeActivity(ActivityRecord prev) {
        ...  
        return mService.startHomeActivityLocked(mCurrentUser);
    } 
```

->mService.startHomeActivityLocked(mCurrentUser)

```java ActivityManagerService.java
    boolean startHomeActivityLocked(int userId) {
        Intent intent = getHomeIntent();
        ActivityInfo aInfo =
            resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(
                    aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);
            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                mStackSupervisor.startHomeActivity(intent, aInfo);
            }
        }
    }
```
-> mStackSupervisor.startHomeActivity

```java ActivityStackSupervisor.java
    void startHomeActivity(Intent intent, ActivityInfo aInfo) {
        moveHomeToTop();
        startActivityLocked(null, intent, null, aInfo, null, null, 0, 0, 0, null, 0,
                null, false, null);
    }

->
startActivityLocked
->
startActivityUncheckedLocked
->
resumeTopActivitiesLocked

```

->ActivityStack.resumeTopActivityLocked

->
ActivityStackSupervisor.startSpecificActivityLocked

-> 
ActivityManagerService.startProcessLocked

->

```java ActivityManagerService.java
            // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, null);
```
->
Process.start

->
startViaZygote

```java Process.java
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex); 
        }    
    }
```

->

```java Process.java
    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
             ...
             return zygoteSendArgsAndGetResult(argsForZygote);
    }
```

```java Process.java
    private static ProcessStartResult zygoteSendArgsAndGetResult(ArrayList<String> args)
            throws ZygoteStartFailedEx {
        openZygoteSocketIfNeeded();
        ...
            sZygoteWriter.write(Integer.toString(args.size()));            sZygoteWriter.newLine();
        ...
            sZygoteWriter.write(arg);
    }
```

openZygoteSocketIfNeeded?

```java Process.java
    private static final String ZYGOTE_SOCKET = "zygote";
    private static void openZygoteSocketIfNeeded()
            throws ZygoteStartFailedEx {
                sZygoteSocket = new LocalSocket();

                sZygoteSocket.connect(new LocalSocketAddress(ZYGOTE_SOCKET,
                        LocalSocketAddress.Namespace.RESERVED));
    }

```

yes -> connect to "zygote" and send args, back to zygote:

```java ZygoteInit.java 
   private static void runSelectLoop(){
                done = peers.get(index).runOnce();
   }
```

->runOnce?

```java ZygoteConnection.java
boolean runOnce(){
...

            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName);
...
handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
}
```
->
Zygote.forkAndSpecialize

```java Zygote.java
    public static int forkAndSpecialize(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, int mountExternal, String seInfo, String niceName) {
        preFork();
        int pid = nativeForkAndSpecialize(
                uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName);
        postFork();
        return pid;
    }
```



[101]:http://stevevallay.github.io/blog/2013/12/23/android-system-properties2/