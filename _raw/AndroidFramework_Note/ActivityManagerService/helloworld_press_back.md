这篇文章讲述 finish activity.
考虑如下场景
1. Launcher打开HelloWorld app, 这时MainActivity获得焦点并显示
2. 按 back 键回到Launcher界面
注意 MainActivity里并没有override onBackPressed或onKeyDown, 这样back键就不会被拦截。

下面来看下 keyevent 的传递 stack
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


```java
public final boolean finishActivity(IBinder token, int resultCode, Intent resultData,
           boolean finishTask) {

    ActivityRecord r = ActivityRecord.isInStackLocked(token);
    // 根据 token 找到 MainActivity 的ActivityRecord
    TaskRecord tr = r.task;
    //获得 MainActivity所在的Task.

    res = tr.stack.requestFinishActivityLocked(token, resultCode,
                       resultData, "app-request", true);
    // 然后在stack中请求 finish 掉activity
}
```

``` java
final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
        String reason, boolean oomAdj) {

    //如果Activity已经被finish掉了，那么就直接返回
    if (r.finishing) {
        Slog.w(TAG, "Duplicate finish request for " + r);
        return false;
    }
    r.makeFinishingLocked(); //设置 ActivityRecord的finishing 为true, 表明正在进行finishing

    final TaskRecord task = r.task;
    final ArrayList<ActivityRecord> activities = task.mActivities;
    final int index = activities.indexOf(r);
    if (index < (activities.size() - 1)) {
        task.setFrontOfTask();  // 如果当前正要被finish掉的 Activity不是前端task时，将该task置为前端task
    }

    // pause 按键颁发
    r.pauseKeyDispatchingLocked();

    // 调整 focused activity, 因为MainActivity 正在finish，所以需要调整focused activity
    adjustFocusedActivityLocked(r, "finishActivity");

    //重置 activity的 result
    finishActivityResultsLocked(r, resultCode, resultData);

    // 因为当前是 MainActivity 获得的焦点，这里会进入这个分支
    if (mResumedActivity == r) {
        boolean endTask = index <= 0;

        //当前没有正在 pausing的 activity, 那么就可以直接pause MainActivity 了
        if (mPausingActivity == null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Finish needs to pause: " + r);

            // 开始 pause MainActivity
            startPausingLocked(false, false, false, false);
        }

        //由于要finish的activity 是TaskRecord里最后一个activity, 这里会remove locked task
        if (endTask) {
            mStackSupervisor.removeLockedTaskLocked(task);
        }
    } else if (r.state != ActivityState.PAUSING) {
        // If the activity is PAUSING, we will complete the finish once
        // it is done pausing; else we can just directly finish it here.
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Finish not pausing: " + r);
        return finishCurrentActivityLocked(r, FINISH_AFTER_PAUSE, oomAdj) == null;
    } else {
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Finish waiting for pause of: " + r);
    }

    return false;
}
```

接下来看下 startPausingLocked 这个函数的实现。
先看下函数的注释

```java
/**
   * Start pausing the currently resumed activity.  It is an error to call this if there
   * is already an activity being paused or there is no resumed activity.
   *
   * @param userLeaving True if this should result in an onUserLeaving to the current activity.
   * @param uiSleeping True if this is happening with the user interface going to sleep (the
   * screen turning off).
   * @param resuming True if this is being called as part of resuming the top activity, so
   * we shouldn't try to instigate a resume here.
   * @param dontWait True if the caller does not want to wait for the pause to complete.  If
   * set to true, we will immediately complete the pause here before returning.
   * @return Returns true if an activity now is in the PAUSING state, and we are waiting for
   * it to tell us when it is done.
   */
```
可知，startPausingLocked的目的是 pause 当前resumed的activity. 如果已经有一个activity已经paused，或者系统里
根本没有 resumed activity， 这时调用这个函数就会有问题。
其中dontWait 是一个boolean的变量，是否等待 pause 完成才退出。

