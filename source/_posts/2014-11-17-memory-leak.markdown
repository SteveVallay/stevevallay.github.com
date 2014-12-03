---
layout: post
title: "android memory leak"
date: 2014-11-17 14:27
comments: true
categories:
- memory
- cursor leak
keywords: Could not allocate CursorWindow
description: Could not allocate CursorWindow due to error -12
---

Record fix issue "**Could not allocate CursorWindow '/data/data/com.android.providers.telephony/databases/mmssms.db' of size 2097152 due to error -12**" issue.

<!-- more -->


###Issue born

One day, we met an issue caused Messages application force close.

This issue happen when do a simple Message case (new a message -> send -> delete all, then repeat many times)  more than 10 hours.

Logcat info as below:

```
E/CursorWindow( 1350): Could not allocate CursorWindow '/data/data/com.android.providers.telephony/databases/mmssms.db' of size 2097152 due to error -12.
E/JavaBinder( 1350): *** Uncaught remote exception!  (Exceptions are not yet supported across processes.)
E/JavaBinder( 1350): android.database.CursorWindowAllocationException: Cursor window allocation of 2048 kb failed. # Open Cursors=4 (# cursors opened by pid 16143=4)
E/JavaBinder( 1350): 	at android.database.CursorWindow.<init>(CursorWindow.java:107)
E/JavaBinder( 1350): 	at android.database.AbstractWindowedCursor.clearOrCreateWindow(AbstractWindowedCursor.java:198)
E/JavaBinder( 1350): 	at android.database.sqlite.SQLiteCursor.fillWindow(SQLiteCursor.java:139)
E/JavaBinder( 1350): 	at android.database.sqlite.SQLiteCursor.getCount(SQLiteCursor.java:133)
E/JavaBinder( 1350): 	at android.database.CursorToBulkCursorAdaptor.getBulkCursorDescriptor(CursorToBulkCursorAdaptor.java:148)
E/JavaBinder( 1350): 	at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:118)
E/JavaBinder( 1350): 	at android.os.Binder.execTransact(Binder.java:404)
E/JavaBinder( 1350): 	at dalvik.system.NativeStart.run(Native Method)
```

###Guess root cause

Our experience told us there were some `Cursor` not close in this application. Then we checked the code , search keywords `cursor` , but seems all `Cursor` were **closed correctly**.

We searched [Google](www.google.com) and  [StackOverflow](www.stackoverflow.com) , some **similar** issues founded :

