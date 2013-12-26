---
layout: post
title: "Android System Properties"
date: 2013-12-13 23:41
comments: true
categories:
- android
- SystemProperties
keywords: android,SystemProperties
description: android,SystemProperties
---

--直朝那个方向走，或许真的能到达那个地方。o(∩∩)o...哈哈



Property system 是 Android 系统中一个重要的 Feature，它以一个 service 的形式来管理系统的配置和状态，每个 property 都是一个 key/value 组，key 和 value 都是字符串。

这些配置和状态信息在 Android 的所有进程中都可以读取、设置和修改，所以 Property system 成了 Android 系统中控制全局配置的一种常用手段。你可以预置 system propterties 作为系统的初始设置，也可以运行是设置和改变 system properties 的值。

因此，system properties 经常作为一些特定 Feature 的控制开关，运行时根据 properties 的值来区分打开/关闭某个 Feature.由于在所有进程都可以访问，也可以用来在 Android 的不同进程间进行简单信息协调，Java 和 native 都不受限制。

下面我们就按自上而下的顺序看看 Android 的这个 Properties system 的实现（Kitkat 4.4)。

<!--more-->


![properties call stack][101]


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

**frameworks/base/core/jni/android_os_SystemProperties.cpp**(android framework 的 native 实现在 **/frameworks/base/core/jni** 下面可以看到)

从代码可以知道，这一层只是调用底层接口，提供 JNI 支持。

```c++
static jstring SystemProperties_getSS(JNIEnv *env, jobject clazz,
                                      jstring keyJ, jstring defJ);
static void SystemProperties_set(JNIEnv *env, jobject clazz,
                                      jstring keyJ, jstring valJ);
...
```

**get/set** 方法内部调用了两个底层接口：

```c++
property_set(key,value);
property_get(key, buf,default);
```
这两个接口定义在哪里呢？

```c
#include "cutils/properties.h"
```

这个.h 文件在 **system/core/include/cutils/properties.h**

在这个文件中可以看到这两个函数的声明。

```c
int property_get(const char *key, char *value, const char *default_value);
int property_set(const char *key, const char *value);
```
这两个函数的实现在哪里呢？ 在 **properties.c** 中

**system/core/libcutils/properties.c**

在这个文件中我们可以看到根据不同的宏定义有几种不同的实现。

```c
#ifdef HAVE_LIBC_SYSTEM_PROPERTIES

#define _REALLY_INCLUDE_SYS__SYSTEM_PROPERTIES_H_
#include <sys/_system_properties.h>
...
#elif defined(HAVE_SYSTEM_PROPERTY_SERVER)
...
#else

/* SUPER-cheesy place-holder implementation for Win32 */

#include <cutils/threads.h>
...
```
在实际的手机运行环境中，property system 使用的是第一种的实现，第二种是模拟器环境的实现，第三种嘛? 嘿嘿 ~

我们重点来看第一种好了，因为第一种是实际的手机运行环境。在这种实现中，同样是调用了两个类似的 `api`  **__system_property_set** 和 **__system_property_get** （在 **sys/_system_properties.h** 中声明的).

```c
#ifdef HAVE_LIBC_SYSTEM_PROPERTIES

#define _REALLY_INCLUDE_SYS__SYSTEM_PROPERTIES_H_
#include <sys/_system_properties.h>

int property_set(const char *key, const char *value)
{
    return __system_property_set(key, value);
}

int property_get(const char *key, char *value, const char *default_value)
{
    int len;

    len = __system_property_get(key, value);
    if(len > 0) {
        return len;
    }

    if(default_value) {
        len = strlen(default_value);
        memcpy(value, default_value, len + 1);
    }
    return len;
}
```
先看一下 **sys/_system_properties.h** 中定义的几个基本结构.
**bionic/libc/include/sys/_system_properties.h**

```
#define PROP_SERVICE_NAME "property_service"

#define TOC_NAME_LEN(toc)       ((toc) >> 24)
#define TOC_TO_INFO(area, toc)  ((prop_info*) (((char*) area) + ((toc) & 0xFFFFFF)))


struct prop_area {
    unsigned volatile count;
    unsigned volatile serial;
    unsigned magic;
    unsigned version;
    unsigned reserved[4];
    unsigned toc[1];
};

struct prop_info {
    char name[PROP_NAME_MAX];
    unsigned volatile serial;
    char value[PROP_VALUE_MAX];
};

struct prop_msg 
{
    unsigned cmd;
    char name[PROP_NAME_MAX];
    char value[PROP_VALUE_MAX];
};

#define PROP_MSG_SETPROP 1

#define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"
#define PROP_PATH_SYSTEM_BUILD     "/system/build.prop"
#define PROP_PATH_SYSTEM_DEFAULT   "/system/default.prop"
#define PROP_PATH_LOCAL_OVERRIDE   "/data/local.prop"

```

这两个 `api` 又在哪里实现呢？ ^_^ 
查看 **bionic/libc/bionic/system_properties.c**

```c
static const char property_service_socket[] = "/dev/socket/" PROP_SERVICE_NAME;

int __system_property_get(const char *name, char *value)
{
    const prop_info *pi = __system_property_find(name);

    if(pi != 0) {
        return __system_property_read(pi, 0, value);
    } else {
        value[0] = 0;
        return 0;
    }
}

int __system_property_set(const char *key, const char *value)
{
    int err;
    int tries = 0;
    int update_seen = 0;
    prop_msg msg;

    if(key == 0) return -1;
    if(value == 0) value = "";
    if(strlen(key) >= PROP_NAME_MAX) return -1;
    if(strlen(value) >= PROP_VALUE_MAX) return -1;

    memset(&msg, 0, sizeof msg);
    msg.cmd = PROP_MSG_SETPROP;
    strlcpy(msg.name, key, sizeof msg.name);
    strlcpy(msg.value, value, sizeof msg.value);

    err = send_prop_msg(&msg);
    if(err < 0) {
        return err;
    }

    return 0;
}

```