```java
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        if (mPausingActivity != null) {
            // 是pending pausing的 activity. 那么继续 pause 该activity
        }

        // mResumedActivity 当前还是 MainActivity
        ActivityRecord prev = mResumedActivity;
        mResumedActivity = null;
        mPausingActivity = prev; //pausing Activity还是MainActivity
        mLastPausedActivity = prev;
        // 保存一些状态变量， 既然已经在pausing一个activity了，那么这里自然应该将 mResumedActivity置为null

        prev.state = ActivityState.PAUSING;  //将要pausing的 activity的状态置为 PAUSING
        prev.task.touchActiveTime();  //更新 activity的 active时间

        if (prev.app != null && prev.app.thread != null) {
            try {
                //开始pause activity
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        }

        if (mPausingActivity != null) {
            // 传过来的 dontWait 为false
            if (dontWait) {
                completePauseLocked(false);
                return false;
            } else {
                // 等待500 ms等待 Pause完成
                Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
                msg.obj = prev;   //这个是表明哪个 prev正在pause
                prev.pauseTime = SystemClock.uptimeMillis();
                mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
                return true;
            }
        }
    }
```

```java
public final void schedulePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    sendMessage(
            finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
            token,
            (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
            configChanges);
}
```
schedulePauseActivity会根据当前的 finished 状态来决定是 PAUSE_ACTIVITY 还是 PAUSE_ACTIVITY_FINISHING

这里由AMS传过来的是 finished 为true. 所以会进入PAUSE_ACTIVITY_FINISHING

```java
private void handlePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        //如果是  userLeaving, 进入 onUserLeaving
        if (userLeaving) {
            performUserLeavingActivity(r);
        }

        performPauseActivity(token, finished, r.isPreHoneycomb());

        // Tell the activity manager we have paused.
        if (!dontReport) {
            try {
                // 告知 AMS，该 activity已经 paused了
                ActivityManagerNative.getDefault().activityPaused(token);
            } catch (RemoteException ex) {
            }
        }
        mSomeActivitiesChanged = true;
    }
}
```
执行真正的 paused 函数

```java
final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
        boolean saveState) {
    // 设置 activity finished 状态
    if (finished) {
        r.activity.mFinished = true;
    }
    try {
        // Next have the activity save its current state and managed dialogs...
        // 进入 onSavedInstanceState
        if (!r.activity.mFinished && saveState) {
            callCallActivityOnSaveInstanceState(r);
        }
        // Now we are idle.
        r.activity.mCalled = false;
        //进入MainActivity的 onPause 生命周期
        mInstrumentation.callActivityOnPause(r.activity);

        // override的 onPause 必须调用 super.onPause; 否则会报错
        if (!r.activity.mCalled) {
            throw new SuperNotCalledException(
                "Activity " + r.intent.getComponent().toShortString() +
                " did not call through to super.onPause()");
        }
    }
    r.paused = true;
    return !r.activity.mFinished && saveState ? r.state : null;
}
```
当MainActivity pause后，就通知 AMS

```java
public final void activityPaused(IBinder token) {
    synchronized(this) {
        // 通过 token 找到ActivityRecord，这里是MainActivity, 然后找到其所在的stack
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            // 开始进入 MainActivity 所在Stack的 activityPausedLocked, 这个 stack 是非Home stack的那个stack
            stack.activityPausedLocked(token, false);
        }
    }
}
```

```java
final void activityPausedLocked(IBinder token, boolean timeout) {
       // 通过 token找到MainActivity的ActivityRecord
       final ActivityRecord r = isInStackLocked(token);
       if (r != null) {
           // AMS 收到了 Activity  pause的回应，这时就需要将 PAUSE_TIMEOUT_MSG 删除掉
           mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
           if (mPausingActivity == r) {
               //当前正在pausing  MainActivity
               if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                       + (timeout ? " (due to timeout)" : " (pause complete)"));
              completePauseLocked(true);
           }
       }
   }
```

完成 Pause

```java
private void completePauseLocked(boolean resumeNext) {
     ActivityRecord prev = mPausingActivity; //当前正在pausing的 activity, 即 MainActivity
     if (prev != null) {
         prev.state = ActivityState.PAUSED;  //置为 PAUSED状态
         if (prev.finishing) {
             // 开始真正的finish 当前的activity
             prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false);
         }
         // Pausing 过程已经完成了，将Pausing activity 置空
         mPausingActivity = null;
     }

     // pausing完activity 需要立即 resume 下一个
     if (resumeNext) {
         final ActivityStack topStack = mStackSupervisor.getFocusedStack();
         // 获得focused的 stack
         if (!mService.isSleepingOrShuttingDown()) { //是否是sleeping 或是已经shutdown了
             // 开始resume top activity
             mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
         }
     }
     ...
 }
```

