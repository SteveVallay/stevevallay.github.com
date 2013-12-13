---
layout: post
title: "android_System_Properties"
date: 2013-12-13 23:41
comments: true
categories:
- android
- SystemProperties
keywords: android,SystemProperties
description: android,SystemProperties
---

--直朝那个方向走，或许真的能到达那个地方。o(∩∩)o...哈哈
### Android SystemProperties

Property system 是 Android 系统中一个重要的 Feature，它以一个 service 的形式来管理系统的配置和状态，每个 property 都是一个 key/value 组，key 和 value 都是字符串。

这些配置和状态信息在 Android 的所有进程中都可以读取、设置和修改，所以 Property system 成了 Android 系统中控制全局配置的一种常用手段。你可以预置 system propterties 作为系统的初始设置，也可以运行是设置和改变 system properties 的值。

因此，system properties 经常作为一些特定 Feature 的控制开关，运行时根据 properties 的值来区分打开/关闭某个 Feature.由于在所有进程都可以访问，也可以用来在 Android 的不同进程间进行简单信息协调，Java 和 native 都不受限制。

下面我们就按自上而下的顺序看看 Android 的这个 Properties system 的实现。

<!--more-->

#### Java 层 

**frameworks/base/core/java/android/os/SystemProperties.java**

java 层的接口在 _SysstemProperties.java_ 这个文件中,经常使用的接口有以下几个：

```java
/*Get the value for the given key.*/
public static String get(String key)
public static String get(String key, String def)
public static int getInt(String key, int def)
public static long getLong(String key, long def）
public static boolean getBoolean(String key, boolean def)

/*Set the value for the given key.*/
public static void set(String key, String val)
```

简单来说就是 `get` 和 `set` 方法，都是静态方法，直接使用 SystemProperties.get/set 就可以访问。不过 SystemProperties 是一个 *hide* 的类，不在 SDK 的标准 API 中，也就意味着，在基于 SDK 的 app 开发中不能直接使用（可以尝试反射 ^_^)。

#### Framework 层

进到这几个方法的里面来看，就会发现，它们都是调用了 native 方法:

```java
    private static native String native_get(String key);
    private static native String native_get(String key, String def);
    private static native int native_get_int(String key, int def);
    private static native long native_get_long(String key, long def);
    private static native boolean native_get_boolean(String key, boolean def);
    private static native void native_set(String key, String def);
```
这些 native 方法在哪里定义和实现呢 ？

**frameworks/base/core/jni/android_os_SystemProperties.cpp**
