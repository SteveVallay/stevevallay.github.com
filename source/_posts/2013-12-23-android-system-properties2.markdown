---
layout: post
title: "android_system_properties 2"
date: 2013-12-23 10:17
comments: true
categories:
- android
- SystemProperties
keywords: android,SystemProperties
description: android,SystemProperties
---

静态的程序代码和程序运行时的状态，有很大的不同。理解代码，要理解其运行时的样子。

<!--more-->

这一篇主要说说 android property service 的运行时状态。

先来看看这个图大致理解一下:

![android-property][101]

可以查看下手机的 /dev/__properties__ 文件(Kitkat 4.4) ：

```
# ls -al /dev/**__proper*
-rw-r--r-- root     root        65536 1970-01-12 05:24 __properties__
```


Init 进程，创建 **/dev/__properties__** 文件，map 到内存，然后从 **/default.prop**等文件中加载 properties, 写入 **/dev/__properties__**. 然后启动 property_service , 就是建立一个 `property_service` 的 socket ，监听这个 socket ，其他进程通过向这个 socket 发送 proper_set 的消息来完成 properties 的设置。

proper_get 是在每个进程在初始化时(libc中) 建立了 **/dev/__properties__**到内存的 map ，得到了可以直接访问的 address，可以直接遍历 properties 的存储空间完成查找.


下面细细到来：


Init 进程，Android 用户空间的第一个进程，内核启动之后会执行这个 init 程序。进入到 init 的main 函数。 在 init.c 的 main 函数里面，初始化 property 相关的存储空间。

**system/core/init/init.c**

```c
 property_init();
```

`property_init` 是调用 property_service.c 的方法。

**/system/core/init/property_service.c** 
```
 void property_init(void)
 {
     init_property_area();
 }

static int init_property_area(void)
{
    if (property_area_inited)
        return -1;

    if(__system_property_area_init()) //关键这个
        return -1;

    if(init_workspace(&pa_workspace, 0))
        return -1;

    fcntl(pa_workspace.fd, F_SETFD, FD_CLOEXEC);

    property_area_inited = 1;
    return 0;
}
```

`__system_property_area_init` 方法在 **bionic/libc/bionic/system_properties.c**


```c
int __system_property_area_init()
{
    return map_prop_area_rw();
}

static int map_prop_area_rw()
{
    prop_area *pa;
    int fd;
    int ret;

    /* dev is a tmpfs that we can use to carve a shared workspace
     * out of, so let's do that...
     */
    fd = open(property_filename, O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC |
            O_EXCL, 0444);
    if (fd < 0) {
        if (errno == EACCES) {
            /* for consistency with the case where the process has already
             * mapped the page in and segfaults when trying to write to it
             */
            abort();
        }
        return -1;
    }

    ret = fcntl(fd, F_SETFD, FD_CLOEXEC);
    if (ret < 0)
        goto out;

    if (ftruncate(fd, PA_SIZE) < 0)
        goto out;

    pa_size = PA_SIZE;
    pa_data_size = pa_size - sizeof(prop_area);
    compat_mode = false;

    pa = mmap(NULL, pa_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if(pa == MAP_FAILED)
        goto out;

    memset(pa, 0, pa_size);
    pa->magic = PROP_AREA_MAGIC;
    pa->version = PROP_AREA_VERSION;
    /* reserve root node */
    pa->bytes_used = sizeof(prop_bt);

    /* plug into the lib property services */
    __system_property_area__ = pa;

    close(fd);
    return 0;

out:
    close(fd);
    return -1;
}

```

首先创建 **/dev/__properties__** ，并且用 `mmap` 映射到内存.（通过变量`__system_property_area__` 的共享，完成 property_service 和 libc 中 properties 相关的交互。)

```c
#define PROP_FILENAME "/dev/__properties__"
static char property_filename[PATH_MAX] = PROP_FILENAME;

 fd = open(property_filename, O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC |
            O_EXCL, 0444);
...
    pa = mmap(NULL, pa_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

    /* plug into the lib property services */
    __system_property_area__ = pa;

```

回到 init.c , 加载 boot defults properties：

init.c
```c
 property_load_boot_defaults();
```

