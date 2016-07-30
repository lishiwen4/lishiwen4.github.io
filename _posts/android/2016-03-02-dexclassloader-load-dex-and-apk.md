---
layout: post
title: "android DexClassLoader"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. ClassLoader

“类装载器”（ClassLoader），顾名思义，就是用来动态装载class文件的。标准的Java SDK中有个ClassLoader类，借助此类可以装载需要的class文件，前提是ClassLoader类初始化必须指定class文件的路径。

每一个ClassLoader必须有一个父ClassLoader，在装载Class文件时，子ClassLoader会先请求其父ClassLoader加载该文件，只有当其父ClassLoader找不到该文件时，子ClassLoader才会继承装载该类。这是一种安全机制。对于Android而言，最终的apk文件包含的是dex类型的文件，dex文件是将class文件重新打包，打包的规则又不是简单地压缩，而是完全对class文件内部的各种函数表，变量表进行优化，产生一个新的文件，即dex文件。因此加载这种特殊的Class文件就需要特殊的类加载器。 android中提供了如下的2个类加载器：

+ DexClassLoader ：可以加载文件系统上的jar、dex、apk
+ PathClassLoader ：可以加载/data/app目录下的apk，这也意味着，它只能加载已经安装的apk
+ URLClassLoader ：可以加载java中的jar，但是由于dalvik不能直接识别jar，所以此方法在android中无法使用，尽管还有这个类

jar必须转换成dalvik所能识别的字节码文件，转换工具可以使用android sdk中platform-tools目录下的dx， 转换命令示例:

    dx --dex --output=dest.jar src.jar
    
利用这种技术， 可以实现apk插件， 被加载的apk称之为插件，因为机制类似于生物学的"寄生"，加载了插件的应用也被称为宿主。

###2. DexClassLoader

DexClassLoader的原型如下：

    DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent)
  
形参的意义如下：
  
+ dexPath 	需要装载的APK或者Jar文件的路径。包含多个路径用File.pathSeparator间隔开,在Android上默认是 ":" 
+ optimizedDirectory 	优化后的dex文件存放目录，不能为null
+ libraryPath 	目标类中使用的C/C++库的列表,每个目录用File.pathSeparator间隔开; 可以为 null
+ parent 	该类装载器的父装载器，一般用当前执行类的装载器， 即 `this.getClass().getClassLoader()`

DexClassLoader的基本使用流程如下：

+ 通过PacageMangager获得指定的apk的安装的目录，dex的解压缩目录，c/c++库的目录
+ 创建一个 DexClassLoader实例
+ 加载指定的类返回一个Class
+ 然后使用反射来调用这个Class

**需要注意的是optimizedDirectory参数， 不要把优化优化后的classes文件存放到外部存储设备上，防代码注入攻击**

###3. DexClassLoader使用示例

####3.1 插件apk

使用Android Studio建立一个app， 使用“Add No Activity”模板(当然也可以使用带Activity的模板), 然后添加一个plug类

    package com.example.sven.plug1;

    /**
    * Created by sven on 16-3-2.
    */
    public class plug {

        public String whoAmI()
        {
            return getClass().getName();
        }

        public int add(int a, int b)
        {
            return a + b;
        }
    }
    
编译后， 将apk安装到android设备上， 由于使用的是“Add No Activity”模板， 因此launcher并不显示图标

####3.2 宿主apk

使用Android Studio建立一个app， 使用“Blank Activity”模板(当然也可以使用带Activity的模板), 然后在floatActionBar的点击事件中动态加2载插件apk

    public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {

                String result = "null";

                ApplicationInfo appInfo = null;
                String packageName = "com.example.sven.plug1";

                try {
                    appInfo = getPackageManager().getApplicationInfo(packageName, 0);

                    String sourceDir = appInfo.sourceDir;
                    String outDir = getApplicationInfo().dataDir;
                    String libraryDir = appInfo.nativeLibraryDir;

                    DexClassLoader dexcl = new DexClassLoader(sourceDir, outDir,
                            libraryDir, this.getClass().getClassLoader());

                    Class<?> loadClass = dexcl.loadClass(packageName + ".plug");
                    Object instance  = loadClass.newInstance();

                    Method methodHello = loadClass.getMethod("whoAmI", new Class[0]);
                    Method methodAdd = loadClass.getMethod("add", new Class[]{Integer.TYPE, Integer.TYPE});

                    result = (String)methodHello.invoke(instance) + " : " + ((int)methodAdd.invoke(instance, 3, 5));

                } catch (PackageManager.NameNotFoundException e) {
                    e.printStackTrace();
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } catch (InstantiationException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }

                Snackbar.make(view, result, Snackbar.LENGTH_LONG)
                        .setAction("Action", null).show();
            }
        });
    }
    
    .....
    }
    
###4. 动态加载apk的缺陷

上述只是一个简单的动态加载apk示例， 事实上， 动态加载apk还涉及到几个难点：

+ 资源的访问 ：因为将apk加载到宿主程序中去执行，就无法通过宿主程序的Context去取到apk中的资源,比如图片，文本， layout等(因此， 插件apk中若涉及到ui， 需要在java code中绘制)，这是很好理解的，因为apk已经不存在上下文了，它执行时所采用的上下文是宿主程序的上下文，用别人的Context是无法得到自己的资源的，这个问题貌似可以这么解决：将apk中的资源解压到某个目录，然后通过文件去操作资源，这只是理论上可行，实际上还是会有很多的难点的。
+ activity的生命周期，因为apk被宿主程序加载执行后，它的activity其实就是一个普通的类，正常情况下，activity的生命周期是由系统来管理的，现在被宿主程序接管了以后，如何替代系统对apk中的activity的生命周期进行管理是有难度的，这个问题比资源的访问好解决一些，比如我们可以在宿主程序中模拟activity的生命周期并合适地调用apk中activity的生命周期方法
