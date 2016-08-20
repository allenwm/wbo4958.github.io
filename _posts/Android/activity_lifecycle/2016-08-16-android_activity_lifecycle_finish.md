---                                                                                                                             
layout: post
category: Android
title: Activity生命周期之finish
tagline:
tags:  [activity lifecyle]
---
{% include JB/setup %}

这篇文章讲述 finish activity的整个过程. 通篇文章基于如下场景

1. Launcher打开HelloWorld app, 这时MainActivity获得焦点并显示.
2. 按 back 键回到Launcher界面.

需要注意的是MainActivity里并没有override onBackPressed或onKeyDown, 这样back键就不会被拦截。

## 1. back key event 的传递
``` java
 -> dispatchInputEventFinished(InputEventSender)
 -> onInputEventFinished(InputMethodManager)
 ...
    -> dispatchKeyEvent (PhoneWindow@DecorView)
    -> dispatchKeyEvent (Activity)
      -> dispatch (Activity)
       -> onKeyDown (Activity)
        -> onBackPressed (Activity)
         -> finishAfterTransition (Activity)
          -> finish (Activity) ->
           -> finish(false) (Activity)
            -> finishActivity (ActivityManagerService)
```
其中 finish(false), 其中false 表示是否直接移掉activity所在的task.
当值为false时，需要请求finish activity.

## 2. 整个流程图如下所示

<div align="center" id="flow_chart"><img src="/assets/images/android/activitylifecycle/Launcher_start_helloworld_and_press_back.png" alt="activity lifecycle-finish"/></div>

图2.1 activity finish流程

> ActivityStack/AMS/ASS 属于 system 进程  
> MainActivity/ActivityThread 属于HelloWorld进程 <br>
> Launcher/ActivityThread 属于 Launcher 进程

因此在此例 finish activity中涉及到三个进程间的通信.
另外HelloWorld是由AMS里的非 HomeStack 的那个栈管理的， 这里命令为 Non-HomeStack, 而Launcher是Home应用，它运行在HomeStack中。

## 3. Activity生命周期转换
**3.1 PAUSING**

AMS通知HelloWorld所在的stack开始pausing MainActivity, 调用 startPausingLocked, 然后 System 就退出就不管了，等着MainActivity pause完成通知再继续完成下一步。
当HelloWorld完成Pause后，通知 AMS  activityPaused, 让AMS 继续进行接下来的工作。

**3.2 PAUSED/STOPPING**

<span id="paused_stopping">
当AMS 收到 activityPaused通知后，表明MainActivity已经Paused了， 这时就需要将MainActivity在AMS 里的状态置为PAUSED</span>

```java
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj) {
       // First things first: if this activity is currently visible,
       // and the resumed activity is not yet visible, then hold off on
       // finishing until the resumed one becomes visible.
       if (mode == FINISH_AFTER_VISIBLE && r.nowVisible) {
           if (!mStackSupervisor.mStoppingActivities.contains(r)) {
               mStackSupervisor.mStoppingActivities.add(r);
               if (mStackSupervisor.mStoppingActivities.size() > 3
                       || r.frontOfTask && mTaskHistory.size() <= 1) {
                   // If we already have a few activities waiting to stop,
                   // then give up on things going idle and start clearing
                   // them out. Or if r is the last of activity of the last task the stack
                   // will be empty and must be cleared immediately.
                   mStackSupervisor.scheduleIdleLocked();
               } else {
                   mStackSupervisor.checkReadyForSleepLocked();
               }
           }
           r.state = ActivityState.STOPPING;
           return r;
       }
       ...
   }
   ...
}
```

在finishCurrentActivityLocked函数中， 如果当前等待 stopping的 activity比较多，或者是最后的一个task里的最后一个activity(本文的例子满足此条件)，就会放弃自动进入idle, 而直接调用 scheduleIdleLocked 进入IDLE处理。然后将MainActivity的状态置为 STOPPING.

scheduleIdleLocked 通过Handler将操作放到 ActivityManager 线程中去操作

