---
layout: post
title: "android studio 调试应用程序"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. java 调试原理

java 平台的调试规范 JPDA（Java Platform Debugger Architecture） 由 3 个部分组成

1. Java 虚拟机工具接口（JVMTI） : 获取及控制当前虚拟机状态
2. Java 调试线协议（JDWP） : 定义 JVMTI 和 JDI 交互的数据格式
3. Java 调试接口（JDI） : 提供 Java API 来远程控制被调试虚拟机

开发人员只需使用 JDWP 和 JVMTI 即可支持跨平台的远程调试，但是直接编写 JDWP 程序费时费力，而且效率不高, JDI 不仅能帮助开发人员格式化 JDWP 数据，而且还能为 JDWP 数据传输提供队列、缓存等优化服务, 因此基于 Java 的 JDI 层的引入，简化了操作，提高了开发人员开发调试程序的效率

![android app调试原理](/images/android/as-debugger-00.png)

Android虚拟机虽然和普通java虚拟机存在不少差异，但是它的调试接口同样是基于jdwpP协议的, android 的 dalvik / art 虚拟机内部有一个专门的jdwp线程，Android系统的adbd进程通过socket与各个虚拟机的jdwp线程进行通信，外部调试器通过adb工具与adbd通信进而完成与jdwp的通信(androidstudio、eclipse 等IDE的调试功能和DDMS的监控功能也都是基于JDWP协议实现的), 我们通常所说的 "attach debugger" 指的就是这个意思——连接到指定的需要调试的进程的jdwp线程

###2. Android Studio 开启 debug

Android Studio目前已经成为开发Android的主要工具, 其中集成了 debug 工具用于 android app 的调试

debug 一个 app 有2种方式:

![android app调试原理](/images/android/as-debugger-01.png)

1. 以 debug 模式启动一个 app : 可以先下好断点，然后用debug模式编译安装这个app
2. attach 到已经启动的 app 上进行调试 ： 可以在启动apk之后，直接下断点，然后attach process到指定进程，条件触发之后就可以直接进入调试模式

要退出debug， 点击 ”attach“ 按钮右边的 stop 按钮即可

无论是那种方式， 请保持app的源码和调试的app一致， 这会使调试过程简单很多， 否则， 会出现行号无法对应。

当行号不能对应时，下断点的时候都有可能出现问题； 比如你在TestClass的第100行下了一个断点，但是由于行号不对应，有可能真正执行的代码第100行是没有意义的空行或者是在下一个函数里面，这样断点就没有起到应有的作用了。

若不能确保行号对应，必须使用方法断点 ： 直接在某个函数的入口设置断点，这样即使行号对不上，也能在正确的入口出断下来，这一点非常重要

###3. Android debug面板

进入调试模式后， android studio 左下方会出现debug 面板

![android app调试原理](/images/android/as-debugger-02.png)

debug 可以分为个功能区

1. 单步调试区 
2. 断点管理区
3. 表达式求值区
4. 线程帧栈区
5. 对象变量区
6. 变量观察区
7. 线程状态区

###4. 基本的单步调试

以 systemui 为例， quicksettings 的 “更多开关” 的点击事件由 com.android.systemui.qs.tiles.MoreFunctionTile#handleClick() 处理, 接下来就以这一过程为例， 演示简单的单步调试

####4.1 设置断点

attach debugger 到 systemui 上后， 在 com.android.systemui.qs.tiles.MoreFunctionTile#handleClick() 中设置断点 (最简单的方式是在源码的行号右侧单击左键即可添加断点)

![android app调试原理](/images/android/as-debugger-03.png)

点击 quicksetting 上的 “更多开关”, 触发断点， 程序停止执行， 等待调试

####4.2 断点处观察上下文信息

![android app调试原理](/images/android/as-debugger-04.png)

当前停在第一个断点处, debug 面板上显示了如下的信息