```java
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj) {
    // First things first: if this activity is currently visible,
    // and the resumed activity is not yet visible, then hold off on
    // finishing until the resumed one becomes visible.
    // 如果正在finish的activity还可见，那么要等到resumed activity可见
    if (mode == FINISH_AFTER_VISIBLE && r.nowVisible) {
        if (!mStackSupervisor.mStoppingActivities.contains(r)) {
            mStackSupervisor.mStoppingActivities.add(r);  //加入到 mStoppingActivities 里
            if (mStackSupervisor.mStoppingActivities.size() > 3
                || r.frontOfTask && mTaskHistory.size() <= 1) { //这里满足了第二个条件
                    // If we already have a few activities waiting to stop,
                    // then give up on things going idle and start clearing
                    // them out. Or if r is the last of activity of the last task the stack
                    // will be empty and must be cleared immediately.
                    mStackSupervisor.scheduleIdleLocked();
                    //开始scheduleIdleLocked，但是它会有一个同步锁，所以它会很晚调用，见下面 scheduleIdleLocked  
                }
        }
        if (DEBUG_STATES) Slog.v(TAG_STATES,
                "Moving to STOPPING: "+ r + " (finish requested)");
        r.state = ActivityState.STOPPING;  //设置ActivityRecord的状态为 STOPPING
        return r;
    }
}
```

会立即调用 scheduleIdleLocked,但是activityIdleInternal 会等待全局锁，所以这里会比较滞后去调用.

```java
final void scheduleIdleLocked() {
    mHandler.sendEmptyMessage(IDLE_NOW_MSG);
}
```

接下来看下 resumeTopActivitiesLocked

```java
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
         Bundle targetOptions) {
     //targetStack 即是MainActivity所在的 stack. 是非home stack的那个stack
     if (targetStack == null) {
         targetStack = mFocusedStack;
     }
     // Do targetStack first.
     boolean result = false;
     if (isFrontStack(targetStack)) { // MainActivity所在的stack还在前端
         result = targetStack.resumeTopActivityLocked(target, targetOptions);
     }
 }

 final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
    if (mStackSupervisor.inResumeTopActivity) { //inResumeTopActivity 表示表示正在resume top activity
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;  //表明现在正在resuming top activity.
        result = resumeTopActivityInnerLocked(prev, options);
    } finally {
        mStackSupervisor.inResumeTopActivity = false;  // resume 成功了, 重置变量
    }
    return result;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
    final ActivityRecord next = topRunningActivityLocked(null);
    final TaskRecord prevTask = prev != null ? prev.task : null;
    // 当前MainActivity已经是finished掉，所以MainActivity所在的stack里没有下一个可以running的activity，这里的
    // next 为null
    if (next == null) {
        // There are no more activities!
        final String reason = "noMoreActivities";
        if (!mFullscreen) { //是否是全屏， 这里是全屏
            ...
        }
        // Let's just start up the Launcher...
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: No more activities go home");
        // Only resume home if on home display
        final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo(); //这里returnTaskType=HOME_ACTIVITY_TYPE
        // 当前的stack已经没有别的activity了，那么就直接 resume HomeStack里的task
        return isOnHomeDisplay() && mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, reason);
    }
}
```

接着来看下 resumeHomeStackTask

```java
boolean resumeHomeStackTask(int homeStackTaskType, ActivityRecord prev, String reason) {
    //这里是 HOME_ACTIVITY_TYPE，所以略过这个分支
    if (homeStackTaskType == RECENTS_ACTIVITY_TYPE) {
        mWindowManager.showRecentApps();
        return false;
    }

    if (prev != null) {
        prev.task.setTaskToReturnTo(APPLICATION_ACTIVITY_TYPE);
    }
    //直接在home stack里将 HOME_ACTIVITY_TYPE 的task移动到top, 这个task即是Launcher所在的task
    mHomeStack.moveHomeStackTaskToTop(homeStackTaskType);
    ActivityRecord r = getHomeActivity();  // Launcher activityrecord
    if (r != null) {
        mService.setFocusedActivityLocked(r, reason);  //设置为focused activity, 并且将HomeStack设置为focused stack
        return resumeTopActivitiesLocked(mHomeStack, prev, null); //再次进入 resumeTopActivitiesLocked, 直接用homestack去resume top activity
    }
}
```