的确在这里找到了 **__system_property_get** 和 **__system_property_set** ,这两个函数的实现有包含了 **__system_property_find** **__system_property_read** 和 **send_prop_msg**
```
const prop_info *__system_property_find(const char *name)
{
    prop_area *pa = __system_property_area__;
    unsigned count = pa->count;
    unsigned *toc = pa->toc;
    unsigned len = strlen(name);
    prop_info *pi;

    while(count--) {
        unsigned entry = *toc++;
        if(TOC_NAME_LEN(entry) != len) continue;

        pi = TOC_TO_INFO(pa, entry);
        if(memcmp(name, pi->name, len)) continue;

        return pi;
    }

    return 0;
}

int __system_property_read(const prop_info *pi, char *name, char *value)
{
    unsigned serial, len;

    for(;;) {
        serial = pi->serial;
        while(SERIAL_DIRTY(serial)) {
            __futex_wait((volatile void *)&pi->serial, serial, 0);
            serial = pi->serial;
        }
        len = SERIAL_VALUE_LEN(serial);
        memcpy(value, pi->value, len + 1);
        if(serial == pi->serial) {
            if(name != 0) {
                strcpy(name, pi->name);
            }
            return len;
        }
    }
}

static int send_prop_msg(prop_msg *msg)
{
    struct pollfd pollfds[1];
    struct sockaddr_un addr;
    socklen_t alen;
    size_t namelen;
    int s;
    int r;
    int result = -1;

    s = socket(AF_LOCAL, SOCK_STREAM, 0);
    if(s < 0) {
        return result;
    }

    memset(&addr, 0, sizeof(addr));
    namelen = strlen(property_service_socket);
    strlcpy(addr.sun_path, property_service_socket, sizeof addr.sun_path);
    addr.sun_family = AF_LOCAL;
    alen = namelen + offsetof(struct sockaddr_un, sun_path) + 1;

    if(TEMP_FAILURE_RETRY(connect(s, (struct sockaddr *) &addr, alen)) < 0) {
        close(s);
        return result;
    }

    r = TEMP_FAILURE_RETRY(send(s, msg, sizeof(prop_msg), 0));
    if(r == sizeof(prop_msg)) {
        // We successfully wrote to the property server but now we
        // wait for the property server to finish its work.  It
        // acknowledges its completion by closing the socket so we
        // poll here (on nothing), waiting for the socket to close.
        // If you 'adb shell setprop foo bar' you'll see the POLLHUP
        // once the socket closes.  Out of paranoia we cap our poll
        // at 250 ms.
        pollfds[0].fd = s;
        pollfds[0].events = 0;
        r = TEMP_FAILURE_RETRY(poll(pollfds, 1, 250 /* ms */));
        if (r == 1 && (pollfds[0].revents & POLLHUP) != 0) {
            result = 0;
        } else {
            // Ignore the timeout and treat it like a success anyway.
            // The init process is single-threaded and its property
            // service is sometimes slow to respond (perhaps it's off
            // starting a child process or something) and thus this
            // times out and the caller thinks it failed, even though
            // it's still getting around to it.  So we fake it here,
            // mostly for ctl.* properties, but we do try and wait 250
            // ms so callers who do read-after-write can reliably see
            // what they've written.  Most of the time.
            // TODO: fix the system properties design.
            result = 0;
        }
    }

    close(s);
    return result;
}

```
看到这里，我们大概知道 get 是从一个 prop_info 的结构提中读取，而 set 的则是向 **property_service_socket("/dev/socket/property_service")** 发送数据。但不免又有很多疑问，property 存储在哪，数据结构是怎样的？proper_set 发送socket 数据是谁来接收和处理的？ property system 是如何启动的？

好吧，我们先来总结一下 Android system properties 相关的目录和文件吧。

Java 层：

- frameworks/base/core/java/android/os/SystemProperties.java

```java
/**
 * Gives access to the system properties store.  The system properties
 * store contains a list of string key-value pairs.
 *
 * {@hide}
 */
```

native 层：

Android framework 的 native 实现，或者成为 runtime 都是在 **frameworks/base/core/jni** 目录下。

和 properties 相关的文件：
- frameworks/base/core/jni/android_os_SystemProperties.cpp

```c
#include "cutils/properties.h"
```

cutils: 

**system** 目录，这个目录有什么用呢 ?

> 
System - source code files for the core Android system. That is the minimal Linux system that is started before the Dalvik VM and any java based services are enabled. This includes the source code for the init process and the default init.rc script that provide the dynamic configuration of the platform.

- system/core/libcutils/properties.c  //包含了 _system_properties.h 
- system/core/include/cutils/properties.h //properties.c 的对外接口 被 jni 包含.
- system/core/init/property_service.h  //property_service 的对外接口
- system/core/init/property_service.c  //
- system/core/init/init.c 

libc: 

**Bionic** 这个目录又是干什么的呢？
> 
Bionic - the C-runtime for Android. Note that Android is not using glibc like most Linux distributions. Instead the c-library is called bionic and is based mostly on BSD-derived sources. In this folder you will find the source for the c-library, math and other core runtime libraries.

- bionic/libc/include/sys/_system_properties.h //包含了下面的 system_properties.h
- bionic/libc/include/sys/system_properties.h //下面的system_properties.c 对外接口声明
- /libc/bionic/system_properties.c




[101]:/images/blog/property-call-stack.png