property_service.c:
```c
#define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"

 void property_load_boot_defaults(void)
 {    
     load_properties_from_file(PROP_PATH_RAMDISK_DEFAULT);
 }
```


`load_properties_from_file` 这个里面就是读取文件，然后 `set_property`。

回到 init.c , 启动 property service :

init.c: 
```c
 queue_builtin_action(property_service_init_action, "property_service_init");

static int property_service_init_action(int nargs, char **args)
{
    /* read any property files on system or data and
     * fire up the property service.  This must happen
     * after the ro.foo properties are set above so
     * that /data/local.prop cannot interfere with them.
     */
    start_property_service();

    /* update with vendor-specific property runtime
     * overrides
     */
    vendor_load_properties();
    return 0;
}

```

在 `start_property_service` 中，加载 properties (/system/build.prop, /system/default.prop, /data/property/xxx)，创建 **/dev/socket/property_service** 这个 socket, 并且监听这个 socket 来接受 set property 的消息。

property_service.c: 
```c

void start_property_service(void)
{
    int fd; 

    load_properties_from_file(PROP_PATH_SYSTEM_BUILD);
    load_properties_from_file(PROP_PATH_SYSTEM_DEFAULT);
    load_override_properties();
    /* Read persistent properties after all default values have been loaded. */
    load_persistent_properties();

    fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM, 0666, 0, 0); 
    if(fd < 0) return;
    fcntl(fd, F_SETFD, FD_CLOEXEC);
    fcntl(fd, F_SETFL, O_NONBLOCK);

    listen(fd, 8); 
    property_set_fd = fd; 
}
```

回到 init.c :

init 进程在 main 函数的最后，进入一个无限循环，等待 **/dev/socket/property_service** 和其他 fd 的事件并处理。

```c

    for(;;) {
        int nr, i, timeout = -1;

        execute_one_command();
        restart_processes();

        if (!property_set_fd_init && get_property_set_fd() > 0) { 
            ufds[fd_count].fd = get_property_set_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0; 
            fd_count++;
            property_set_fd_init = 1; 
        }    
        if (!signal_fd_init && get_signal_fd() > 0) { 
            ufds[fd_count].fd = get_signal_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0; 
            fd_count++;
            signal_fd_init = 1; 
        }    
        if (!keychord_fd_init && get_keychord_fd() > 0) { 
            ufds[fd_count].fd = get_keychord_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0; 
            fd_count++;
            keychord_fd_init = 1; 
        }    
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

        nr = poll(ufds, fd_count, timeout);
        if (nr <= 0)
            continue;

        for (i = 0; i < fd_count; i++) {
            if (ufds[i].revents == POLLIN) {
                if (ufds[i].fd == get_property_set_fd())
                    handle_property_set_fd();                 //关键这里
                else if (ufds[i].fd == get_keychord_fd())
                    handle_keychord();
                else if (ufds[i].fd == get_signal_fd())
                    handle_signal();
            }
        }
    }

```

`handler_property_set_fd` 里面接收 `/dev/socket/property_service` 的消息并处理。

property_service.c:

```c
void handle_property_set_fd()
{
    prop_msg msg;
    int s; 
    int r;  
    int res;    
    struct ucred cr;
    struct sockaddr_un addr;
    socklen_t addr_size = sizeof(addr);
    socklen_t cr_size = sizeof(cr);
    char * source_ctx = NULL;

    if ((s = accept(property_set_fd, (struct sockaddr *) &addr, &addr_size)) < 0) {
        return;
    }   
            
    /* Check socket options here */
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {
        close(s);
        ERROR("Unable to receive socket options\n");
        return;     
    }           
                    
    r = TEMP_FAILURE_RETRY(recv(s, &msg, sizeof(msg), 0));
    if(r != sizeof(prop_msg)) {
        ERROR("sys_prop: mis-match msg size received: %d expected: %d errno: %d\n",
              r, sizeof(prop_msg), errno);
        close(s);
        return;
    }
    switch(msg.cmd) {
    case PROP_MSG_SETPROP:                              //处理 set proper 请求
        msg.name[PROP_NAME_MAX-1] = 0;
        msg.value[PROP_VALUE_MAX-1] = 0;

        if (!is_legal_property_name(msg.name, strlen(msg.name))) {
            ERROR("sys_prop: illegal property name. Got: \"%s\"\n", msg.name);
            close(s);
            return;
        }

        getpeercon(s, &source_ctx);

        if(memcmp(msg.name,"ctl.",4) == 0) {
            // Keep the old close-socket-early behavior when handling
            // ctl.* properties.
            close(s);
            if (check_control_perms(msg.value, cr.uid, cr.gid, source_ctx)) {
                handle_control_message((char*) msg.name + 4, (char*) msg.value);
            } else {
                ERROR("sys_prop: Unable to %s service ctl [%s] uid:%d gid:%d pid:%d\n",
                        msg.name + 4, msg.value, cr.uid, cr.gid, cr.pid);
            }
        } else {
            if (check_perms(msg.name, cr.uid, cr.gid, source_ctx)) {
                property_set((char*) msg.name, (char*) msg.value);    //设置 prop
            } else {
                ERROR("sys_prop: permission denied uid:%d  name:%s\n",
                      cr.uid, msg.name);
            }

            // Note: bionic's property client code assumes that the
            // property server will not close the socket until *AFTER*
            // the property is written to memory.
            close(s);
        }
        freecon(source_ctx);
        break;

    default:
        close(s);
        break;
    }
}
```

