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

在 init 启动 zygote 









[101]:http://stevevallay.github.io/blog/2013/12/23/android-system-properties2/