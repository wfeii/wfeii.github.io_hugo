---
layout: post
title: "Android Log 分析"
date: 2017-11-04 16:40:12
---

Bug在任何类型的开发中都会存在的，而Bug报告又是定位问题，解决问题的关键。

Bug报告中包含的内容：  

 - logcat文件以文字形式存储所有的log信息
    - main 包含了所有的log
      - system log 存储framework相关的log
      - event log  进程的正在做的事情  
 - dumpsys获取系统运行状态的信息，通过adb shell 可以获取想要的系统信息，内存使用情况的信息等等
    - Activity Manager的状态 - dumpsys activity  
      - Package信息 - dumpsys package  
      - 内存使用情况 - dumpsys meminfo  
      - 进程一段时间内的内存使用情况 - dumpsys procstats  
      - 界面相关的状态 - dumpsys gfxinfo  
 - dumpstate 系统运行状态的信息

>常用命令 ：
> adb shell bugreport  获取logcat信息，dumpsys信息，dumpstate信息
> adb shell logcat -v threadtime -d 获取logcat文件  
>	> adb shell logcat -b events -v threadtime -d  获取events log  
>	> adb shell logcat -b main -v threadtime -d 获取main log
> adb shell dumpsys 获取系统信息。
> adb shell dumpstate 获取系统状态信息，dumpstate包含dumpsys  

下面的部分会细化bug报告中的各部分，描述一个问题给出相关技巧和使用**grep**来查找相关的log

### **LogCat**   
logcat文件以文字形式存储了所有了的日志信息，**system** 部分存储了framework的相关log，而 **main** 包含了所有log。每行log都以 **timestamp PID(Process ID) TID(Thread ID)  log-level** 开头。   

```
\------ SYSTEM LOG (logcat -v threadtime -d *:v) ------
--------- beginning of system  
Blah  
Blah  
Blah  

--------- beginning of main  
Blah   
Blah  
Blah  
```

#### **阅读event log**   
event log 里面包含了文字形式的二进格式的log信息，它比logcat里面的信息简洁但是更难理解。当阅读event log的时候，我们应该搜索特定的PID来看进程做了什么，格式为**timestamp PID TID log-level log-tag tag-values.**   

log等级包括：
- V: verbose
- D: debug
- I: information
- W: warning
- E: error  

```
------ EVENT LOG (logcat -b events -v threadtime -d *:v) ------  
09-28 13:47:34.179   785  5113 I am_proc_bound: [0,23054,com.google.android.gms.unstable]  
09-28 13:47:34.777   785  1975 I am_proc_start: [0,23134,10032,com.android.chrome,broadcast,com.android.chrome/org.chromium.chrome.browser.precache.PrecacheServiceLauncher]  
09-28 13:47:34.806   785  2764 I am_proc_bound: [0,23134,com.android.chrome]  
...  
```

