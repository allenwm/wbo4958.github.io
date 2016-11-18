---                                                                                                                             
layout: post
category: Android
title: Home 应用的启动
tagline:
tags:  [home应用启动]
---
{% include JB/setup %}

> 这篇 blog 主要是记录自己对于Android AMS的理解

### 环境

**Env:** Android 7.0

涉及到的几个非常重要的类

- __ActivityManagerService (AMS)__

    这个Android 核心类，它运行在 SystemServer 中，管理 Android 的所有组件

- __ActivityStarter__

    Activity的启动控制器，它控制 Activity 的启动逻辑。可能会拦截不让其启动

- __ActivityStackSupervisor (ASS)__

    固名思义，它是 stack 的管理者

- __ActivityStack__

    表示一个具体的 stack

- __ActivityRecord__

    在AMS里 用来表示一个 Activity

- __TaskRecord__

    表示一个任务


**Home 应用启动**

<div align="center">
    <img src="assets\images\android\AMS\home_starting.png"
     alt="Home 应用启动"/>
</div>		

### 创建系统首个 Home Stack

``` java
startOtherServices()
    mActivityManagerService.setWindowManager(wm);

//ActivityStackSupervisor.java
void setWindowManager(WindowManagerService wm) {
    synchronized (mService) {
        mHomeStack = mFocusedStack = mLastFocusedStack =
                getStack(HOME_STACK_ID, CREATE_IF_NEEDED, ON_TOP);
    }
}
```

现在来看下mHomeStack, mFocusedStack, mLastFocusedStack的定义

``` java
 /** The stack containing the launcher app. Assumed to always be attached to
 * Display.DEFAULT_DISPLAY. */
ActivityStack mHomeStack;

/** The stack currently receiving input or launching the next activity. */
ActivityStack mFocusedStack;

/** If this is the same as mFocusedStack then the activity on the top of the focused stack has
 * been resumed. If stacks are changing position this will hold the old stack until the new
 * stack becomes resumed after which it will be set to mFocusedStack. */
private ActivityStack mLastFocusedStack;
```

mHomeStack是包含有launcher app, HOME应用程序的栈，它的stack id 是HOME_STACK_ID定义为0

为什么需要在这里初始化home stack呢？
因为马上要启动home应用了。

### 获得HOME Activity 信息

``` java
Intent intent = getHomeIntent(userId);
ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
```

在 Android 7.0 里，AOSP里包含了两个HOME 应用

- FallbackHome.java

``` xml
<activity android:name=".FallbackHome"
          android:excludeFromRecents="true"
          android:theme="@style/FallbackHome">                                                                                                               
    <intent-filter android:priority="-1000">
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.HOME" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

- Launcher.java

```xml
<activity
    android:name="com.android.launcher3.Launcher"
    android:launchMode="singleTask"
    android:clearTaskOnLaunch="true"
    android:stateNotNeeded="true"
    android:theme="@style/Theme"
    android:windowSoftInputMode="adjustPan"
    android:screenOrientation="nosensor"
    android:configChanges="keyboard|keyboardHidden|navigation"
    android:resumeWhilePausing="true"
    android:taskAffinity=""                                                                                                                                  
    android:enabled="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.HOME" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.MONKEY"/>
    </intent-filter>
</activity>
```

Launcher.java 是默认优先级，而 FallbackHome 是-1000优先级，数值越小，它的优先级越低。

Android N 里，因为一些flag的原因, 这里并不去做深究(具体可以在 resolveIntent-> updateFlagsForResolve查找)

getHomeIntent 第一次会找到 Settings里的FallbackHome，该Home Activity应该只是一个过度的Home 界面，它会去监听
ACTION_USER_UNLOCKED, 具体可以参考

[Supporting Direct Boot](https://developer.android.com/training/articles/direct-boot.html)

[Android N "直接启动"](http://sanwen8.cn/p/1b4VO6X.html)

FallbackHome接受到这个广播时，会把自己给 finish掉。

然后AMS 检测到系统里ua没有更多的activity, 这时又会重新调用 startHomeActivity, 这时会找到两个，
一个是 FallbackHome, 一个是Launcher， Launcher的priority 是0，FallbackHome的priority是-1000,
所以这时会启动 Launcher, 这时候 Launcher才正式启动起来.

FallbackHome 仅是一个过度的 Home Activity, 所以以下都是基于第二次 home 应用启动来写的。

### 生成 ActivityRecord, 创建 TaskRecord, 建立 TaskRecord 与 ActivityRecord之间的联系

- 在**startActivityLocked** 里会生成Launcher对应的ActivityRecord

``` java
ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, mSupervisor, container,
                options, sourceRecord);
```

- computeStackFocus

计算使用哪个栈进行TaskRecord 的创建, 这里会直接返回的是Home Stack

- createTaskRecord

在找到的home stack中创建TaskRecord, 并建立Stack/TaskRecord/ActivityRecord 之间的联系

既然 Stack/Task 已经建立好了，那么就可以在 Home stack 启动 Launcher了

### 通知 Zygnote 创建进程并运行 ActivityThread

``` java
resumeFocusedStackTopActivityLocked()
  resumeTopActivityUncheckedLocked()
    resumeTopActivityInnerLocked()
      startSpecificActivityLocked()
        startProcessLocked()
```

通知zygnote fork一个新的进程，并开始运行 ActivityThread.main()

此时新进程里并没有运行任何和Launcher有关的代码，仅仅是ActivityThread 的代码.

ActivityThread 通过binder将自己attach到AMS, 这样AMS就可以方便调度管理对应的app进程了。

### attachApplication/bindApplication

应用进程将自己的ApplicationThread 的binder attach给AMS, 然后AMS将Launcher 的相关信息
bind给应用程序

``` java
thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
        profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
        app.instrumentationUiAutomationConnection, testMode,
        mBinderTransactionTrackingEnabled, enableTrackAllocation,
        isRestrictedBackupMode || !normalMode, app.persistent,
        new Configuration(mConfiguration), app.compat,
        getCommonServicesLocked(app.isolated),
        mCoreSettingsObserver.getCoreSettingsLocked());
```

应用进程在接受到这些信息后，才将自己设置为Launcher进程, 此刻才是真正的 Launcher进程了。
然后创建 Application 实例，并进入Application onCreate() 生命周期

### realStartActivityLocked

AMS继续启动真正的Activity了, 通知Launcher进程启动Launcher Activity

``` java
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
        System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
        new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
        task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
        newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
```

Launcher 进程接受到AMS的指令后，

- 通过反射机制生成一个Launcher Activity实例，
- 并进入Activity的生命周期onCreate onPostCreate,
- 进入Activity 的onResume 生命周期
- 准备调度ActivityThread  Idler (当ActivityThread没有事件处理时，就调度 Idler)
- Idler 通知AMS activityIdle

这样，Launcher activity就启动成功了

### 发送 BOOT_COMPLETED 广播

AMS在收到activityIdle后，开始进入系统启动的最后阶段,

``` java
activityIdle()
  activityIdleInternalLocked()
    checkFinishBootingLocked()

      finishBooting()
```

然后发送ACTION_USER_UNLOCKED ACTION_BOOT_COMPLETED 广播
系统启动完成
