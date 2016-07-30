---
layout: post
title: "java 反射机制"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. java reflect

反射的概念是由Smith在1982年首次提出的，主要是指程序可以访问、检测和修改它本身状态或行为的一种能力，java反射机制是指java语言动态获取信息以及动态调用对象的方法的特性：

+ 在运行时判断任意一个对象所属的类
+ 在运行时构造任意一个类的对象
+ 在运行时判断任意一个类所具有的成员变量和方法
+ 在运行时调用任意一个对象的方法
+ 生成动态代理

java程序中一些类或方法中经常加上“@hide”注释标记。它的作用是使这个方法或类在生成SDK时不可见，因此由此注释的东西，你在编译期是不可见的，但是这样的类或者方法可以通过反射来访问

###2. Class相关基础

Java中有一个Class类。Class类本身表示Java对象的类型，Class类是整个Java反射机制的源头

获得Class对象的方法有许多，但是没有一种方法是通过Class的构造函数来生成Class对象的：

+ 类实例.class
   + 例如 `Test test； Class cl = test.class`
+ 类名.getClass()
   + 例如  `Class Test{};  Class cl = Test.getClass();`
+ Class.forName("类名")
   + 例如  `Class cl = Class.forName(xxx.xxx.xxx.Test);`
+ 类名.TYPE （只适用于boolean，byte，char，short，int，long，float，double，void这些基本数据类型）
   + 例如  `Class cl = Integer.TYPE;`

####2.1 Class的主要方法

+ public static Class<?> forName(String className)
   + 动态加载类
+ public T newInstance() 
   + 根据对象的class新建一个对象，用于反射
+ public ClassLoader getClassLoader()
   + 获得类的类加载器， 一般为system classloader， java中有如下3种类加载器
   + Bootstrap ClassLoader 此加载器采用c++编写，一般开发中很少见。
   + Extension ClassLoader 用来进行扩展类的加载，一般对应的是jre\lib\ext目录中的类
   + AppClassLoader 加载classpath指定的类，是最常用的加载器。同时也是java中默认的加载器
+ public String getName()
   + 获取类或接口的名字，记住enum为类，annotation为接口
+ public native Class getSuperclass()
   + 获取类的父类，继承了父类则返回父类，否则返回java.lang.Object

#### java.lang.reflect

如果要对一个类的信息重要性进行排名的话，那么这三个信息理应获得前三的名次。它们分别是：构造函数、成员函数、成员变量。

也许你不同意我的排名，没关系。对于Java反射来说，这三个信息与之前介绍的基本信息相比较而言，有着本质的区别。那就是，之前的信息仅仅是只读的，而这三个信息可以在运行时被调用（构造函数和成员函数）或者被修改（成员变量）。所以，我想无可否认，至少站在Java反射机制的立场来说，这三者是最重要的信息

Constructor 标识构造函数， 获取构造函数的方法有以下几个：

    Constructor getConstructor(Class[] params) 
    Constructor[] getConstructors()
    Constructor getDeclaredConstructor(Class[] params) 
    Constructor[] getDeclaredConstructors()

可以获取指定形参的构造函数， 也可以获取所有的构造函数

**`getConstructor(Class[] params)` 和`getConstructors()`仅仅可以获取到public的构造函数，而`getDeclaredConstructor(Class[] params) 和getDeclaredConstructors()`则能获取所有（包括public和非public）的构造函数”**


和获取构造函数的方法类似，获取成员函数的方法有以下一些：

    Method getMethod(String name, Class[] params)
    Method[] getMethods()
    Method getDeclaredMethod(String name, Class[] params) 
    Method[] getDeclaredMethods() 

**`getMethod(String name, Class[] params)` 和`getMethods()`仅仅可以获取到public的成员方法，而`getDeclaredMethod(String name, Class[] params) 和Method[] getDeclaredMethods()`则能获取所有（包括public和非public）的成员方法”**


获取成员变量的方法与上面两种方法类似，具体如下：

    Field getField(String name)
    Field[] getFields()
    Field getDeclaredField(String name)
    Field[] getDeclaredFields()
    
lass类支持反射的概念，Java附带的库java.lang.reflect包含了Field、Method以及Constructor类 (每个类都实现了Member接口)。这些类型的对象是由JVM在运行时创建的，用以表示未知类里对应的成员。这样可以使用Constructor创建新 的对象，用get()和set()方法读取和修改与Field对象关联的字段，用invoke()方法调用与Method对象关联的方法。另外，还可以调 用getFields()、getMethods、getConstrucotrs()方法，返回表示字段、方法以及构造器的对象的数组
    
###2. 反射机制的使用示例

假设有一个java类Sample， 需要在运行时， 使用反射来获取它的信息

    package com.example.sven.reflect;

    public class Sample {

        public String name;
        public static int num = 7;

        static{
            num = 3 + 5;
        }

        public Sample(String t)
        {
            name = t;
        }

        public String hello()
        {
            return "hello world";
        }

        public static int add(int a, int b)
        {
            return a + b;
        }
    }
    
