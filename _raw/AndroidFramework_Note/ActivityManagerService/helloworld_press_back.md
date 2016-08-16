这篇文章讲述 finish activity人的整个过程. 通篇文章基于如下场景
1. Launcher打开HelloWorld app, 这时MainActivity获得焦点并显示.
2. 按 back 键回到Launcher界面.

需要注意的是MainActivity里并没有override onBackPressed或onKeyDown, 这样back键就不会被拦截。

### back key event 的传递
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


### 整个流程图如下所示

![]( finish activity 流程图)


> ActivityStack/AMS/ASS 属于 system 进程  
> MainActivity/ActivityThread 属于HelloWorld进程 <br>
> Launcher/ActivityThread 属于 Launcher 进程

因此在此例 finish activity中涉及到三个进程间的通信.
另外HelloWorld是由AMS里的非 HomeStack 的那个栈管理的， 这里命令为 Non-HomeStack, 而Launcher是Home应用，它运行在HomeStack中。

### PAUSING
AMS通知HelloWorld所在的stack开始pausing MainActivity, 调用 startPausingLocked, 然后 System 就退出了，等着MainActivity pause完成通知再继续完成下一步
当HelloWorld完成Pause后，通知 AMS  activityPaused, 让AMS 继续进行接下来的工作。

### PAUSED/STOPPING
当AMS 收到 activityPaused通知后，表明MainActivity已经Paused了， 这时就需要将MainActivity在AMS 里的状态置为PAUSED

### Resumed

### FINISHING/DESTROYING

### DESTROYED