对于Event log标签更详细的信息,可以阅读[Event log tags](http://download.csdn.net/download/wfeii/9966454)   

### **ANR和死锁**      
log能帮助我们找到导致ANR错误和死锁原因。    

####  **识别无响应的APP**  
当应用一段时间无响应的时候，通常是由于主线程阻塞了或者是主线程做的事太多。系统会kill掉应用并且把相关的堆栈信息存储在/data/anr目录下。  
可以使用**grep am_anr**在event log找到原因：    

```   
grep "am_anr" bugreport-2015-10-01-18-13-48.txt  
10-01 18:12:49.599  4600  4614 I am_anr  : [0,29761,com.google.android.youtube,953695941,executing service com.google.android.youtube/com.google.android.apps.youtube.app.offline.transfer.OfflineTransferService]  
10-01 18:14:10.211  4600  4614 I am_anr  : [0,30363,com.google.android.apps.plus,953728580,executing service com.google.android.apps.plus/com.google.android.apps.photos.service.PhotosService]  
```

或者在logcat中使用**grep ANR**来搜索，搜索结果将会包含ANR有关CPU使用的信息。    

#### **查找堆栈信息**  
我们经常可以发现对应的ANR的堆栈信息，确保时间和进程ID匹配正在调查的ANR。然后检查进程的主线的相关堆栈信息。    

> 需要牢记的:

 > - 主线程只是说明线程ANR的时候正在做什么，但是不一定是造成ANR的原因（堆栈信息可能是无辜的，有些事可能卡顿了主线程一段时间但是不足以构成ANR）   
 > - 可能不止一个堆栈信息，确保浏览正确的部分   

```
------ VM TRACES AT LAST ANR (/data/anr/traces.txt: 2015-10-01 18:14:41) ------  

----- pid 30363 at 2015-10-01 18:14:11 -----
Cmd line: com.google.android.apps.plus
Build fingerprint: 'google/angler/angler:6.0/MDA89D/2294819:userdebug/dev-keys'
ABI: 'arm'
Build type: optimized
Zygote loaded classes=3978 post zygote classes=27
Intern table: 45068 strong; 21 weak
JNI: CheckJNI is off; globals=283 (plus 360 weak)
Libraries: /system/lib/libandroid.so /system/lib/libcompiler_rt.so /system/lib/libjavacrypto.so /system/lib/libjnigraphics.so /system/lib/libmedia_jni.so /system/lib/libwebviewchromium_loader.so libjavacore.so (7)
Heap: 29% free, 21MB/30MB; 32251 objects
Dumping cumulative Gc timings
Total number of allocations 32251
Total bytes allocated 21MB
Total bytes freed 0B
Free memory 9MB
Free memory until GC 9MB
Free memory until OOME 490MB
Total memory 30MB
Max memory 512MB
Zygote space size 1260KB
Total mutator paused time: 0
Total time waiting for GC to complete: 0
Total GC count: 0
Total GC time: 0
Total blocking GC count: 0
Total blocking GC time: 0

suspend all histogram:  Sum: 119.728ms 99% C.I. 0.010ms-107.765ms Avg: 5.442ms Max: 119.562ms
DALVIK THREADS (12):  
"Signal Catcher" daemon prio=5 tid=2 Runnable
  | group="system" sCount=0 dsCount=0 obj=0x12c400a0 self=0xef460000  
  | sysTid=30368 nice=0 cgrp=default sched=0/0 handle=0xf4a69930  
  | state=R schedstat=( 9021773 5500523 26 ) utm=0 stm=0 core=1 HZ=100  
  | stack=0xf496d000-0xf496f000 stackSize=1014KB  
  | held mutexes= "mutator lock"(shared held)  
  native: #00 pc 0035a217  /system/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, int, char const*, art::ArtMethod*, void*)+126)  
  native: #01 pc 0033b03b  /system/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&) const+138)  
  native: #02 pc 00344701  /system/lib/libart.so (art::DumpCheckpoint::Run(art::Thread*)+424)  
  native: #03 pc 00345265  /system/lib/libart.so (art::ThreadList::RunCheckpoint(art::Closure*)+200)  
  native: #04 pc 00345769  /system/lib/libart.so (art::ThreadList::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&)+124)  
  native: #05 pc 00345e51  /system/lib/libart.so (art::ThreadList::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char> >&)+312)  
  native: #06 pc 0031f829  /system/lib/libart.so (art::Runtime::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char> >&)+68)  
  native: #07 pc 00326831  /system/lib/libart.so (art::SignalCatcher::HandleSigQuit()+896)  
  native: #08 pc 003270a1  /system/lib/libart.so (art::SignalCatcher::Run(void*)+324)  
  native: #09 pc 0003f813  /system/lib/libc.so (__pthread_start(void*)+30)  
  native: #10 pc 00019f75  /system/lib/libc.so (__start_thread+6)  
  (no managed stack frames)  

"main" prio=5 tid=1 Suspended  
  | group="main" sCount=1 dsCount=0 obj=0x747552a0 self=0xf5376500  
  | sysTid=30363 nice=0 cgrp=default sched=0/0 handle=0xf74feb34  
  | state=S schedstat=( 331107086 164153349 851 ) utm=6 stm=27 core=3 HZ=100  
  | stack=0xff00f000-0xff011000 stackSize=8MB  
  | held mutexes=
  kernel: __switch_to+0x7c/0x88  
  kernel: futex_wait_queue_me+0xd4/0x130  
  kernel: futex_wait+0xf0/0x1f4  
  kernel: do_futex+0xcc/0x8f4  
  kernel: compat_SyS_futex+0xd0/0x14c  
  kernel: cpu_switch_to+0x48/0x4c  
  native: #00 pc 000175e8  /system/lib/libc.so (syscall+28)  
  native: #01 pc 000f5ced  /system/lib/libart.so (art::ConditionVariable::Wait(art::Thread*)+80)  
  native: #02 pc 00335353  /system/lib/libart.so (art::Thread::FullSuspendCheck()+838)  
  native: #03 pc 0011d3a7  /system/lib/libart.so (art::ClassLinker::LoadClassMembers(art::Thread*, art::DexFile const&, unsigned char const*, art::Handle<art::mirror::Class>, art::OatFile::OatClass const*)+746)  
  native: #04 pc 0011d81d  /system/lib/libart.so (art::ClassLinker::LoadClass(art::Thread*, art::DexFile const&, art::DexFile::ClassDef const&, art::Handle<art::mirror::Class>)+88)  
  native: #05 pc 00132059  /system/lib/libart.so (art::ClassLinker::DefineClass(art::Thread*, char const*, unsigned int, art::Handle<art::mirror::ClassLoader>, art::DexFile const&, art::DexFile::ClassDef const&)+320)  
  native: #06 pc 001326c1  /system/lib/libart.so (art::ClassLinker::FindClassInPathClassLoader(art::ScopedObjectAccessAlreadyRunnable&, art::Thread*, char const*, unsigned int, art::Handle<art::mirror::ClassLoader>, art::mirror::Class**)+688)  
  native: #07 pc 002cb1a1  /system/lib/libart.so (art::VMClassLoader_findLoadedClass(_JNIEnv*, _jclass*, _jobject*, _jstring*)+264)  
  native: #08 pc 002847fd  /data/dalvik-cache/arm/system@framework@boot.oat (Java_java_lang_VMClassLoader_findLoadedClass__Ljava_lang_ClassLoader_2Ljava_lang_String_2+112)
  at java.lang.VMClassLoader.findLoadedClass!(Native method)  
  at java.lang.ClassLoader.findLoadedClass(ClassLoader.java:362)  
  at java.lang.ClassLoader.loadClass(ClassLoader.java:499)  
  at java.lang.ClassLoader.loadClass(ClassLoader.java:469)  
  at android.app.ActivityThread.installProvider(ActivityThread.java:5141)  
  at android.app.ActivityThread.installContentProviders(ActivityThread.java:4748)  
  at android.app.ActivityThread.handleBindApplication(ActivityThread.java:4688)  
  at android.app.ActivityThread.-wrap1(ActivityThread.java:-1)  
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1405)  
  at android.os.Handler.dispatchMessage(Handler.java:102)
  at android.os.Looper.loop(Looper.java:148)  
  at android.app.ActivityThread.main(ActivityThread.java:5417)  
  at java.lang.reflect.Method.invoke!(Native method)  
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)  
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)    

  ...  
  Stacks for other threads in this process follow  
  ...  
```

从上面可以看出是主线程中做了数据库相关的操作，导致了ANR。   

#### **发现死锁**  
死锁常常表现为ANR，因为线程被阻塞。如果死锁触及到系统服务，watchdog最终会kill掉它，在log中会有一条类似**WATCHDOG KILLING SYSTEM PROCESS**的语句。从用户的角度看，设备会重启，虽然从技术上是运行时重启不是真正的重启。  

- 运行时重启，系统服务死了并且重启，用户看到设备返回到启动动画
- 重启，内核crash了，用户看到设备返回到Google的启动log  

为了查找死锁，检查堆栈信息查找类似这样的格式： Thread A等待Thread B持有的东西，Thread B等待线程A等待的东西。       

```
"Binder_B" prio=5 tid=73 Blocked  
  | group="main" sCount=1 dsCount=0 obj=0x13faa0a0 self=0x95e24800  
  | sysTid=2016 nice=0 cgrp=default sched=0/0 handle=0x8b68d930  
  | state=S schedstat=( 9351576559 4141431119 16920 ) utm=819 stm=116 core=1 HZ=100  
  | stack=0x8b591000-0x8b593000 stackSize=1014KB  
  | held mutexes=
  at com.android.server.pm.UserManagerService.exists(UserManagerService.java:387)  
  ** - waiting to lock <0x025f9b02> (a android.util.ArrayMap) held by thread 20**    
  at com.android.server.pm.PackageManagerService.getApplicationInfo(PackageManagerService.java:2848)  
  at com.android.server.AppOpsService.getOpsRawLocked(AppOpsService.java:881)  
  at com.android.server.AppOpsService.getOpsLocked(AppOpsService.java:856)  
  at com.android.server.AppOpsService.noteOperationUnchecked(AppOpsService.java:719)  
  - locked <0x0231885a> (a com.android.server.AppOpsService)  
  at com.android.server.AppOpsService.noteOperation(AppOpsService.java:713)  
  at com.android.server.AppOpsService$2.getMountMode(AppOpsService.java:260)  
  at com.android.server.MountService$MountServiceInternalImpl.getExternalStorageMountMode(MountService.java:3416)  
  at com.android.server.am.ActivityManagerService.startProcessLocked(ActivityManagerService.java:3228)  
  at com.android.server.am.ActivityManagerService.startProcessLocked(ActivityManagerService.java:3170)  
  at com.android.server.am.ActivityManagerService.startProcessLocked(ActivityManagerService.java:3059)  
  at com.android.server.am.BroadcastQueue.processNextBroadcast(BroadcastQueue.java:1070)  
  - locked <0x044d166f> (a com.android.server.am.ActivityManagerService)  
  at com.android.server.am.ActivityManagerService.finishReceiver(ActivityManagerService.java:16950)  
  at android.app.ActivityManagerNative.onTransact(ActivityManagerNative.java:494)  
  at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2432)  
  at android.os.Binder.execTransact(Binder.java:453)  
...  
  "PackageManager" prio=5 tid=20 Blocked  
  | group="main" sCount=1 dsCount=0 obj=0x1304f4a0 self=0xa7f43900  
  | sysTid=1300 nice=10 cgrp=bg_non_interactive sched=0/0 handle=0x9fcf9930  
  | state=S schedstat=( 26190141996 13612154802 44357 ) utm=2410 stm=209 core=2 HZ=100  
  | stack=0x9fbf7000-0x9fbf9000 stackSize=1038KB  
  | held mutexes=
  at com.android.server.AppOpsService.noteOperationUnchecked(AppOpsService.java:718)  
  **- waiting to lock <0x0231885a> (a com.android.server.AppOpsService) held by thread 73**  
  at com.android.server.AppOpsService.noteOperation(AppOpsService.java:713)  
  at com.android.server.AppOpsService$2.getMountMode(AppOpsService.java:260)  
  at com.android.server.AppOpsService$2.hasExternalStorage(AppOpsService.java:273)  
  at com.android.server.MountService$MountServiceInternalImpl.hasExternalStorage(MountService.java:3431)  
  at com.android.server.MountService.getVolumeList(MountService.java:2609)  
  at android.os.storage.StorageManager.getVolumeList(StorageManager.java:880)  
  at android.os.Environment$UserEnvironment.getExternalDirs(Environment.java:83)  
  at android.os.Environment.isExternalStorageEmulated(Environment.java:708)  
  at com.android.server.pm.PackageManagerService.isExternalMediaAvailable(PackageManagerService.java:9327)  
  at com.android.server.pm.PackageManagerService.startCleaningPackages(PackageManagerService.java:9367)  
  - locked <0x025f9b02> (a android.util.ArrayMap)  
  at com.android.server.pm.PackageManagerService$PackageHandler.doHandleMessage(PackageManagerService.java:1320)  
  at com.android.server.pm.PackageManagerService$PackageHandler.handleMessage(PackageManagerService.java:1122)  
  at android.os.Handler.dispatchMessage(Handler.java:102)  
  at android.os.Looper.loop(Looper.java:148)  
  at android.os.HandlerThread.run(HandlerThread.java:61)  
  at com.android.server.ServiceThread.run(ServiceThread.java:46)  
```

### **Activity**   

Activity是提供一个界面与用户交互的组件，比如拨打电话，拍照，发邮件等等，从bug报告的角度看，Activity是一个用户能够聚焦的事物，在crash点那个Activity获取了焦点是很重要。Activity是运行在进程中的，所以定位到启动，停止特定Activity的所有进程也有助于故障的排除。  

#### **查看聚焦的Activity**  

要查看聚焦的Activity可以搜索**am_focused_activity**  
```
grep "am_focused_activity"  bugreport-2015-10-01-18-13-48.txt  
10-01 18:10:41.409  4600 14112 I am_focused_activity: [0,com.google.android.GoogleCamera/com.android.camera.CameraActivity]  
10-01 18:11:17.313  4600  5687 I am_focused_activity: [0,com.google.android.googlequicksearchbox/com.google.android.launcher.GEL]   
10-01 18:11:52.747  4600 14113 I am_focused_activity: [0,com.google.android.GoogleCamera/com.android.camera.CameraActivity]  
10-01 18:14:07.762  4600  5687 I am_focused_activity: [0,com.google.android.googlequicksearchbox/com.google.android.launcher.GEL]  
```

####  **查看进程的启动**  

为了去查看进程启动的记录，可以搜索**Start proc**  

```
 grep "Start proc" bugreport-2015-10-01-18-13-48.txt
10-01 18:09:15.309  4600  4612 I ActivityManager: Start proc 24533:com.metago.astro/u0a240 for broadcast com.metago.astro/com.inmobi.commons.analytics.androidsdk.IMAdTrackerReceiver  
10-01 18:09:15.687  4600 14112 I ActivityManager: Start proc 24548:com.google.android.apps.fitness/u0a173 for service com.google.android.apps.fitness/.api.services.ActivityUpsamplingService  
10-01 18:09:15.777  4600  6604 I ActivityManager: Start proc 24563:cloudtv.hdwidgets/u0a145 for broadcast cloudtv.hdwidgets/cloudtv.switches.SwitchSystemUpdateReceiver  
10-01 18:09:20.574  4600  6604 I ActivityManager: Start proc 24617:com.wageworks.ezreceipts/u0a111 for broadcast com.wageworks.ezreceipts/.ui.managers.IntentReceiver  
...  
```

#### **设备系统波动**    
为了判断设备系统是不是[thrashing](https://en.wikipedia.org/wiki/Thrashing_(computer_science))，检测是不是短时间大量的进程被杀（am_proc_died） 和进程启动（am_proc_start）

```
grep -e "am_proc_died" -e "am_proc_start" bugreport-2015-10-01-18-13-48.txt
10-01 18:07:06.494  4600  9696 I am_proc_died: [0,20074,com.android.musicfx]  
10-01 18:07:06.555  4600  6606 I am_proc_died: [0,31166,com.concur.breeze]  
10-01 18:07:06.566  4600 14112 I am_proc_died: [0,18812,com.google.android.apps.fitness]    
10-01 18:07:07.018  4600  7513 I am_proc_start: [0,20361,10113,com.sony.playmemories.mobile,broadcast,com.sony.playmemories.mobile/.service.StartupReceiver]  
10-01 18:07:07.357  4600  4614 I am_proc_start: [0,20381,10056,com.google.android.talk,service,com.google.android.talk/com.google.android.libraries.hangouts.video.CallService]  
10-01 18:07:07.784  4600  4612 I am_proc_start: [0,20402,10190,com.andcreate.app.trafficmonitor:loopback_measure_serivce,service,com.andcreate.app.trafficmonitor/.loopback.LoopbackMeasureService]  
10-01 18:07:10.753  4600  5997 I am_proc_start: [0,20450,10097,com.amazon.mShop.android.shopping,broadcast,com.amazon.mShop.android.shopping/com.amazon.identity.auth.device.storage.LambortishClock$ChangeTimestampsBroadcastReceiver]  
10-01 18:07:15.267  4600  6605 I am_proc_start: [0,20539,10173,com.google.android.apps.fitness,service,com.google.android.apps.fitness/.api.services.ActivityUpsamplingService]  
10-01 18:07:15.985  4600  4612 I am_proc_start: [0,20568,10022,com.android.musicfx,broadcast,com.android.musicfx/.ControlPanelReceiver]  
10-01 18:07:16.315  4600  7512 I am_proc_died: [0,20096,com.google.android.GoogleCamera]  
```

### **内存**  

因为Android设备只有有限的物理内存，因此管理RAM是至关重要的，Bug报告中包含一些低内存的提示和提供了内存快照的dumpstate   

#### **识别低内存**  

低内存导致系统的波动，为了启动其他进程不得不去杀其他进程。为了找到对应的证据可以搜索event log的**am_proc_died**和**am_proc_start**标签。  

低内存拖慢任务的切换而且会阻止返回（因为用户尝试去返回的任务可能被杀了）。如果launcher被杀，它会在用户触摸主页按钮的时候重新启动，并且日志会显示launcher重新加载其内容。   

#### **查看历史信息**  
在event log中当出现**am_low_memory**时候表示最后的缓存进程已经被杀了。这之后，系统会启动被杀的服务。

```
grep "am_low_memory" bugreport-2015-10-01-18-13-48.txt  
10-01 18:11:02.219  4600  7513 I am_low_memory: 41  
10-01 18:12:18.526  4600 14112 I am_low_memory: 39  
10-01 18:12:18.874  4600  7514 I am_low_memory: 38  
10-01 18:12:22.570  4600 14112 I am_low_memory: 40  
10-01 18:12:34.811  4600 20319 I am_low_memory: 43  
10-01 18:12:37.945  4600  6521 I am_low_memory: 43  
10-01 18:12:47.804  4600 14110 I am_low_memory: 43  
```

#### **查看系统波动指标**

其他系统波动的指标(分页，直接回收等等)包括**kswapd**,**kworker**，**mmcqd** 的消耗回收（log收集的时候也可能导致波动的指标）    

```
------ CPU INFO (top -n 1 -d 1 -m 30 -t) ------

User 15%, System 54%, IOW 28%, IRQ 0%  
User 82 + Nice 2 + Sys 287 + Idle 1 + IOW 152 + IRQ 0 + SIRQ 5 = 529  

  PID   TID PR CPU% S     VSS     RSS PCY UID      Thread          Proc  
15229 15229  0  19% R      0K      0K  fg root     kworker/0:2  
29512 29517  1   7% D 1173524K 101188K  bg u0_a27   Signal Catcher  com.google.android.talk  
24565 24570  3   6% D 2090920K 145168K  fg u0_a22   Signal Catcher  com.google.android.googlequicksearchbox:search  
19525 19525  2   6% R   3476K   1644K  fg shell    top             top  
24957 24962  2   5% R 1706928K 125716K  bg u0_a47   Signal Catcher  com.google.android.GoogleCamera  
19519 19519  3   4% S      0K      0K  fg root     kworker/3:1  
  120   120  0   3% S      0K      0K  fg root     mmcqd/1  
18233 18233  1   3% S      0K      0K  fg root     kworker/1:1  
25589 25594  1   2% D 1270476K  75776K  fg u0_a8    Signal Catcher  com.google.android.gms  
19399 19399  2   1% S      0K      0K  fg root     kworker/2:2  
 1963  1978  1   0% S 1819100K 125136K  fg system   android.fg      system_server  
 1963  1981  3   0% S 1819100K 125136K  fg system   android.display system_server    
```

ANR的log也能提供类似的内存快照  

```
10-03 17:19:59.959  1963  1976 E ActivityManager: ANR in com.google.android.apps.magazines  
10-03 17:19:59.959  1963  1976 E ActivityManager: PID: 18819   
10-03 17:19:59.959  1963  1976 E ActivityManager: Reason: Broadcast of Intent { act=android.net.conn.CONNECTIVITY_CHANGE flg=0x4000010 cmp=com.google.android.apps.magazines/com.google.apps.dots.android.newsstand.appwidget.NewsWidgetProvider (has extras) }  
10-03 17:19:59.959  1963  1976 E ActivityManager: Load: 19.19 / 14.76 / 12.03  
10-03 17:19:59.959  1963  1976 E ActivityManager: CPU usage from 0ms to 11463ms later:  
10-03 17:19:59.959  1963  1976 E ActivityManager:   54% 15229/kworker/0:2: 0% user + 54% kernel  
10-03 17:19:59.959  1963  1976 E ActivityManager:   38% 1963/system_server: 14% user + 23% kernel / faults: 17152 minor 1073 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   11% 120/mmcqd/1: 0% user + 11% kernel  
10-03 17:19:59.959  1963  1976 E ActivityManager:   10% 2737/com.android.systemui: 4.7% user + 5.6% kernel / faults: 7211 minor 149 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   0.2% 1451/debuggerd: 0% user + 0.2% kernel / faults: 15211 minor 147 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   8.7% 6162/com.twofortyfouram.locale: 4% user + 4.7% kernel / faults: 4924 minor 260 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   6.1% 24565/com.google.android.googlequicksearchbox:search: 2.4% user + 3.7% kernel / faults: 2902 minor 129 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   6% 55/kswapd0: 0% user + 6% kernel  
10-03 17:19:59.959  1963  1976 E ActivityManager:   4.9% 18819/com.google.android.apps.magazines: 1.5% user + 3.3% kernel / faults: 10129 minor 986 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   2.8% 18233/kworker/1:1: 0% user + 2.8% kernel  
10-03 17:19:59.959  1963  1976 E ActivityManager:   4.2% 3145/com.android.phone: 2% user + 2.2% kernel / faults: 3005 minor 43 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   4.2% 8084/com.android.chrome: 2% user + 2.1% kernel / faults: 4798 minor 380 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   3.4% 182/surfaceflinger: 1.1% user + 2.3% kernel / faults: 842 minor 13 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   3% 18236/kworker/1:2: 0% user + 3% kernel  
10-03 17:19:59.959  1963  1976 E ActivityManager:   2.9% 19231/com.android.systemui:screenshot: 0.8% user + 2.1% kernel / faults: 6119 minor 348 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   2.3% 15350/kworker/0:4: 0% user + 2.3% kernel  
10-03 17:19:59.959  1963  1976 E ActivityManager:   2.2% 1454/mediaserver: 0% user + 2.2% kernel / faults: 479 minor 6 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   2% 16496/com.android.chrome:sandboxed_process10: 0.1% user + 1.8% kernel / faults: 3610 minor 234 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   1% 3119/com.android.nfc: 0.4% user + 0.5% kernel / faults: 1789 minor 17 major   
10-03 17:19:59.959  1963  1976 E ActivityManager:   1.7% 19337/com.jarettmillard.localeconnectiontype:background: 0.1% user + 1.5% kernel / faults: 7854 minor 439 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   0.7% 3066/com.google.android.inputmethod.latin: 0.3% user + 0.3% kernel / faults: 1336 minor 7 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   1% 25589/com.google.android.gms: 0.3% user + 0.6% kernel / faults: 2867 minor 237 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   0.9% 1460/sensors.qcom: 0.5% user + 0.4% kernel / faults: 262 minor 5 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   0.8% 3650/mpdecision: 0% user + 0.8% kernel / faults: 160 minor 1 major  
10-03 17:19:59.959  1963  1976 E ActivityManager:   0.1% 3132/com.redbend.vdmc: 0% user + 0% kernel / faults: 1746 minor 5 major  
```

#### **获取内存快照**   
dumpstate列出了运行的java和native进程的内存快照，记住快照只是某个特定时间点的，系统可能在时间点之前已经恢复了也可能更糟糕。  

- 为了理解进程运行的时间可以参考下面** Process runtime**一节   
- 为了明白现在哪些事物在运行请看**Why is a process running**  

使用**adb shell dumpstate**命令获取信息,或者通过**adb shell dumpsys meminfo**获取。  

```  
Total PSS by OOM adjustment:  
    86752 kB: Native  
               22645 kB: surfaceflinger (pid 197)  
               18597 kB: mediaserver (pid 204)  
               ...   
   136959 kB: System  
              136959 kB: system (pid 785)  
   220218 kB: Persistent  
              138859 kB: com.android.systemui (pid 947 / activities)  
               39178 kB: com.android.nfc (pid 1636)  
               28313 kB: com.android.phone (pid 1659)  
               13868 kB: com.redbend.vdmc (pid 1646)  
     9534 kB: Persistent Service
                9534 kB: com.android.bluetooth (pid 23807)   
   178604 kB: Foreground  
              168620 kB: com.google.android.googlequicksearchbox (pid 1675 / activities)
                9984 kB: com.google.android.apps.maps (pid 13952)    
   188286 kB: Visible  
               85326 kB: com.google.android.wearable.app (pid 1535)   
               38978 kB: com.google.process.gapps (pid 1510)   
               31936 kB: com.google.android.gms.persistent (pid 2072)   
               27950 kB: com.google.android.gms.wearable (pid 1601)  
                4096 kB: com.google.android.googlequicksearchbox:interactor (pid 1550)   
    52948 kB: Perceptible  
               52948 kB: com.google.android.inputmethod.latin (pid 1566)   
   150851 kB: A Services   
               81121 kB: com.google.android.gms (pid 1814)    
               37586 kB: com.google.android.talk (pid 9584)  
               10949 kB: com.google.android.music:main (pid 4019)  
               10727 kB: com.motorola.targetnotif (pid 31071)  
               10468 kB: com.google.android.GoogleCamera (pid 9984)  
    33298 kB: Previous
               33298 kB: com.android.settings (pid 9673 / activities)  
   165188 kB: B Services
               49490 kB: com.facebook.katana (pid 15035)  
               22483 kB: com.whatsapp (pid 28694)  
               21308 kB: com.iPass.OpenMobile (pid 5325)   
               19788 kB: com.google.android.apps.googlevoice (pid 23934)   
               17399 kB: com.google.android.googlequicksearchbox:search (pid 30359)   
                9073 kB: com.google.android.apps.youtube.unplugged (pid 21194)    
                7660 kB: com.iPass.OpenMobile:remote (pid 23754)   
                7291 kB: com.pujie.wristwear.pujieblack (pid 24240)   
                7157 kB: com.instagram.android:mqtt (pid 9530)   
                3539 kB: com.qualcomm.qcrilmsgtunnel (pid 16186)   
   204324 kB: Cached   
               43424 kB: com.amazon.mShop.android (pid 13558)   
               22563 kB: com.google.android.apps.magazines (pid 13844)   
               ...   
                4298 kB: com.google.android.apps.enterprise.dmagent (pid 13826)   
```

####  **进程**   
Bug报告中包含大量的进程信息，包括启动，停止时间，运行时长，关联的服务，oom_adj值等等。对于Android是怎么管理进程的，请参考[Processes and Threads](https://developer.android.com/guide/components/processes-and-threads.html)  
#### 确定进程的运行
**procstats** 部分包含了进程运行了多长时间，运行的关联的服务的完整数据。为了获取易读的摘要，可以搜索**AGGREGATED OVER**来浏览最近3小时或者24小时的数据。搜索的摘要：浏览进程列表，这些进程在各种优先事项中运行了多久，和RAM的使用情况，展示格式min-average-max PSS/min-average-max USS。  

>  Tips: 使用**adb shell dumpsys procstats**命令获取   


```
-------------------------------------------------------------------------------   
DUMP OF SERVICE processinfo:  
-------------------------------------------------------------------------------    
DUMP OF SERVICE procstats:  
COMMITTED STATS FROM 2015-10-19-23-54-56 (checked in):    
...    
COMMITTED STATS FROM 2015-10-20-03-00-00 (checked in):    
...    
CURRENT STATS:    
...    
AGGREGATED OVER LAST 24 HOURS:   
System memory usage:   
...   
Per-Package Stats:    
...   
Summary:   
...    
  * com.google.android.gms.persistent / u0a13 / v8186448:    
           TOTAL: 100% (21MB-27MB-40MB/20MB-24MB-38MB over 597)   
             Top: 51% (22MB-26MB-38MB/21MB-24MB-36MB over 383)   
          Imp Fg: 49% (21MB-27MB-40MB/20MB-25MB-38MB over 214)    
…   
          Start time: 2015-10-19 09:14:37   
  Total elapsed time: +1d0h22m7s390ms (partial) libart.so   

AGGREGATED OVER LAST 3 HOURS:    
System memory usage:  
...  
Per-Package Stats:  
...  
Summary:  
  * com.google.android.gms.persistent / u0a13 / v8186448:   
           TOTAL: 100% (23MB-27MB-32MB/21MB-25MB-29MB over 111)  
             Top: 61% (23MB-26MB-31MB/21MB-24MB-28MB over 67）  
          Imp Fg: 39% (23MB-28MB-32MB/21MB-26MB-29MB over 44)   
...   
          Start time: 2015-10-20 06:49:24  
  Total elapsed time: +2h46m59s736ms (partial) libart.so   
```

#### **为什么进程会运行?**

**adb shell dumpsys activity processes** 部分列出了所有当前正在运行的进程并且通过oom_adj值排序（Android 标记进程的重要性是通过分配给进程一个oom_adj值，这个值会通过ActivityManager动态的更新）。这个输出类似于 **memory snapshot** 但是会包含一些进程正在运行的原因。下面的例子中，黑色条目中表示 **gms.persistent** 进程运行在 **vis** （visible）优先级下是因为 **system** 进程绑定到了它的 **NetworkLocationService** 服务。  

```
-------------------------------------------------------------------------------  
ACTIVITY MANAGER RUNNING PROCESSES (dumpsys activity processes)  
...  
Process LRU list (sorted by oom_adj, 34 total, non-act at 14, non-svc at 14):  
    PERS #33: sys   F/ /P  trm: 0 902:system/1000 (fixed)  
    PERS #32: pers  F/ /P  trm: 0 2925:com.android.systemui/u0a27 (fixed)  
    PERS #31: pers  F/ /P  trm: 0 3477:com.quicinc.cne.CNEService/1000 (fixed)  
    PERS #30: pers  F/ /P  trm: 0 3484:com.android.nfc/1027 (fixed)  
    PERS #29: pers  F/ /P  trm: 0 3502:com.qualcomm.qti.rcsbootstraputil/1001 (fixed)  
    PERS #28: pers  F/ /P  trm: 0 3534:com.qualcomm.qti.rcsimsbootstraputil/1001 (fixed)  
    PERS #27: pers  F/ /P  trm: 0 3553:com.android.phone/1001 (fixed)  
    Proc #25: psvc  F/ /IF trm: 0 4951:com.android.bluetooth/1002 (service)  
        com.android.bluetooth/.hfp.HeadsetService<=Proc{902:system/1000}  
    Proc # 0: fore  F/A/T  trm: 0 3586:com.google.android.googlequicksearchbox/u0a29 (top-activity)  
  Proc #26: vis   F/ /SB trm: 0 3374:com.google.android.googlequicksearchbox:interactor/u0a29 (service)
        com.google.android.googlequicksearchbox/com.google.android.voiceinteraction.GsaVoiceInteractionService<=Proc{902:system/1000}   
    Proc # 5: vis   F/ /T  trm: 0 3745:com.google.android.gms.persistent/u0a12 (service)
        com.google.android.gms/com.google.android.location.network.NetworkLocationService<=Proc{902:system/1000}    
    Proc # 3: vis   F/ /SB trm: 0 3279:com.google.android.gms/u0a12 (service)
        com.google.android.gms/.icing.service.IndexService<=Proc{947:com.google.android.googlequicksearchbox:search/u0a29}  
    Proc # 2: vis   F/ /T  trm: 0 947:com.google.android.googlequicksearchbox:search/u0a29 (service)
        com.google.android.googlequicksearchbox/com.google.android.sidekick.main.remoteservice.GoogleNowRemoteService<=Proc{3586:com.google.android.googlequicksearchbox/u0a29}  
    Proc # 1: vis   F/ /T  trm: 0 2981:com.google.process.gapps/u0a12 (service)
        com.google.android.gms/.tapandpay.hce.service.TpHceService<=Proc{3484:com.android.nfc/1027}  
    Proc #11: prcp  B/ /IB trm: 0 3392:com.google.android.inputmethod.latin/u0a64 (service)
        com.google.android.inputmethod.latin/com.android.inputmethod.latin.LatinIME<=Proc{902:system/1000}  
    Proc #24: svc   B/ /S  trm: 0 27071:com.google.android.music:main/u0a67 (started-services)  
    Proc #22: svc   B/ /S  trm: 0 853:com.qualcomm.qcrilmsgtunnel/1001 (started-services)  
    Proc # 4: prev  B/ /LA trm: 0 32734:com.google.android.GoogleCamera/u0a53 (previous)  
    Proc #23: svcb  B/ /S  trm: 0 671:com.qualcomm.telephony/1000 (started-services)  
    Proc #20: cch   B/ /CE trm: 0 27659:com.android.providers.calendar/u0a2 (provider)
        com.android.providers.calendar/.CalendarProvider2<=Proc{27697:com.google.android.calendar/u0a40}
```