1. 断点所在的线程， 示例中的 handleClick() 函数在 QSTileHost 线程中运行
2. 调用栈， 示例中的 handleClick() 由 QSTile 类的 handleMessage() 调用
3. 调用栈切换按钮， 通过上下键可以在调用栈中上下移动， 选择不同的栈帧， 相应的也会打开的对应的源码文件， 点击漏斗按钮可以选择是否隐藏libra中的栈帧（即是否只显示源码中的栈帧）
4. 断点所处的对象的成员变量的值 (在源码编辑框中， 也会显示局部变量的值)

####4.3 单步执行程序

如下的7个按钮可以控制单步调试

![android app调试原理](/images/android/as-debugger-05.png)

1. step over : 单步执行， 不会进入到调用的方法里面
2. step into : 单步执行， 如果下一步是有方法调用并且是自定义方法， 则会进入该方法， 如果是官方类库， 则不会进入，
3. force step into ： 单步执行， 如果下一步是有方法调用则进入， 无论是自定义方法还是是官方类库中的方法
4. step out ： 如果调试时， step into 或者 force step into 了一个方法， 并且觉得该方法没有问题，你就可以使用step out跳出该方法，返回到该方法被调用处的下一步前并停止
5. drop frame ： 放弃当前的堆栈， 返回到当前方法的调用处，并且所有上下文变量的值也回到那个时候（只要调用链中还有上级方法）， 可以重新执行某个方法
6. run to cursor ： 继续执行，到光标所在处才停止 (途中遇到断点也不会停止)
7. resume program ： 继续执行， 直到遇到下一个断点， 或者程序执行完成

通过使用单步调试和变量观察区， 可以执行到程序中的不同位置， 查看相应的状态的变化

在调试过程中可禁用单个或者所有的 断点， 避免重复的删除和添加

####5. 进阶技巧

####5.1 观察变量值

debug 面板上的 对象变量区 可以查看当前对象的成员变量， 如果某个类或方法中变量太多，在Variables面板里观察的话会很费劲，这时就会需要用到 变量观察区 了， 通过如下的任一方式， 可以把变量添加到变量观察区， 方便观察

1. 在 对象变量区 选中某个变量， 单击邮件， 选择 "Add To Waches"
2. 在 变量观察区 点击 "New Watches" 按钮(带加号的那个图标)， 输入变量名称

![android app调试原理](/images/android/as-debugger-06.png)

####5.2 修改变量值

调试的过程中， 可以方便的修改某个变量的值， 在 对象变量区， 选中某个变量， 单击右键， 选择 "Set Value", 即可修改变量的值

####5.3 表达式求值

这是一个强大的功能， 在断点处停止时， 可以嵌入一个交互式解释器，在该解释器中，你可以执行任何你想要执行的表达式进行求值操作

![android app调试原理](/images/android/as-debugger-07.png)

####5.4 查看线程状态

点击线程 Dump 按钮, 获取所有线程的状态

![android app调试原理](/images/android/as-debugger-08.png)

线程状态值如下 :

![android app调试原理](/images/android/as-debugger-09.png)

1. Thread is suspended
2. Thread is waiting on a monitor lock
3. Thread is running
4. Thread is executing network operation, and is waiting for data to be passed
5. Thread is idle
6. Event Dispatch Thread that is busy
7. Thread is executing disk operation

####5.4 对象标记

在调试的过程中，这个方式允许你给class的某个特定的object打标签，以便后面的断点里面可以进行识别这个变量

在对象变量区选中所需的对象， 右键选择 "Mark Object" 后输入 label， 即可标记对象

###6. 高级断点技巧

很多地方将断点分类为 ： ”条件断点“ ”日志断点“ ”异常断点“ ”方法断点“ ”变量断点“， 这种分类方式其实并不明确， 例如， 一个断点既可以同时是 ”条件断点“ 和 ”日志断点“

先来看一看 android studio 中的 断点查看 页面， 左侧按断点的作用对象来分类， 右侧是断点的配置选项, **这些配置并不是孤立的， 可以结合使用以满足不同的需求**

![android app调试原理](/images/android/as-debugger-10.png)

可以从如下的 3 个维度来区分断点:

1. 作用的对象
2. 触发的条件
3. 触发后的动作

####6.1 断点作用的对象

####6.1.1 行号断点

