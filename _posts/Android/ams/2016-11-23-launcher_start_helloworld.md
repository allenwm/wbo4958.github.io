---
layout: post
category: Android
title: Launcher启动HelloWorld
tagline:
tags:  [startActivity]
---
{% include JB/setup %}

> 继上一篇 [Home应用的启动](https://wbo4958.github.io/2016-11-18-android_home_starting) 后，
大概了解了下 Home 应用的启动过程.<br>
而这篇笔记主要是记录由Launcher启动一个应用程序的过程。其中会
涉及到栈的查找，新栈的建立，task的建立等等。

### 1. 环境

Android: 7.0.0

HelloWorld apk: 仅一个MainActivity

<div align="center">
    <img src="/assets/images/android/AMS/launcher_start_helloworld.png"
    alt="launcher start helloworld 图"/>
</div>

在Launcher打开HelloWorld前，注意以下几个前置条件：
- 当前只有一个 Home stack.
- Home stack 当前resumed activity 是 Launcher

### 2. Launcher通知 AMS 启动 MainActivity

图中的1-6过程是Launcher直接调用startActivity启动HelloWorld的MainActivity(至于Launcher是怎么调到AMS的startActivity
这部分逻辑，这里并不做解释，可以阅读Lancher代码);

其中 intent 为

```
act=android.intent.action.MAIN
cat=android.intent.category.LAUNCHER
cmp=com.example.helloworld/.MainActivity
```
通过这两项就能找到要启动的是 helloworld apk里的 MainActivity

其中步骤 5 是生成MainActivity对应的AMS里的ActivityRecord

### 3. 准备好栈与task

图中 7-12 是创建 stack / task 并建立对应联系的过程

- 通过 computeStackFocus 计算出需要哪个栈

    这里找到的是非HomeStack的栈，即建立一个新栈stack id为**FULLSCREEN_WORKSPACE_STACK_ID**

- 然后在找到的栈中通过createTaskRecord创建task
- 调用 startActivityLocked 建立起 ActivityRecord与 task之间的关系

图中 13-15 将新的栈移动到前端，
- 设置 MainActivity 为 Focused Activity
- 将 新栈移动到 Front, 并设置为 Focused Stack, 这时 focused 栈就改变了

### 4. 既然 stack/task 已经准备好了，就可以 resume MainActivity了

**注意** 当前resumed activity还是HomeStack里的Launcher, 既然要resume HelloWorld的
MainActivity, 这时就应该先stop Launcher了

图中19-24 是stop Launcher 的过程

``` java
boolean pauseBackStacks(boolean userLeaving, boolean resuming, boolean dontWait) {
    boolean someActivityPaused = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFocusedStack(stack) && stack.mResumedActivity != null) {
                someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming,
                        dontWait);
            }
        }
    }
    return someActivityPaused;
}
```

由上可知要pause stack里的activity的条件是
- stack 不是focused的  (这里是指 HomeStack)
- stack 中的 mResumedActivity 不为空 (HomeStack里当前mResumedActivity还是指向Launcher)

那么这里就是要pausing Launcher了

Stack里面的状态值
- 4.1 PAUSING

    20-22 进入Launcher的onPause(), 并且设置Launcher的状为PAUSING

- 4.2 PAUSED

    23-24 设置Launcher的状态为PAUSED

### 5. 开始resume top activity了

由于第4节已经将所有的非focused stack暂停了, 那么就可以resume focused stack中的HelloWorld的MainActivity了，

- 29-30 创建HelloWorld进程 (HelloWorld还没有启动过，需要新建立一个进程)
- 31-32 HelloWorld里的ActivityThread与AMS进行 attach/bind
- 33-40 生成MainActivity实例，并进入对应的生命周期 onCreate/onResume

### 6. 停止Launcher Activity

当HelloWorld已经完成生命周期onResume后, ActivityThread 里的Handler会调用Idler()，
即已经没有任何Message调度了，就调用这个Idler(). (**注意** 这个Idler()是在 **handleResumeActivity** 加入的，并不是说Handler一空闲就调用Idler())

Idler()通知AMS  activity 空闲了

一个Activity(MainActivity)的resume是先将之前的activity(Launcher) pausing了的，既然
MainActivity已经resume了，这时就得stop掉Launcher了

- 41-46 stoping Launcher activity
- 47-49 通知AMS Launcher已经stop了
