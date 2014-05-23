---
layout: post
title: "file access permissions summary "
date: 2014-05-23 23:08
comments: true
categories: 
- permissions
keywords: file permissions
description: file access permission bits

---

<!-- more --> 

总结一下常用的 file access permission bits. `man 2 stat ` 可以看到几乎所有的 file permission bits. 

###S_ISUID

set-user-ID 

文件： 设置了 `S_ISUID` 的文件在执行的时候会使得进程的 effective UID 和 文件的 UID 相同。

目录： not used 


###S_ISGID 
set-group-ID

文件： 如果文件的 group execute 位设置了，那么设置了 `S_ISGID` 的文件在执行的时候（group 的身份来执行）会使得进程的 effective GID 和 文件 GID 相同。

目录： 目录内新建的文件的 GID 和 目录的GID 一样。


###S_ISVTX 
striky bit 

文件： 内存会对文件进行 cache 

目录： 只允许文件的 owner , 目录的 owner 和 super user 对 目录内文件进行 delete、rename。(目录的 other WX 都设置了的情况下）


###S_IRUSR
user-read 

文件：user 读取文件内容

目录：user 读取目录中文件的名称

###S_IWUSR

user-write

文件：user 写文件

目录：user 对目录中文件进行 rename/delete/create (只有在 S_IXUSR 同时设置了才可以)

###S_IXUSR

user-execute 

文件：user 执行文件

目录：user 访问目录内文件， read/write/exexute/chmod 

###S_IRGRP
group-read 

文件：group 读取文件内容

目录：group 读取目录中文件的名称

###S_IWGRP

group-write

文件：group 写文件

目录：group 对目录中文件进行 rename/delete/create (只有在 S_IXUSR 同时设置了才可以)

###S_IXGRP

group-execute 

文件：group 执行文件

目录：group 访问目录内文件 - read/write/exexute/chmod 


###S_IROTH
other-read 

文件：other 读取文件内容

目录：other 读取目录中文件的名称

###S_IWOTH

other-write

文件：other 写文件

目录：other 对目录中文件进行 rename/delete/create (只有在 S_IXUSR 同时设置了才可以)

###S_IXOTH

other-execute 

文件：other 执行文件

目录：other 访问目录内文件 - read/write/exexute/chmod


###S_IRWXU 
= S_IRUSR | S_IWUSR | S_IXUSR

###S_IRWXG
= S_IRGRP | S_IWGRP | S_IXGRP 

###S_IRWXO
= S_IROTH | S_IWOTH | S_IXOTH 


---

参考 `man 2 stat` 

```
           S_ISUID    0004000   set UID bit
           S_ISGID    0002000   set-group-ID bit (see below)
           S_ISVTX    0001000   sticky bit (see below)
           S_IRWXU    00700     mask for file owner permissions
           S_IRUSR    00400     owner has read permission
           S_IWUSR    00200     owner has write permission
           S_IXUSR    00100     owner has execute permission
           S_IRWXG    00070     mask for group permissions
           S_IRGRP    00040     group has read permission
           S_IWGRP    00020     group has write permission
           S_IXGRP    00010     group has execute permission
           S_IRWXO    00007     mask for permissions for others (not in group)
           S_IROTH    00004     others have read permission
           S_IWOTH    00002     others have write permission
           S_IXOTH    00001     others have execute permission

```












