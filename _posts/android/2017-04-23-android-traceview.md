---
layout: post
title: "android traceview"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. TraceView

Traceview是Android平台特有的数据采集和分析工具，它主要用于分析Android中应用程序的hotspot

Traceview本身只是一个数据分析工具，而数据的采集则需要使用Android Framework中的android.os.Debug类, 或者利用 android sdk 中的 DDMS 工具， 其各自的优缺点如下:

1. 使用 Debug 类 :
   + 优点是测试的代码范围精确
   + 缺点是需要修改代码并且自己导出文件
2. 使用 DDMS 工具 :
   + 优点是 操作简单， 不需要修改代码, 可以在任意时间 开启/关闭 trace
   + 缺点是测试的代码范围很宽泛， 不够精确

###2. 使用 android.os.Debug 采集trace数据

在需要profile的代码片段的首尾, 添加 Debug 类的如下方法来 开始/结束统计

    public static void startMethodTracing(String tracePath);
    public static void stopMethodTracing();

这两个函数运行过程中将采集运行时间内该应用所有线程（注意，只能是Java线程）的函数执行情况

开启profile时，解释的代码运行速度会较慢，并不代表实际执行速度, 在Android的4.4和更高版本，取样会减少性能影响。使用startMethodTracingSampling()即可，停止依旧使用stopMethodTracing()

注意, 生成的 trace 文件位于 “/sdcard/” 目录下， 若调用 startMethodTracing() 时不指定 tracePath 参数， 则默认生成名为 'dmtrace.trace' 的文件， 否则， 生成以 tracePath 参数为文件名， '.trace' 为后缀的 trace 文件， 若文件已经存在， 则其中的内容会被截取

例如， 在 Activity 的 ‘onCreate()’ 和 ‘onStart()’ 之间追踪： 

    
    public class MainActivity extends Activity {  
  
        @Override  
        protected void onCreate(Bundle savedInstanceState) {  
            super.onCreate(savedInstanceState);  
            setContentView(R.layout.activity_main);  
          
            Debug.startMethodTracing("test_01");  
        }  
  
        @Override  
        protected void onStart() {  
            super.onStart();  
          
            Debug.stopMethodTracing();  
        }  
      
    }
    
则执行后， 生成 "/sdcard/test_01.trace" 文件

**注意， 生成 trace 文件需要写 sd 卡， 所以要为app 声明 permission**

###3. 使用 am 命令来采集 trace 数据

adb shell 中 启动 / 关闭 trace 的命令如下

    am profile <PROCESS> start <FILE>
    am profile <PROCESS> stop
    
其中 "&lt;PROCESS&gt;" 填写进程名,android java app， 进程名通常是 package name，  "&lt;FILE&gt;" 填写 trace file 文件名

例如:

    am profile com.app.example start /sdcard/test.trace
    am profile com.app.example stop


另外， 还可以在 adb shell 中启动一个 Activity 的时候， 进行追踪

    am start -n <Package Name>/<Package Name>.<Activity Name> --start-profiler <FILE>
    
例如:

    am start -n com.example/com.app.example.MainActivity --start-profiler /sdcard/test.trace    

###4. 使用 DDMS 工具来采集 trace 数据

在 android studio 中点击图标来启动 Device Monitor 或者直接执行  Sdk/tools/monitor 来启动 Device Monitor

首先， 在左侧 device 管理界面， 选中需要追踪的 app， 然后点击 "start method profiling"， 当需要结束追踪时， 再次点击该界面即可

![点击 start method profiling](/images/android/traceview-01.png)


###5. TraceView 查看 trace 文件

Device Monitor 中自带了 TraceView 以查看 生成的 trace 数据 (使用 DDMS 工具采集的数据会自动在其中打开， 而使用 Debug 或者 am 命令生成的 trace 文件， 则需要自己拖动到其中打开)

![点击 start method profiling](/images/android/traceview-02.png)

traceview 面板可以分为2部分，其中:

