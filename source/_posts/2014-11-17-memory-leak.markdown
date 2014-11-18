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

Then I use `procrank` to check status, find Vss of phone and testlink(my test app) are very high:
\> 2000M

``` sh
# procrank
  PID       Vss      Rss      Pss      Uss  cmdline
  919   643824K   61016K   33791K   30616K  system_server
 1147  2082976K   52676K   26426K   23688K  com.android.phone
...
31155  2009512K   22032K    4000K    3384K  com.example.testlink

```