在 `property_set` 之前会 `check_perms`, 不同的 property_set 需要什么权限呢?

```c
/* White list of permissions for setting property services. */
struct {
    const char *prefix;
    unsigned int uid;
    unsigned int gid;
} property_perms[] = { 
    { "net.rmnet0.",      AID_RADIO,    0 },
    { "net.gprs.",        AID_RADIO,    0 },
    { "net.ppp",          AID_RADIO,    0 },
    { "net.qmi",          AID_RADIO,    0 },
    { "net.lte",          AID_RADIO,    0 },
    { "net.cdma",         AID_RADIO,    0 },
    { "ril.",             AID_RADIO,    0 },
    { "gsm.",             AID_RADIO,    0 },
    { "persist.radio",    AID_RADIO,    0 },
    { "net.dns",          AID_RADIO,    0 },
    { "sys.usb.config",   AID_RADIO,    0 },
    { "net.",             AID_SYSTEM,   0 },
    { "dev.",             AID_SYSTEM,   0 },
    { "runtime.",         AID_SYSTEM,   0 },
    { "hw.",              AID_SYSTEM,   0 },
    { "sys.",             AID_SYSTEM,   0 },
    { "sys.powerctl",     AID_SHELL,    0 },
    { "service.",         AID_SYSTEM,   0 },
    { "wlan.",            AID_SYSTEM,   0 },
    { "bluetooth.",       AID_BLUETOOTH,   AID_SYSTEM },
    { "dhcp.",            AID_SYSTEM,   0 },
    { "dhcp.",            AID_DHCP,     0 },
    { "debug.",           AID_SYSTEM,   0 },
    { "debug.",           AID_SHELL,    0 },
    { "log.",             AID_SHELL,    0 },
    { "service.adb.root", AID_SHELL,    0 },
    { "service.adb.tcp.port", AID_SHELL,    0 },
    { "persist.sys.",     AID_SYSTEM,   0 },
    { "persist.service.", AID_SYSTEM,   0 },
    { "persist.security.", AID_SYSTEM,   0 },
    { "persist.service.bdroid.", AID_BLUETOOTH,   0 },
    { "selinux."         , AID_SYSTEM,   0 },
    { NULL, 0, 0 } 
};
```

接着就 `property_set`:

先查询有的话，就 `__system_property_update` , 没有就 `__system_property_add` ,如果是 `persist.`的话，要 `write_persistent_property` 写入到 **/data/property/xxx**