```
void activityIdleInternal(ActivityRecord r) {
    synchronized (mService) {
        activityIdleInternalLocked(r != null ? r.appToken : null, true, null);
    }
}
```
真正的处理函数 activityIdleInternal 会等待 mService 同步锁，而该锁正被最外侧的finishActivity持有，因此会一直等到  finishActivity 结束后才调用。

**3.3 RESUMED**

completePauseLocked会继续去resume top的activity. 但是需要注意的是，当前的focused stack还是Non-HomeStack.  但是该栈中已经没有别的activity了，所以直接 resumeHomeStackTask了，接着就是将HomeStack移动到top, 将Launcher移动到最前. 然后通知 Launcher 进程进行 resume. 并进入Launcher的  onStart-> onResume的生命周期。

**3.4 FINISHING/DESTROYING**

在AMS 调用完 scheduleResumeActivity 后，很快AMS就退出了 activityPausedLocked函数进而退出了 finishActivity. 然后 activityIdleInternal才开始调用(见[3.2](#paused_stopping)), 最后调用到 finishCurrentActivityLocked里。

```java
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj) {
    r.state = ActivityState.FINISHING;  //将MainActivity的状态置为 FINISHING

    if (mode == FINISH_IMMEDIATELY
            || (mode == FINISH_AFTER_PAUSE && prevState == ActivityState.PAUSED)
            || prevState == ActivityState.STOPPED
            || prevState == ActivityState.INITIALIZING) {
        // If this activity is already stopped, we can just finish
        // it right now.
        r.makeFinishingLocked();
        boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm");
        return activityRemoved ? null : r;
    }
}
```
destroyActivityLocked会通知HelloWorld进行Activity的Destroy, 即进入MainActivity的 onStop-> onDestroy.

**3.5 DESTROYED**

MainActivity在Destroy后会通知 AMS 进行后续处理(activityDestroyed).

```java
final void activityDestroyedLocked(IBinder token, String reason) {
    try {
        ActivityRecord r = ActivityRecord.forTokenLocked(token);
        if (isInStackLocked(r) != null) {
            if (r.state == ActivityState.DESTROYING) {
                cleanUpActivityLocked(r, true, false);
                removeActivityFromHistoryLocked(r, reason);
            }
        }
        mStackSupervisor.resumeTopActivitiesLocked();
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}

private void removeActivityFromHistoryLocked(ActivityRecord r, String reason) {

    r.state = ActivityState.DESTROYED; //设置Activity的状态为 DESTROYED
    r.app = null;   //清除掉 activity 与 processrecord的关系
    mWindowManager.removeAppToken(r.appToken);

    final TaskRecord task = r.task;
    if (task != null && task.removeActivity(r)) {  //从task里移除掉 MainActivity
        removeTask(task, reason);  //最后一个activity, 然后删除 task, 发现是最后一个task, 那么将stack也删掉了。
    }
    cleanUpActivityServicesLocked(r);
    r.removeUriPermissionsLocked();
}
```

## 4. onResume/ onStop onDestroy的时间顺序
很多关于 Activity Lifecycle 的文章都是讲的  onResume会在onStop/onDestroy的前面。其实这是不正确的，比如上例中当 finish掉HelloWorld的MainActivity的时候，在resume Launcher的时候，Activity的生命周期就有可能是 onStop/onDestroy在 onResume之前。


因为这里涉及的是三个进程的并发执行问题, 如 [图2.1](#flow_chart)中，
AMS 在调用完 24- scheduleResumeActivity后就继续往下执行了，而真正的Launcher的resume是在 Launcher进程进行的。
AMS 继续执行会很快调用 34- scheduleDestroyActivity, 然后将MainActivity的finish交给HelloWorld进程执行.


从调用时隙上， 理论上Launcher进行会先resume Launcher, 然后再destroy  MainActivity. 但是在多处理器上，以及进程调度上，也不一定launcher一定会比helloworld进程执行的早。


本人在debug的时候就遇到过多次 MainActivity的 onStop/onDestroy 先成 Launcher 的onResume的。