1. [Could not allocate CursorWindow](http://stackoverflow.com/questions/17495713/could-not-allocate-cursorwindow)
2. [SQLite Android Database Cursor window allocation of 2048 kb failed](http://stackoverflow.com/questions/11340257/sqlite-android-database-cursor-window-allocation-of-2048-kb-failed)

They all said you need close cursor in finally, but actually cursor already closed in finally.

Then we find our issue's log really has some difference with issues from [StackOverflow](www.stackoverflow.com)

- log of our issue:

E/JavaBinder( 1350): android.database.CursorWindowAllocationException: Cursor window allocation of 2048 kb failed. # **Open Cursors=4** (# cursors opened by pid 16143=4)

- log of [StackOverflow](www.stackoverflow.com) issue:

android.database.CursorWindowAllocationException: Cursor window allocation of 2048 kb failed. # **Open Cursors=861** (# cursors opened by this proc=861)


Opened cursor number is totally different, our is **4**, issue of [StackOverflow](www.stackoverflow.com) 's is **861**. If opened cursor number is large (hundreds of), we believe your code opened many cursors without close them, but this is not our case. 

 
###Check code detail

Failed to create new `CursorWindow`  throw such exception.

``` java frameworks/base/core/java/android/database/CursorWindow.java  CursorWindow
        if (sCursorWindowSize < 0) {
            /** The cursor window size. resource xml file specifies the value in kB.
             * convert it to bytes here by multiplying with 1024.
             */
            sCursorWindowSize = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_cursorWindowSize) * 1024;
        }
        mWindowPtr = nativeCreate(mName, sCursorWindowSize);
        if (mWindowPtr == 0) {
            throw new CursorWindowAllocationException("Cursor window allocation of " +
                    (sCursorWindowSize / 1024) + " kb failed. " + printStats());
        }

```


`nativeCreate` returned  0.

``` c frameworks/base/core/jni/android_database_CursorWindow.cpp nativeCreate
    CursorWindow* window;
    status_t status = CursorWindow::create(name, cursorWindowSize, &window);
    if (status || !window) {
        ALOGE("Could not allocate CursorWindow '%s' of size %d due to error %d.",
                name.string(), cursorWindowSize, status);
        return 0;
    }

```

`CursorWindow::create` returned non-zero.

``` c frameworks/base/libs/androidfw/CursorWindow.cpp  CursorWindow::create
    status_t result;
    int ashmemFd = ashmem_create_region(ashmemName.string(), size);
    if (ashmemFd < 0) {
        result = -errno;
    } else {
        result = ashmem_set_prot_region(ashmemFd, PROT_READ | PROT_WRITE);
        if (result >= 0) {
            void* data = ::mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, ashmemFd, 0);
            if (data == MAP_FAILED) {
                result = -errno;
            } else {
                result = ashmem_set_prot_region(ashmemFd, PROT_READ);
                if (result >= 0) {
                    CursorWindow* window = new CursorWindow(name, ashmemFd,
                            data, size, false /*readOnly*/);
                    result = window->clear();
                    if (!result) {
                        LOG_WINDOW("Created new CursorWindow: freeOffset=%d, "
                                "numRows=%d, numColumns=%d, mSize=%d, mData=%p",
                                window->mHeader->freeOffset,
                                window->mHeader->numRows,
                                window->mHeader->numColumns,
                                window->mSize, window->mData);
                        *outCursorWindow = window;
                        return OK;
                    }
                    delete window;
                }
            }
            ::munmap(data, size);
        }
        ::close(ashmemFd);
    }
    *outCursorWindow = NULL;
    return result;

```

we can see that -12 is -errno, which means **out of memory**:

``` c
#define ENOMEM      12  /* Out of memory */
```

It must returned from **mmap** operation. we can check this by `man mmap` :

> ENOMEM No memory is available, or the process's maximum number of mappings would have been exceeded.


### Test code

Because when this issue occur after 10 hours test(on customer side) and will got message restart , so hard to check phone's status when this happen.

I decided to write some test code to reproduce similar issue.

``` java 
    int number =1000;
    private Cursor [] cursors = new Cursor[number];

    private void testCursor(){
        for(int i=0; i<number;i++){
            cursors[i]= getContentResolver().query(Uri.parse("content://sms"),SMS_PROJECTION, null, null, null);
            Log.d(TAG, "i = " + i + "  count:" + cursors[i].getCount());
        }
    }

```
It will generate logcat info and cause app crash:

```
E/CursorWindow( 1147): Could not allocate CursorWindow '/data/data/com.android.providers.telephony/databases/mmssms.db' of size 2097152 due to error -
12.
E/JavaBinder( 1147): *** Uncaught remote exception!  (Exceptions are not yet supported across processes.)
E/JavaBinder( 1147): android.database.CursorWindowAllocationException: Cursor window allocation of 2048 kb failed. # Open Cursors=732 (# cursors opene
d by pid 845=732)
E/JavaBinder( 1147):    at android.database.CursorWindow.<init>(CursorWindow.java:107)
E/JavaBinder( 1147):    at android.database.AbstractWindowedCursor.clearOrCreateWindow(AbstractWindowedCursor.java:198)
E/JavaBinder( 1147):    at android.database.sqlite.SQLiteCursor.fillWindow(SQLiteCursor.java:139)
E/JavaBinder( 1147):    at android.database.sqlite.SQLiteCursor.getCount(SQLiteCursor.java:133)
E/JavaBinder( 1147):    at android.database.CursorToBulkCursorAdaptor.getBulkCursorDescriptor(CursorToBulkCursorAdaptor.java:148)
E/JavaBinder( 1147):    at android.content.ContentProviderNative.onTransact(ContentProviderNative.java:118)
E/JavaBinder( 1147):    at android.os.Binder.execTransact(Binder.java:404)
E/JavaBinder( 1147):    at dalvik.system.NativeStart.run(Native Method)
D/AndroidRuntime(  845): Shutting down VM
```

Then I use `procrank` (`top` or `ps` also OK)to check status, find Vss of phone(mmssms provider running in phone process) and testlink(my test app) are very high:
\> 2000M

``` sh
\# procrank
  PID       Vss      Rss      Pss      Uss  cmdline
  919   643824K   61016K   33791K   30616K  system_server
 1147  2082976K   52676K   26426K   23688K  com.android.phone
...
31155  2009512K   22032K    4000K    3384K  com.example.testlink

```

Then I use `cat /proc/1147/maps` to see the phone process's memory consumption for the its mappings.

I can see 744 records of `CursorWindow` as below (samilar in testlink process's maps): 

```
61fb5000-621b5000 rw-s 00000000 00:04 146971     /dev/ashmem/CursorWindow: /data/data/com.android.providers.telephony/databases/mmssms.db (deleted)

621b5000-623b5000 rw-s 00000000 00:04 146972     /dev/ashmem/CursorWindow: /data/data/com.android.providers.telephony/databases/mmssms.db (deleted)

623b5000-625b5000 rw-s 00000000 00:04 146973     /dev/ashmem/CursorWindow: /data/data/com.android.providers.telephony/databases/mmssms.db (deleted)

625b5000-627b5000 rw-s 00000000 00:04 146974     /dev/ashmem/CursorWindow: /data/data/com.android.providers.telephony/databases/mmssms.db (deleted)

```


Maybe `procmem` is better for this, I find `procmem` later. Use `procmem` to check  the phone process's maps :

Also can see 744 records of `CursorWindow`, each one cost  2048K Vss.

```
Vss      Rss      Pss      Uss     ShCl     ShDi     PrCl     PrDi  Name

-------  -------  -------  -------  -------  -------  -------  -------  
...
  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

  2048K       4K       4K       4K       0K       0K       0K       4K  /dev/ashmem/CursorWindow:

   132K      36K      16K      16K      20K       0K      16K       0K  [stack]

     0K       0K       0K       0K       0K       0K       0K       0K  [vectors]

-------  -------  -------  -------  -------  -------  -------  -------  

2079416K   39824K   20503K   19360K   20412K      52K   16592K    2976K  TOTAL

```

Theoretically, on 32 bit OS, Vss limit can be 4GB, but on 32 bit Android, Vss limit is 2GB. A process can not use Vss more than this limit. I do not know why.


Now, I can say, in case of my test app, every query try to use a new cursor which need allocate 2M 
Vss . When alloacated Vss reach limit (2G) will return fail with errno 12 `ENOMEM`.


But for the first case , opened cursor number is only 4, most consume 8M memory, should not cause memory exhaust.

To find out what cost the memory, we need check maps of mms process from customer devices. Since we are not there with customer, we need a script to capture related logs.


### Capture logs & fix first issue

Then I wrote an bash script to capture related logs every half an hour.

```bash 
#!/system/bin/sh

set `ps| grep com.android.mms`
pidOfMms=$2
echo "pid of mms is: $pidOfMms"

set `ps|grep com.android.phone`
pidOfPhone=$2
echo "pid of phone is: $pidOfPhone"

if [ ! -d "/sdcard/logs" ]; then
    mkdir "/sdcard/logs"
    echo "create new dir /sdcard/logs"
fi

while [ -d "/sdcard/logs" ]
do
    echo "start capture logs"
    t=`date +"%Y-%m-%d-%H-%M-%S"`
    echo $$ > /sdcard/logs/pid.log
    procrank > "/sdcard/logs/procrank"$t".log"
    dumpsys meminfo com.android.mms > "/sdcard/logs/meminfo.mms"$t".log"
    dumpsys meminfo com.android.phone > "/sdcard/logs/meminfo.phone"$t".log"
    cat /proc/$pidOfPhone/maps > "/sdcard/logs/maps.phone"$t".log"
    cat /proc/$pidOfMms/maps > "/sdcard/logs/maps.mms"$t".log"
    sleep 1800
done
```

`push` this script to **/system/bin/** and add execute permission, run this `sh /system/bin/log.sh &` before test , it will capture logs ever half an hour.

Then I got the maps of info of phone and mms process, after analysis , find 684 records of `/system/framework/framework-res.apk` as below:

```
b75f7000-b772a000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
b7791000-b78c4000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
b78c4000-b79f7000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
b79f7000-b7b2a000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
b7b2a000-b7c5d000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
...
bb165000-bb298000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
bb298000-bb3cb000 r--s 00907000 b3:10 853        /system/framework/framework-res.apk
```

I find lots of overlay errors in adb logs as below: 

```
11-06 10:55:47: D/asset   ( 1241): createIdmapFileLocked: originalPath=/system/framework/framework-res.apk overlayPath=/vendor/ty/overlay/framework/framework-res.apk idmapPath=/data/resource-cache/vendor@ty@overlay@framework@framework-res.apk@idmap
11-06 10:55:47: W/asset   ( 1241): failed to find resources.arsc in /vendor/ty/overlay/framework/framework-res.apk
11-06 10:55:47: W/asset   ( 1241): failed to add overlay package /vendor/ty/overlay/framework/framework-res.apk
...
```

Then, I think maybe [runtime overlay][100] failure caused **/system/framework/framework-res.apk**  loaded every time when app try to get framework resources. Lots of times load **/system/framework/framework-res.apk** without free previous cost memory caused memory leak. 

[runtime overlay][100] failure is because **failed to find resources.arsc in /vendor/ty/overlay/framework/framework-res.apk**. After check **/vendor/ty/overlay/framework/framework-res.apk**, found no resources.arsc in this package because no resource in source code of this package.

Then removed **/vendor/ty/overlay/framework/framework-res.apk** and verify, after testing 10 hours , issue not observed. 



### Same error , another issue

After running 40 hours , similar error observed. Check maps of mms process (before the error), found 576 records of `stack` as below: 

```
711fa000-712f7000 rw-p 00000000 00:00 0          [stack:1851]
712fe000-713fb000 rw-p 00000000 00:00 0          [stack:1852]
71402000-714ff000 rw-p 00000000 00:00 0          [stack:1853]
7436d000-7446a000 rw-p 00000000 00:00 0          [stack:1854]
7446b000-74568000 rw-p 00000000 00:00 0          [stack:1855]
74569000-74666000 rw-p 00000000 00:00 0          [stack:1856]
...
```

Also checked `top` info and `/proc/pid-of-mms/status` info , find Vss is high (1607952K) and lots of  threads (576) there: 

- top info:

```
 PID PR CPU% S  #THR     VSS     RSS PCY UID      Name
 1847 0 1%   S   576 1607952K 210208K  fg u0_a9    com.android.mms
```

- status info:

```
Name:	com.android.mms
State:	S (sleeping)
Tgid:	1847
Pid:	1847
PPid:	203
...
Threads:	578
```

I think there are lots of unexpected thread created, so need capture threads info. Then wrote another script to capture threads info: 

```
#!/system/bin/sh

set `ps| grep com.android.mms`
pidOfMms=$2

tasks=`ls /proc/$pidOfMms/task`

if [ ! -d "/sdcard/logs" ]; then
    mkdir "/sdcard/logs"
    echo "create new dir /sdcard/logs"
fi

for var in $tasks; do
    debuggerd -b $var > "/sdcard/logs/mms-tid-"$var".log"
done
```

When got thread info, found lots of same name threads there: 

```
"k worker thread" sysTid=10137
  #00  pc 00021a58  /system/lib/libc.so (__futex_syscall3+8)
  #01  pc 0000f01c  /system/lib/libc.so (__pthread_cond_timedwait_relative+48)
  #02  pc 0000f07c  /system/lib/libc.so (__pthread_cond_timedwait+64)
  #03  pc 00055c67  /system/lib/libdvm.so
  #04  pc 0006a5d5  /system/lib/libdvm.so
  #05  pc 00029820  /system/lib/libdvm.so
  #06  pc 00030cac  /system/lib/libdvm.so (dvmMterpStd(Thread*)+76)
  #07  pc 0002e344  /system/lib/libdvm.so (dvmInterpret(Thread*, Method const*, JValue*)+184)
  #08  pc 0006347d  /system/lib/libdvm.so (dvmCallMethodV(Thread*, Method const*, Object*, bool, JValue*, std::__va_list)+336)
  #09  pc 000634a1  /system/lib/libdvm.so (dvmCallMethod(Thread*, Method const*, Object*, JValue*, ...)+20)
  #10  pc 0005817f  /system/lib/libdvm.so
  #11  pc 0000d228  /system/lib/libc.so (__thread_entry+72)
```






































[100]:http://developer.sonymobile.com/2014/04/22/sony-contributes-runtime-resource-overlay-framework-to-android-code-example/