####2.1 利用反射来访问成员变量name
                
    try {
        Class cl = Class.forName("com.example.sven.reflect.Sample");
        Constructor cons = cl.getConstructor(new Class[]{String.class});
        Object object = cons.newInstance("sample");

        // get 如果这个属性是非公有的，这里会报IllegalAccessException
        Field field = cl.getField("name");
        String ori = field.get(object).toString();

        // set
        field.set(object, "SAMPLE");
        String now = field.get(object).toString();
           
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    }
    catch (IllegalAccessException e) {
        e.printStackTrace();
    }catch (ClassNotFoundException e) {
        e.printStackTrace();
    }catch (NoSuchMethodException e) {
        e.printStackTrace();
    }catch (InvocationTargetException e) {
        e.printStackTrace();
    }catch (InstantiationException e) {
        e.printStackTrace();
    }
    
###2.2 利用反射来访问静态成员变量num

    try {
        Class cl = Class.forName("com.example.sven.reflect.Sample");
        
        //get
        Field field = cl.getField("num");
        int ori = (int)field.get(cl);

        //set
        field.set(cl, 615);
        int now = (int)field.get(cl);

    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }catch (ClassNotFoundException e) {
        e.printStackTrace();
    }

###2.3 利用反射来调用成员函数hello()

    try {
        Class cl = Class.forName("com.example.sven.reflect.Sample");

        Constructor cons = cl.getConstructor(new Class[]{String.class});
        Object object = cons.newInstance("sample");

        Method method = cl.getMethod("hello");
        String result = method.invoke(object).toString();
        
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }catch (ClassNotFoundException e) {
        e.printStackTrace();
    }catch (NoSuchMethodException e) {
        e.printStackTrace();
    }catch (InvocationTargetException e) {
        e.printStackTrace();
    }catch (InstantiationException e) {
        e.printStackTrace();
    }

###2.4 利用反射来调用静态成员函数add()

    try {
        Class cl = Class.forName("com.example.sven.reflect.Sample");

        Method method = cl.getMethod("add", new Class[]{Integer.TYPE, Integer.TYPE});
        int result = (int)method.invoke(null, 5, 6);
        
    }catch (IllegalAccessException e) {
        e.printStackTrace();
    }catch (ClassNotFoundException e) {
        e.printStackTrace();
    }catch (NoSuchMethodException e) {
        e.printStackTrace();
    }catch (InvocationTargetException e) {
        e.printStackTrace();
    }
    
###2.5 利用反射来查看未知的类

上述的4个示例是建立在清楚了解Sample类的基础上的， 对于一个不清楚内部结构的类， 还可以使用反射来查看其构造函数， 成员变量和成员方法

    cl = Class.forName("com.example.sven.reflect.Sample");
    Field[] field = cl.getDeclaredFields();
    //遍历所有的成员变量的名称
    for(int i = 0; i < field.length; i++ )
    {
        System.out.println(field[0].getName());
        ......
    }

    Class c = Class.forName("com.example.sven.reflect.Sample");
    Constructor[] cons = c.getDeclaredConstructors();
    
    // 遍历所有声明的构造方法
    for (int i = 0; i < cons.length; i++)
    {
        ......
    }

    //遍历第一个构造函数的参数的类型    
    if( cons.length > 0 )
    {
        for(int i =0; i < params.length; i++) {
            System.out.println(params[0].getName());
        }
    }
            
    Class c = Class.forName("com.example.sven.reflect.Sample");
    Method[] ms = c.getDeclaredMethods();
    //遍历所有声明的方法
    for (int i = 0; i < ms.length; i++)
    {
        ......
    }
    
### 关于反射的安全问题

Java的安全机制其实是比较复杂的，至少对于我来说是如此。作为Java的安全模型，它包括了：字节码验证器、类加载器、安全管理器、访问控制器等一系列的组件。之前文中提到过，我把Android安全权限划分为三个等级：第一级是针对编译期的“@hide”标记；第二级是针对访问权限的private等修饰；第三级则是以安全管理器为托管的Permission机制。

Java反射确实可以访问private的方法和属性，这是绕过第二级安全机制的方法（之一）。它其实是Java本身为了某种目的而留下的类似于“后门”的东西，或者说是为了方便调试？不管如何，它的原理其实是关闭访问安全检查。

如果你具有独立钻研的精神的话，你会发现之前我们提到的Field、Method和Constructor类，它们都有一个共同的父类AccessibleObject 。AccessibleObject 有一个公共方法：void setAccessible(boolean flag)。正是这个方法，让我们可以改变动态的打开或者关闭访问安全检查，从而访问到原本是private的方法或域。另外，访问安全检查是一件比较耗时的操作，关闭它反射的性能也会有较大提升