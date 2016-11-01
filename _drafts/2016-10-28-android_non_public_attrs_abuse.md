---                                                                                                                             
layout: post
category: Android
title: 记一个CTS case 对android non-public属性乱用的问题
tagline:
tags:  [non-public attrs, cts]
---
{% include JB/setup %}

> 最近解了一个Android CTS的问题，最后发现是Google CTS package里的一个case 乱用了一个
Android Non-public attrs而引起的。用这篇文章记录一下。

### 一、环境

OS: Android 7.0

CTS version: android-cts-7.0_r1

Repro Step:

```
run cts -m CtsServicesHostTestCases -t android.server.cts.ActivityManagerPinnedStackTests#testAlwaysFocusablePipActivity

--logcat-on-failure
```

Observation:

``` console
Test failing with error 'junit.framework.AssertionFailedError: Pinned stack must be the focused
 stack. expected:<4> but was:<0>'
```

### 二、 testAlwaysFocusablePipActivity

在Cts code里找到 ActivityManagerPinnedStackTests.java 里的testAlwaysFocusablePipActivity
方法， 这个Case主要是执行 adb shell 命令，然后dumpsys 一些信息来检查是否满足预期。整理如下:

``` console
adb shell am start -n android.server.app/.AlwaysFocusablePipActivity                                                                                                 
adb shell am stack move-top-activity-to-pinned-stack 1 0 0 500 500

adb shell dumpsys activity activities
```

主要分为三步：
第一, 启动 AlwaysFocusablePipActivity
第二, 将 AlwaysFocusablePipActivity 从stack 1 移动到 PIP stack (stack id 4).
第三, dump activity的activities信息， 检查  mFocusedStack 是否 stack id为4.

即最后的预期 stack id as follow

``` console
mFocusedStack=ActivityStack{7d928bf stackId=4, 2 tasks}
```

关键是 adb shell am stack move-top-activity-to-pinned-stack 1 0 0 500 500 这句命令，背后的目的就是将
AlwaysFocusablePipActivity 从stack 1 移动到 PINNED stack (stack id 4), 然后将 PINNED Stack设置为 Focused
的stack.

Tracking

``` java
moveTopActivityToPinnedStack
    moveTopStackActivityToPinnedStackLocked
        moveActivityToPinnedStackLocked
            moveTaskToStackLocked
                moveTaskToStackUncheckedLocked
                    moveToFrontAndResumeStateIfNeeded
                        moveToFront
                            setFocusStackUnchecked
```

```java
void setFocusStackUnchecked(String reason, ActivityStack focusCandidate) {
    if (!focusCandidate.isFocusable()) {
        // The focus candidate isn't focusable. Move focus to the top stack that is focusable.
        focusCandidate = focusCandidate.getNextFocusableStackLocked();
    }
}

boolean isFocusable() {
    if (StackId.canReceiveKeys(mStackId)) {
        return true;
    }
    // The stack isn't focusable. See if its top activity is focusable to force focus on the
    // stack.
    final ActivityRecord r = topRunningActivityLocked();
    return r != null && r.isFocusable();
}

public static boolean canReceiveKeys(int stackId) {
    return stackId != PINNED_STACK_ID;
}

boolean isFocusable() {
    return StackId.canReceiveKeys(task.stack.mStackId) || isAlwaysFocusable();
}

boolean isAlwaysFocusable() {
    return (info.flags & FLAG_ALWAYS_FOCUSABLE) != 0;
}
```

当要设置当前PINNED stack为Focused stack的时候会去检查 这个stack是否是 可 focusable 的！
其中 PINNED_STACK 是不能接收Key的，所以只能当topRunningActivity  可 focusable，那么 PINNED stack
才能Focusable.

从 isAlwaysFocusable 可以看出来，只有当PINNED STACK里的 Activity的 flag 设置了FLAG_ALWAYS_FOCUSABLE
才能focuse.

FLAG_ALWAYS_FOCUSABLE 是个什么东西？

### 三、FLAG_ALWAYS_FOCUSABLE标志

FLAG_ALWAYS_FOCUSABLE 这个标志是在 PKMS 去解析AndroidManifest.xml里设置的。

```java
if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysFocusable, false)) {
    a.info.flags |= FLAG_ALWAYS_FOCUSABLE;
}
```

那么问题就转换为要为AlwaysFocusablePipActivity设置 alwaysFocusable 属性了

找到AlwaysFocusablePipActivity所在的AndroidManifest.xml里的描述，