resume Launcher3

``` java

void activityIdleInternal(ActivityRecord r) {
    synchronized (mService) {
        activityIdleInternalLocked(r != null ? r.appToken : null, true, null);
    }
}

final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
        Configuration config) {
    ArrayList<ActivityRecord> stops = null;
    ArrayList<ActivityRecord> finishes = null;
    ArrayList<UserState> startingUsers = null;
    int NS = 0;
    int NF = 0;
    boolean booting = false;
    boolean activityRemoved = false;

    ActivityRecord r = ActivityRecord.forTokenLocked(token); //这里并没有传token过来，为NULL

    if (allResumedActivitiesIdle()) { //现在 Launcher还没resume,这里为false.
    }

    // Atomically retrieve all of the other things to do.
    stops = processStoppingActivitiesLocked(true);  //获得 stopping activities
    NS = stops != null ? stops.size() : 0;

    // Stop any activities that are scheduled to do so but have been
    // waiting for the next one to start.
    for (int i = 0; i < NS; i++) {
        r = stops.get(i);
        final ActivityStack stack = r.task.stack;
        if (stack != null) {
            if (r.finishing) {
                stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false);
                //这里是在 finish, finishCurrentActivityLocked
            } else {
                //不是finish，就只 stop
                stack.stopActivityLocked(r);
            }
        }
    }

    // Finish any activities that are scheduled to do so but have been
    // waiting for the next one to start.
    for (int i = 0; i < NF; i++) {
        r = finishes.get(i);
        final ActivityStack stack = r.task.stack;
        if (stack != null) {
            activityRemoved |= stack.destroyActivityLocked(r, true, "finish-idle");
        }
    }

    if (activityRemoved) {
        resumeTopActivitiesLocked();
    }

    return r;
}
```

```java
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj) {
     // make sure the record is cleaned out of other places.
     // 清除掉 r
     mStackSupervisor.mStoppingActivities.remove(r);
     mStackSupervisor.mGoingToSleepActivities.remove(r);
     mStackSupervisor.mWaitingVisibleActivities.remove(r);
     if (mResumedActivity == r) {
         mResumedActivity = null;
     }
     final ActivityState prevState = r.state;
     if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to FINISHING: " + r);
     r.state = ActivityState.FINISHING;

     if (mode == FINISH_IMMEDIATELY
             || (mode == FINISH_AFTER_PAUSE && prevState == ActivityState.PAUSED)
             || prevState == ActivityState.STOPPED
             || prevState == ActivityState.INITIALIZING) {

         r.makeFinishingLocked();
         boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm");
         if (activityRemoved) {
             mStackSupervisor.resumeTopActivitiesLocked();
         }
         return activityRemoved ? null : r;
     }

     mStackSupervisor.mFinishingActivities.add(r);
     r.resumeKeyDispatchingLocked();
     mStackSupervisor.getFocusedStack().resumeTopActivityLocked(null);
     return r;
 }
```

```java
final boolean destroyActivityLocked(ActivityRecord r, boolean removeFromApp, String reason) {
    cleanUpActivityLocked(r, false, false);  // clean up activity

    final boolean hadApp = r.app != null;

    if (hadApp) {
        try {
            // 开始在HelloWorld进程里进入 destroy 生命周期
            r.app.thread.scheduleDestroyActivity(r.appToken, r.finishing,
                    r.configChangeFlags);
        } catch (Exception e) {
        }

        // If the activity is finishing, we need to wait on removing it
        // from the list to give it a chance to do its cleanup.  During
        // that time it may make calls back with its token so we need to
        // be able to find it on the list and so we don't want to remove
        // it from the list yet.  Otherwise, we can just immediately put
        // it in the destroyed state since we are not removing it from the
        // list.
        if (r.finishing && !skipDestroy) {
            //这里会等待 destroy 完成.
            r.state = ActivityState.DESTROYING;
            Message msg = mHandler.obtainMessage(DESTROY_TIMEOUT_MSG, r);
            mHandler.sendMessageDelayed(msg, DESTROY_TIMEOUT);
        }
    }

    return removedFromHistory;
}
```
