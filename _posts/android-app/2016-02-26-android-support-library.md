---
layout: post
title: "android support library"
description:
category: android-app
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. android.support

google提供了Android Support Library package 系列的包来保证来高版本sdk开发的向下兼容性，即我们用4.x开发时，在1.6等版本上，可以使用高版本的有些特性，如fragement,ViewPager等, 它本质上是一个 java library


###2. android.support.v4

最早（2011年4月份）实现的库。用在Android1.6 (API lever 4)或者更高版本之上。它包含了相对V4, V13大的多的功能。（例如：Fragment，NotificationCompat，LoadBroadcastManager，ViewPager，PageTabAtrip，Loader，FileProvider 等， 这个包是使用最广泛的，eclipse新建工程时，都默认带有了


###3. android.support.v7

这个包是为了考虑Android2.1(API level 7) 及以上版本而设计的，但不包含更低，故如果不考虑1.6,我们可以采用再加上这个包， 但是v7是要依赖v4这个包的，也就是如果要使用，两个包得同时 被引用。（v7支持了Action Bar。）

###4. android.support.v13

这个包的设计是为了android 3.2及更高版本的，一般我们都不常用，平板开发中能用到

###5. android.support.design

这一lib包含了8个新的material design组件， 最低支持Android 2.1， 这个库和github上的很多开源项目是有很大关系的，material design的很多效果，同一种效果在github上有太多的实现，现在官方把部分效果标准化了

Design Support Library包含8个控件，具体如下：

+ android.support.design.widget.TextInputLayout             强大带提示的MD风格的EditText
+ android.support.design.widget.FloatingActionButton        MD风格的圆形按钮，来自于ImageView
+ android.support.design.widget.Snackbar                    类似Toast，添加了简单的单个Action
+ android.support.design.widget.TabLayout                   选项卡方式来切换view
+ android.support.design.widget.NavigationView              抽屉式导航
+ android.support.design.widget.CoordinatorLayout           超级FrameLayout
+ android.support.design.widget.AppBarLayout                MD风格的滑动Layout
+ android.support.design.widget.CollapsingToolbarLayout     可折叠MD风格ToolbarLayout