```xml
<activity android:name=".AlwaysFocusablePipActivity"
          android:theme="@style/Theme.Transparent"
          android:resizeableActivity="true"
          android:supportsPictureInPicture="true"
          androidprv:alwaysFocusable="true"                                                                                                                  
          android:exported="true"
          android:taskAffinity="nobody.but.AlwaysFocusablePipActivity"/>
```

很明显  alwaysFocusable 已经被设置为 true (这里的 androidprv只是一个命名空间，无实际用处), 但是
PKMS 居然没有把它给解析出来。 那这肯定就是 PKMS 的问题了？ 不然，请继续查看

### 四、 PKMS解析 alwaysFocusable 属性

明明已经设置了 alwaysFocusable 属性了，为什么 PKMS 解析不出来呢？？？？ 这么奇怪的问题。 从 三 中
PKMS 获得该属性的方法得知，无非就是从  TypedArray 里去获得么，这还有错，莫非是AssetManager没有把该值
解析出来，赶紧 debug AssetManager.

```java
TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestActivity);
```

R.styleable.AndroidManifestActivity 定义在 framework-res里的 attrs_manifest.xml中

```xml
<declare-styleable name="AndroidManifestActivity" parent="AndroidManifestApplication">
    <!-- Required name of the class implementing the activity, deriving from
        {@link android.app.Activity}.  This is a fully
        qualified class name (for example, com.mycompany.myapp.MyActivity); as a
        short-hand if the first character of the class
        is a period then it is appended to your package name. -->
    <attr name="name" />
    <attr name="theme" />
    <attr name="label" />
    <attr name="description" />
    <attr name="icon" />
    <attr name="banner" />
    ...
    <attr name="alwaysFocusable" format="boolean" />
    <attr name="enableVrMode" />
</declare-styleable>
```

R.styleable.AndroidManifestActivity 就一组int型数组， 数组元素是一个一个属性通过 aapt编译出来的ID,
具体可以从

```
out/debug/target/common/obj/APPS/framework-res_intermediates/public_resources.xml
```

去获得对应的 id 值, xml文件里面包含了所有的整个系统的resource id

本地编译的属性值

```xml
<public type="attr" name="theme" id="0x01010000" />
<public type="attr" name="label" id="0x01010001" />
<public type="attr" name="icon" id="0x01010002" />                                                                                                                 
<public type="attr" name="name" id="0x01010003" />

<!-- Declared at frameworks/base/core/res/res/values/attrs_manifest.xml:1882 -->
<public type="^attr-private" name="systemUserOnly" id="0x011600c4" />
<!-- Declared at frameworks/base/core/res/res/values/attrs_manifest.xml:1899 -->
<public type="^attr-private" name="alwaysFocusable" id="0x011600c5" />
```


obtainAttributes 最终会调用 frameworks/base/core/jni/android_util_AssetManager.cpp
里的 android_content_AssetManager_retrieveAttributes 函数， 而该函数获得对应属性值就是
通过 该属性的 id 值。

而且很奇怪的一点，用本地编出来的CTS package去测试，结果就PASS了，而AOSP的CTS package 确实
失败的。 这时通过加log, 发现 AOSP 编出来的CTS 里的 alwaysFocusable 的属性值与 本地编出来
的属性是不一样的.

AOSP 属性值

```xml
<!-- Declared at frameworks/base/core/res/res/values/attrs_manifest.xml:1899 -->
<public type="^attr-private" name="alwaysFocusable" id="0x011600c3" />
```

从这里基本就可以知道为什么 alwaysFocusable 不能被解析出来了。

### 五、 结论

为什么本地与AOSP的 alwaysFocusable 的 ID 属性值是不一样的呢？ 从 type="^attr-private
与 type="attr" 可以猜出来 该 alwaysFocusable 应该是 私有的 属性，非 public的属性，

定义成 public 的属性的ID 值 不管是在什么平台下编出来，应该是一样的。而非public 的属性就
不一定了，因为vendor可能自己会定义一些属性，这样  aapt 在解析这些属性时赋值时 可能就不
一样了。 而public的属性，作为AOSP 提供给 上层 app 开发者，它们的值必须一样，否则就会导致
app 在不同的厂商不兼容的情况。

public的属性值可以从 frameworks/base/core/res/res/values/public.xml 里查看

因此，从这里可以看出来 这个是 Google 的问题， 因为Google 用了非public 的属性值编译出来
的CTS package 来测试不同的 vendor, 这样是不恰当的。
