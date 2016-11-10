> 这篇 blog 主要是记录 自己对于 Android Stack 的理解


### 环境

- Env: Android 7.0

- File:

涉及到的几个非常重要的类

- ActivityManagerService (AMS)
这个Android 核心类，它运行在 SystemServer 中，管理 Android 的所有组件

- ActivityStarter
    Activity的启动控制器，它控制 Activity 的启动逻辑。 可能会拦截不让其启动

- ActivityStackSupervisor (ASS)
    固名思义，它是 stack 的管理者

- ActivityStack
    表示一个具体的 stack

- TaskRecord
    表示一个任务

### 创建首个 stack

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

 现在来看下  mHomeStack, mFocusedStack, mLastFocusedStack的定义

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

mHomeStack 是包含有 launcher app的栈，它的stack id 是 HOME_STACK_ID 为 0

为什么需要在这里初始化 home stack 呢？
因为马上要启动 home 应用了。


接下来看下 getStack

```java
ActivityStack getStack(int stackId, boolean createStaticStackIfNeeded, boolean createOnTop) {

    if (!createStaticStackIfNeeded || !StackId.isStaticStack(stackId)) {
        return null;
    }
    return createStackOnDisplay(stackId, Display.DEFAULT_DISPLAY, createOnTop);
}
```

在 Android 7.0 以上， 静态的stack 只有5个， (在Android 7.0 以下，静态栈只有一个 home stack)

- HOME_STACK_ID 0

    Launcher所在的 Stack

- FULLSCREEN_WORKSPACE_STACK_ID 1

    全屏的 activities 所在的 stack

- FREEFORM_WORKSPACE_STACK_ID 2

    freeform/resized 的 activities 所在的 stack

- DOCKED_STACK_ID 3

    占据着固定区域的 activities 所在的stack

- PINNED_STACK_ID 4

    PIP stack

### HOME Stack 中启动 Launcher

- 将 HomeStack
``` java
ActivityManagerService.systemReady()
  startHomeActivityLocked()   //ActivityManagerService
    startHomeActivityLocked() //ActivityStarter
      moveHomeStackTaskToTop() //ActivityStackSupervisor
        mHomeStack.moveHomeStackTaskToTop()   //指定Home stack 将 home task 移动到顶端
      startActivityLocked() //ActivityStarter 开始启动Home Activity
```