最基础的断点， 在方法体内的任一行上设置的断点即为行号断点， 在 android studio 的源码窗口行号右侧单击左键即可设置行号断点

![android app调试原理](/images/android/as-debugger-11.png)

使用行号断点时， 源码文件一定要和带调试的文件一致， 否则会出现行号不对应， 调试时实际停下来的位置并不是在源码窗口上指定的位置

#####6.1.2 方法断点

方法断点设置在方法名上， 而不是方法体内， 在 android studio 的源码窗口里方法名所在行的行号右侧单击左键即可设置方法断点

使用方法断点可以观察方法的参数和返回值

![android app调试原理](/images/android/as-debugger-12.png)

另外， 当无法保证源码文件和待调试的 app 一致时， 可以使用方法断点， 即使行号对不上， 也可以在方法的入口处停下来

方法变量可以指定在何时触发(可多选)

![android app调试原理](/images/android/as-debugger-13.png)

1. Method entry ： 进入方法触发断点
2. Method exit ： 退出方法时触发断点

#####6.1.3 变量断点

在 class 的某个成员变量上设置的断点称为 变量断点

![android app调试原理](/images/android/as-debugger-14.png)

变量断点可以选择在变量被访问或者被修改时触发， 停在被访问或者被修改的位置

![android app调试原理](/images/android/as-debugger-15.png)

####6.1.4 异常断点

异常断点是设置在某个异常上的断点， 和上述3种断点不同， 异常断点需要在 断点查看 页面进行设置

异常断点可以选择在异常被caught时进行通知还是未被caught时通知

![android app调试原理](/images/android/as-debugger-16.png)

####6.2 断点的触发条件

断点可以设置不同的触发条件， 以减少不必要的触发

#####6.2.1 条件触发

断点可以设置一个条件，当给定的条件满足时， 才会触发， 否则会略过  

![android app调试原理](/images/android/as-debugger-17.png)

#####6.2.2 断点触发的断点

断点触发的断点， 默认是 disable 的状态， 可以指定另一个断点， 只有指定的断点被出触发后， 这一断点才会被 enable， 

![android app调试原理](/images/android/as-debugger-18.png)

并且还可以选择被enable的时效 :

1. disable agin : 变为 enable 后， 被触发一次后又继续变为 disable
2. leave enbaled ： 变为 enbale 后， 就一直保持 enbale 状态

#####6.2.3 一次性断点

可以指定断点在被触发一次后就自动移除

![android app调试原理](/images/android/as-debugger-19.png)

#####6.2.4 对象实例过滤

在某个class 中设置的断点， 可以指定 在该 class 的某个 特定对象上 触发 或者 不触发

![android app调试原理](/images/android/as-debugger-20.png)

#####6.2.5 class 过滤

在某个class 中设置的断点， 可以指定 在该 class 的某个 特定子类上 触发 或者 不触发

![android app调试原理](/images/android/as-debugger-21.png)

#####6.2.6 指定次数过滤

可以指定某个断定满足触发条件时， 先略过指定次数， 不触发

![android app调试原理](/images/android/as-debugger-22.png)

####6.3 断点触发后的动作

可以定义断点触发和执行不同的动作

#####6.3.1 触发后暂停程序运行

可以选择在断点被触发时， 是否暂停程序的运行

![android app调试原理](/images/android/as-debugger-23.png)

勾选后， 有2个互斥选项供选择暂停的范围 ： 

1. All : 暂停所有的线程
2. Thread ： 只暂停断点所在的线程

**注意， 当选择all时， 断点触发后， UI线程也会被阻塞， 此时不要去操作ui， 否则会触发ANR**

#####6.3.2 触发后记录log到console

当断点被触发时， 记录 log 信息 到 debug 面板的 console 页面中， **注意不是logcat console**

![android app调试原理](/images/android/as-debugger-24.png)

#####6.3.3 触发后计算表达式的值 （只支持行号断点）

当断点被触发时， 计算指定的表达式， 并记录 log 信息 到 debug 面板的 console 页面中

当然也可以在 表达式中使用 Log.x 输出 log 到 logcat

![android app调试原理](/images/android/as-debugger-25.png)


