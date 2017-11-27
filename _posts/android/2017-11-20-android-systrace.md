---
layout: post
title: "android systrace"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. systrace

systrace是一个分析android性能问题的基础工具, 在系统的一些关键链路（比如System Service，虚拟机，Binder驱动）插入一些追踪信息，通过追踪信息的开始和结束来确定某个过程的执行时间，然后把这些追踪信息收集起来得到系统关键路径的运行时间信息，进而得到整个系统的运行性能信息

systrace 实质上是一系列工具的封装：

1. ftrace ： Systrace利用了Linux Kernel中的ftrace功能。所以，如果要使用Systrace的话，必须开启kernel中和ftrace相关的模块
2. Trace 类 ： Android定义了一个Trace类。应用程序可利用该类把统计信息输出给ftrace
3. aTrace ： 从ftrace中读取统计信息然后交给数据分析工具来处理
4. systrace.py ： 通过 atrace 配置数据采集的方式（如采集数据的标签、输出文件名等）和收集ftrace统计数据并生成一个网页文件供用户查看

###2. 应用中使用systrace

应用中可以使用 android 提供的 java 和 native 接口来使用 systrace:

1. java 接口 (frameworks/base/core/java/android/os/Trace.java):
   + `void beginSection(String sectionName)`
   + `void endSection()`
2. native 接口 （frameworks/native/include/android/trace.h）:
   + `void ATrace_beginSection(const char* sectionName)`
   + `void ATrace_endSection()`
   
例如 : 

    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
            Bundle savedInstanceState) {
        Trace.beginSection("Fragement_onCreateView");
        // .. 其他代码
        // ...
        // .. 结束处
        Trace.endSection();
    }

###3. 在非 debug 版 app 上开启 systrace

待trace的App必须是debuggable的，但是debuggable的App与非debuggable的性能有较大差别；因为系统为了支持debug开启了一些列功能并且关闭掉了某些重要的优化

如果我们想要待分析的App尽可能接近真实情况，那么必须要在非Debug的App中能启用自定义 systrace section；因为相同情况下Debug的App性能比非Debuggable的差，你无法确保在debuggable版本上分析出来的结论能准确推广到非debuggable的版本上

我们可以在非debug的app里面手动开启自定义 systrace section，方法也很简单，调用一个函数即可；但是这个函数是SDK @hide的，我们需要反射调用

    Class<?> trace = Class.forName("android.os.Trace");
    Method setAppTracingAllowed = trace.getDeclaredMethod("setAppTracingAllowed", boolean.class);
    setAppTracingAllowed.invoke(null, true);
    
把这段代码放在Application的`attachBaseContext` 中，这样就可以手动开启自定义的systrace section了

###4. 使用命令启动 systrace

进入systrace的目录，也即(假设$ANDROID_HOME是你Android SDK的根目录）：

    cd $ANDROID_HOME/platform-tools/systrace
    
目录中可以找 ‘systrace.py’ 工具, 它的一般用法是

    systrace.py [options] [category1 [category2 ...]]
    
1. options : 命令参数
   + -a <package_name>：这个选项可以开启指定包名App中自定义 systrace section， 默认情况下， 自定义section是不生效的， 这一参数不能少
   + -t N：用来指定Trace运行的时间，取决于你需要分析过程的时间；还是那句话，在需要的时候尽可能缩小时间；当然，绝对不要把时间设的太短导致你操作没完Trace就跑完了，这样会出现`Did not finish` 的标签，分析数据就基本无效了
   + -l：这个用来列出你分析的那个手机系统支持的Trace模块；也就是上面命令中 `[category...]`能使用的部分
   + -o FILE：指定trace数据文件的输出路径，如果不指定就是当前目录的`trace.html`
2. category : 等是你感兴趣的系统模块，常用的模块有：
   + `sched`: CPU调度的信息，非常重要；你能看到CPU在每个时间段在运行什么线程；线程调度情况，比如锁信息。
   + `gfx`：Graphic系统的相关信息，包括SerfaceFlinger，VSYNC消息，Texture，RenderThread等；分析卡顿非常依赖这个。
   + `view`: View绘制系统的相关信息，比如onMeasure，onLayout等；对分析卡顿比较有帮助。
   + `am`：ActivityManager调用的相关信息；用来分析Activity的启动过程比较有效。
   + `dalvik`: 虚拟机相关信息，比如GC停顿等。
   + `binder_driver`: Binder驱动的相关信息，如果你怀疑是Binder IPC的问题，不妨打开这个。
   + `core_services`: SystemServer中系统核心Service的相关信息，分析特定问题用。

am代表ActivityManager（包含Activity创建过程等）；分析不同问题的时候，可以选择不同你感兴趣的模块。需要重复的是，尽可能缩小需要Trace的模块，其一是数据量小易与分析；其二，虽然systrace本身开销很小，但是缩小需要Trace的模块也能减少运行时开销。比如你分析卡顿的时候，`power`, `webview` 就几乎是无用的

###5. 图形界面启动 systrace

android studio 的 Device Monitor 界面上， 打开 DDMS 窗口， 点击 systrace 图标， 即可启动 systrace

![启动systrace](/images/android/systrace-01.png)

在弹出的对话框上选择抓取的package， 持续时间和关注的模块以及生成的trace文件的路径

![选择systrace参数](/images/android/systrace-02.png)

点击ok后， 在android设备上进行要trace的操作， 持续时间结束后， 会生成对应的trace文件

###6. 查看 trace文件

将生成的 trace 文件拖动到 chrome 中打开

![帮助对话框](/images/android/systrace-03.png)

左侧红框是数遍模式切换面板， 右边红框是帮助对话框， 点击后弹出

![帮助对话框](/images/android/systrace-04.png)

通常， 使用 'a' 's' 'w' 'd' 键 和鼠标滚轮即可完成大部分操作 

###7. 定位时间段

将鼠标模式切换到 timimg 模式， 可以拖动选取一段时间， 突出显示

![帮助对话框](/images/android/systrace-05.png)

选中某一段， 按 'm' 键 突出显示这一段

![帮助对话框](/images/android/systrace-06.png)

选中某一个 frame， 按 'm' 键 突出显示这一段

![帮助对话框](/images/android/systrace-07.png)

###8. systrace性能调优

我们主要查看界面上的Alert、Frames，其中可以用绿色、黄色、红色来区分当前界面的性能等级分别是优、良、差， 例如， 点击某一个标红色的 frame

![帮助对话框](/images/android/systrace-08.png)

底部会相对应的优化提示以及可能会出现优化的视频教程链接， 比如， 上面提到， measure/layout 和 draw 所花的时间都太长了

当然Systrace无法帮你定位到代码里面的具体到某一行代码，但是我们可以通过Alerts和Frames来能基本上优化了不足的地方，然后我们可以根据TraceView来分析具体函数花了多长时间来进一步优化代码提高性能