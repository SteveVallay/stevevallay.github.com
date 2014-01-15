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

打开 init.c （kitkat)，找到 `main` 函数,来看看 `main` init 主要做了哪些工作（...表示省略部分代码)：

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





``` system/core/rootdir/init.rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

[101]:http://stevevallay.github.io/blog/2013/12/23/android-system-properties2/