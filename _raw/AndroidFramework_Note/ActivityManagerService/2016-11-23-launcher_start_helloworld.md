> 继上一篇 [Home应用的启动](https://wbo4958.github.io/2016-11-18-android_home_starting) 后，
大概了解了下 Home 应用的启动过程，而这篇笔记主要是记录由 Launcher 启动一个应用程序的过程。其中会
涉及到栈的查找，新栈的建立，task的建立等等。

1. ### 环境

Android: 7.0.0

HelloWorld apk: 仅一个MainActivity
