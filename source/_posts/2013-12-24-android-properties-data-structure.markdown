---
layout: post
title: "android properties data structure"
date: 2013-12-24 13:04
comments: true
categories:
- android
- SystemProperties
keywords: android,SystemProperties，data structure
description: android,SystemProperties
---

写文章不应太多拘泥于细节：


- 言简意赅
- 图胜于文
- 图文并茂
- 点到为止

<!-- more -->

从 Jelly Bean 到 KitKat 版本， `properties` 的存储的数据结构还是有些变化的.


实际存储位置： **/dev/\_\_properties\_\_** 文件 (root RW， 其他只能 R)
```bash
ls -al /dev/__proper*
-rw-r--r-- root     root        65536 1970-01-14 05:26 __properties__
```

关于 Jelly Bean（或者以前的)， 直接上张图，比较明了： 

![jb proper data structure][101]


先看下相关结构的定义:

```c  system/core/init/property_service.c
typedef struct {
    void *data;
    size_t size;
    int fd; 
} workspace;

```

```c bionic/libc/bionic/system_properties.c 
struct prop_area {
    unsigned volatile count;
    unsigned volatile serial;  //不知道这个是干什么的?
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
```
toc[i] 里面存放的是 property name 的长度 (前 8 位) 和对应 proper_info 对应的地址 (后 24位)。
proper_info 的 serial (前 8 位) 保存的是 property value 的 length。



但是有几个问题:

- 大小是不对的 , Jelly Bean 上大小是 65536, 可以放 495 个 properties.


```c system/core/init/properties_service.c
/* (8 header words + 495 toc words) = 2012 bytes */
/* 2016 bytes header and toc + 495 prop_infos @ 128 bytes = 65376 bytes */

#define PA_COUNT_MAX  495
#define PA_INFO_START 2016
#define PA_SIZE       65536
```

- share memory 是不对的, 这个图在 init 进程里面没什么问题。但是其他进程访问（读取的时候) 其实是将同样的文件(**/dev/\_\_properties\_\_**) map 了一份到自己的内存空间。所以每个进程的这个结构是自己的，不是共享的。


KitKat ： 

KK 上总大小是  128*1024 

数据结构:
```c bionic/libc/bionic/system_properties.c
struct prop_area {
    unsigned bytes_used;
    unsigned volatile serial;
    unsigned magic;
    unsigned version;
    unsigned reserved[28];
    char data[0];
};

struct prop_info {
    unsigned volatile serial;
    char value[PROP_VALUE_MAX];
    char name[0];
};

struct prop_bt {
    uint8_t namelen;
    uint8_t reserved[3];

    prop_off_t prop;

    prop_off_t left;
    prop_off_t right;

    prop_off_t children;

    char name[0];
};

```

Kitkat 上 proper info 的存储已经改为 bTree 了，bytes_used 用来记录存储这些信息所使用的空间。

```c bionic/libc/bionic/system_properties.c
 * +-----+   children    +----+   children    +--------+
 * |     |-------------->| ro |-------------->| secure |
 * +-----+               +----+               +--------+
 *                       /    \                /   |
 *                 left /      \ right   left /    |  prop   +===========+
 *                     v        v            v     +-------->| ro.secure |
 *                  +-----+   +-----+     +-----+            +-----------+
 *                  | net |   | sys |     | com |            |     1     |
 *                  +-----+   +-----+     +-----+            +===========+
```



好了， 就到这里吧! 感谢 ～ 

参考文档: http://rxwen.blogspot.com/2010/01/android-property-system.html 
























[101]:/images/blog/prop_struct_jb.png