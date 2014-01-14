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

init 是一个二进制文件，代码在: system/core/init/init.c





``` system/core/rootdir/init.rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

