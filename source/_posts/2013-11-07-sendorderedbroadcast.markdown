---
layout: post
title: "Android sendOrderedBroadcast"
date: 2013-11-07 18:17
comments: true
categories: [Andrid]
keywords: android,Broadcast， monkey
description: sendOrderedBroadcast fail
---

今天遇到一个 monkey 测试中发现的 sendOrderedBroadcast fail 的问题。记录一下。

<!--more--> 


错误 log 如下：

```
Shutting down VM
07-04 06:11:24.319 W/dalvikvm(23361): threadid=1: thread exiting with uncaught exception (group=0x415a8898)
07-04 06:11:24.319 W/BroadcastQueue(  937): Failure sending broadcast Intent { act=android.intent.action.QUERY_PACKAGE_RESTART dat=package:com.cootek.smartinputv5.language.cangjie flg=0x10 (has extras) }
07-04 06:11:24.319 W/BroadcastQueue(  937): android.os.DeadObjectException
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.os.BinderProxy.transact(Native Method)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.content.IIntentReceiver$Stub$Proxy.performReceive(IIntentReceiver.java:124)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at com.android.server.am.BroadcastQueue.performReceiveLocked(BroadcastQueue.java:376)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at com.android.server.am.BroadcastQueue.deliverToRegisteredReceiverLocked(BroadcastQueue.java:449)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at com.android.server.am.BroadcastQueue.processNextBroadcast(BroadcastQueue.java:656)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at com.android.server.am.ActivityManagerService.finishReceiver(ActivityManagerService.java:12451)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.content.BroadcastReceiver$PendingResult.sendFinished(BroadcastReceiver.java:419)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.content.BroadcastReceiver$PendingResult.finish(BroadcastReceiver.java:395)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:780)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.os.Handler.handleCallback(Handler.java:730)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.os.Handler.dispatchMessage(Handler.java:92)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at android.os.Looper.loop(Looper.java:137)
07-04 06:11:24.319 W/BroadcastQueue(  937): 	at com.android.server.ServerThread.run(SystemServer.java:1066)

07-04 06:11:24.329 E/AndroidRuntime(23361): FATAL EXCEPTION: main
07-04 06:11:24.329 E/AndroidRuntime(23361): java.lang.RuntimeException: Error receiving broadcast Intent { act=android.intent.action.QUERY_PACKAGE_RESTART dat=package:com.cootek.smartinputv5.language.cangjie flg=0x10 (has extras) } in com.android.settings.applications.InstalledAppDetails$2@42070e38
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:773)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at android.os.Handler.handleCallback(Handler.java:730)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at android.os.Handler.dispatchMessage(Handler.java:92)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at android.os.Looper.loop(Looper.java:137)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at android.app.ActivityThread.main(ActivityThread.java:5136)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at java.lang.reflect.Method.invokeNative(Native Method)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at java.lang.reflect.Method.invoke(Method.java:525)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:737)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:553)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at dalvik.system.NativeStart.main(Native Method)
07-04 06:11:24.329 E/AndroidRuntime(23361): Caused by: java.lang.NullPointerException
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at com.android.settings.applications.InstalledAppDetails$2.onReceive(InstalledAppDetails.java:1225)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:763)
07-04 06:11:24.329 E/AndroidRuntime(23361): 	... 9 more
07-04 06:11:24.329 I/ActivityManager(  937): Notify an ApplicationCrash
```

查看下代码:

```java
1218     private final BroadcastReceiver mCheckKillProcessesReceiver = new BroadcastReceiver() {
1219         @Override
1220         public void onReceive(Context context, Intent intent) {
1221             updateForceStopButton(getResultCode() != Activity.RESULT_CANCELED);
1222 
1223             Intent i = new Intent("qualcomm.android.LEDFlashlight.processKilled");
1224             i.putExtra(Intent.EXTRA_PACKAGES, mAppEntry.info.packageName);
1225             getActivity().sendStickyBroadcast(i);
1226         }
1227     };
```


可以看出 `NullPointerException` 的直接原因是 `getActivity` 失败了～ 

```java
1225             getActivity().sendStickyBroadcast(i);
```

可是这个怎么会失败呢？


[Stack Overflow][1] 上搜索到的 [解释][2] :

>
I can see this happening if the component that called sendOrderedBroadcast() was destroyed prior to the broadcast winding its way back to the supplied instance of the BroadcastReceiver anonymous subclass.


这个解释说是因为 `sendOrderedBroadcast()` 的组件在 `broadcast` 还没有回调到这个匿名内部类的实例 mCheckKillProcessesReceiver  的时候就已经 destroy 了。

虽然我没有亲自验证，但是有两个问题：

1. 如果 component 已经 destroy 了，按照 Android 的机制，那么 intent 过来的时候应该重新构造这个 component 才对，那么 destroy 又有什么关系呢？
2. 这个匿名内部类的实例化是怎么完成的？ 实例化的时候是否正确的创建了 Context 信息？


[1]:http://stackoverflow.com/
[2]:http://stackoverflow.com/questions/12934990/deadobjectexception-when-trying-to-use-context-sendorderedbroadcast
