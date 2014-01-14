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


``` system/core/rootdir/init.rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