1. 上面是时间轴面板 (Timeline Panel)：
   + 纵轴显示的是线程信息
   + 横轴显示的是时间信息
   + 线程执行时， 显示的是黑色背景， 线程暂停时， 显示的是白色背景
   + 线程在执行不同的方法时， 会被标记为不同的颜色， 当鼠标放置在其上时， 在顶部会显示当前所执行的方法的具体的信息
2. 下面是分析面板(Profile Panel) : Profile Panel 是 TraceView 的核心界面，其内涵非常丰富。它主要展示了某个线程（先在 Timeline Panel 中选择线程）中各个函数调用的情况，包括 CPU 使用时间、调用次数等信息。而这些信息正是查找 hotspot 的关键依据

####5.1 Profile Panel 中各统计列的意义

+ name : 函数的名称， 这一列通常可以展开如下的2项
   + parent : 追踪过程中， 所有直接调用了该函数的函数
   + children : 追踪过程中， 所有该函数直接调用的函数
+ Incl Cpu Time : 某函数占用的CPU时间，包含内部调用其它函数的CPU时间
+ Excl Cpu Time : 某函数占用的CPU时间，但不含内部调用其它函数所占用的CPU时间
+ Incl Real Time : 某函数运行的真实时间（以毫秒为单位），内含调用其它函数所占用的真实时间
+ Excl Real Time : 某函数运行的真实时间（以毫秒为单位），不含调用其它函数所占用的真实时间
+ Call+Recur Calls/Total : 某函数被调用次数 和 递归调用次数占总调用次数的百分比 (例如 8230+0 说明被调用8230次， 没有递归调用)
+ Cpu Time/Call : 某函数调用CPU时间与调用次数的比。相当于该函数平均执行时间
+ Real Time/Call : 同CPU Time/Call类似，只不过统计单位换成了真实时间

注意 cpu time 和  real time 的差别:

+ cpu time : 某一函数在执行过程中， 其所在的线程 真实占用cpu的时间
+ real time :  某一函数从开始执行到执行完成， 所耗费的真实时间， 在这段时间内， 因为 暂停，上下文切换，任务调度， GC 等原因， cpu time 通常小于 real time

**另外，每一个Time列还对应有一个用时间百分比来统计的列: ** 比如Incl Cpu Time列对应还有一个列名为Incl Cpu Time %的列，表示某一个函数的 Incl Cpu Time 占总的 Incl Cpu Time 的百分比

###6. 使用 android studio 查看 trace 文件

android studio 支持查看 traceview 生成的 trace 文件，在 android studio 中直接打开即可

android studio 中会以如下的图形的方式显示函数的调用关系， 可以结合 ddms 上的 traceview 面板使用

![点击 start method profiling](/images/android/traceview-03.png)

###7. 使用 dmtracedump 查看 trace 文件

dmtracedump 是 android sdk 中附带的另一个根据 trace 文件生成图形化调用堆栈的工具（platform-tools/dmtracedump）， 它支持生成 html 和 png 图片格式的输出

输出 html :

    dmtracedump -h ddmsxxx.trace > a.html

![点击 start method profiling](/images/android/traceview-04.png)

输出 png : 

    dmstracedump -g a.png ddmsxxx.trace 

生成的 png 中每个节点中依次为：

    [inclusive_time_rank]function_name(total_inclusive_time, total_exclusive_time, execute_times)

![点击 start method profiling](/images/android/traceview-05.png)

生成 png图片需要依赖 dot， 可使用如下的命令安装:

    sudo apt-get install graphviz

###8. traceview 查找 hotspot

程序中的 hotspot 可以分为2类 :

1. 被调用次数较少， 但是每次被调用耗时较长的的函数
   + 针对这一类 hotspot， 将 "Cpu Time / call" 这一列降序排列， 关注点放在那些单次执行时间较长的函数上
2. 每次被调用耗时较短， 但是调用次数较多的函数
   + 针对这一类 hotspot，点击 Call/Recur Calls/Total 列箭头，使之按降序排列, 关注点放在那些调用频繁并且占用资源较多的函数上