```c
int property_set(const char *name, const char *value)
{
    prop_info *pi;
    int ret;

    size_t namelen = strlen(name);
    size_t valuelen = strlen(value);

    if (!is_legal_property_name(name, namelen)) return -1; 
    if (valuelen >= PROP_VALUE_MAX) return -1; 

    pi = (prop_info*) __system_property_find(name);

    if(pi != 0) {
        /* ro.* properties may NEVER be modified once set */
        if(!strncmp(name, "ro.", 3)) return -1; 

        __system_property_update(pi, value, valuelen);
    } else {
        ret = __system_property_add(name, namelen, value, valuelen);
        if (ret < 0) {
            ERROR("Failed to set '%s'='%s'\n", name, value);
            return ret;
        }   
    }   
    /* If name starts with "net." treat as a DNS property. */
    if (strncmp("net.", name, strlen("net.")) == 0)  {
        if (strcmp("net.change", name) == 0) {
            return 0;
        }   
       /*  
        * The 'net.change' property is a special property used track when any
        * 'net.*' property name is updated. It is _ONLY_ updated here. Its value
        * contains the last updated 'net.*' property.
        */
        property_set("net.change", name);
    } else if (persistent_properties_loaded &&
            strncmp("persist.", name, strlen("persist.")) == 0) {
        /*  
         * Don't write properties to disk until after we have read all default properties
         * to prevent them from being overwritten by default values.
         */
        write_persistent_property(name, value);
    } else if (strcmp("selinux.reload_policy", name) == 0 &&
               strcmp("1", value) == 0) {
        selinux_reload_policy();
    }
    property_changed(name, value);
    return 0;
}
```


OK， `proper_set` 流程先到这里，下面来看看 `proper_get` . `proper_get` 直接向下到 libc 的 `__system_property_get` 的 api.

**/bionic/libc/bionic/system_properties.c**

```c
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
```

这里是调用了 `__system_property_find` 来查找这个值。在往下看：
```
const prop_info *__system_property_find(const char *name)
{
    if (__predict_false(compat_mode)) {  //貌似没打开 compat_mode 
        return __system_property_find_compat(name);
    }
    return find_property(root_node(), name, strlen(name), NULL, 0, false); //看这里
}
```

这里调用 `find_property` 来访问 `root_node()`, 什么是 `root_node()`?

```c
static prop_bt *root_node()
{
    return to_prop_obj(0);
}

static void *to_prop_obj(prop_off_t off)
{
    if (off > pa_data_size)
        return NULL;

    return __system_property_area__->data + off;
}
```

实际上获取 `__system_property_area__->data `的地址开始访问。OK, `__system_property_area__` 是哪里？ 正是 **/dev/__properties__** map 到内存的地址。

init.c 进程调用了 system_properties.c 的 `__system_property_area_init` -> `map_prop_area_rw` 在这里：

创建 `property_filename`(/dev/_-properties__), `mmap` 到内存， 将地址赋值给 `__system_property_area__` 。

```c
static int map_prop_area_rw()
{
...
fd = open(property_filename, O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC |
            O_EXCL, 0444);
...
 pa = mmap(NULL, pa_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

...
/* plug into the lib property services */
    __system_property_area__ = pa;
...
}
```

但是，慢着！ 这些都是在 init 进程里面执行的，其他进程调用 `__system_property_find` 怎么可以直接访问 `__system_property_area__` ? 难道 其他进程和 init 共享了这个地址，这是不可能的，这个变量怎会传递到 init 的子进程？就算传递了，它们怎么可能访问相同的地址呢？它们可是不同进程啊，各自使用自己独享的内存空间啊！

所以，呵呵！ 其他进程肯定也对 `__system_property_area__` 进行初始化了！ 在哪里 ？
**/bionic/libc/bionic/libc_init_common.cpp** 中 调用了 `__system_properties_init()`
```c
void __libc_init_common(KernelArgumentBlock& args) {
...
 __system_properties_init();
}
```

**/bionic/libc/bionic/system_properties.c**
```c
int __system_properties_init()
{
    return map_prop_area();
}

static int map_prop_area()
{
 ...
 fd = open(property_filename, O_RDONLY | O_NOFOLLOW | O_CLOEXEC);
 ...
prop_area *pa = mmap(NULL, pa_size, PROT_READ, MAP_SHARED, fd, 0);
...
 __system_property_area__ = pa;
...
}
```

以 RDONLY 模式打开了 `/dev/__properties__` 文件，并且 mmap 到内存，将地址赋值给 `__system_property_area__` ！ 所以，其他进程可以直接访问 ``__system_property_area__`` 不过这个肯定和 init 进程里那个是不同的！！


OK！ 结束，下一篇讲讲 property 存储区的数据结构 ！ 











[101]:/images/blog/android_property.png