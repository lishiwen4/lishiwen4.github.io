---
layout: post
title: "多用户与 userid uid appid"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. 单用户时代的 app 与 linux uid/gid

android 从诞生之初， 即将 linux uid/gid 体系用于 android package 的管理， 即为不同的 package 分配不同的 uid/gid (当然， 多个package也可以共享gid/uid, 另外gid通常 与 uid相同)

android.os.Process 中定义了 uid 的分配

    public class Process {
        ......
        public static final int SYSTEM_UID = 1000;
        public static final int PHONE_UID = 1001;
        ......
    }
    
对于系统中的package， 可以使用如下的方法来查看其uid

+ 运行该package对应的 app， 使用`adb shell ps | grep <package name>` 查看输出结果
+ 查看 "/data/system/packages.xml" 文件， 找到该 package 信息对应的 section， 其中， "userId=xxx" 即为该package的 uid
+ 执行 `adb shell pm dump <package name>` 命令, 其中 "userId=xxx" 即为该package的 uid

###2. android 多用户支持

Android 4.2开始支持多用户， UserManager 的 getMaxSupportedUsers()方法可以获取当前设备支持的最大用户数目的上限

    public static int getMaxSupportedUsers() {
        ......
        // 小内存设备， 只支持 1个用户
        if (ActivityManager.isLowRamDeviceStatic()) return 1; 

        int multiuserMaximumUsers = 1;
        try {
            switch(ramSizeLevel()) {
                case 1:
                    multiuserMaximumUsers = Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers_1G);
                    break;
                case 2:
                    multiuserMaximumUsers = Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers);
                    break;
                case 3:
                    multiuserMaximumUsers = Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers_3G);
                    break;
                case 4:
                    multiuserMaximumUsers = Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers_4G);
                    break;
                default:
                    multiuserMaximumUsers = Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers);
                    break;
            }
        } catch (Resources.NotFoundException rnfe) {
            multiuserMaximumUsers = 1;
        }

        return SystemProperties.getInt("fw.max_users", multiuserMaximumUsers);
    }

可以按照设备的内存大小， 修改对应的config值， 限定用户数目

###3. 多用户与 userid

每一个android user都有一个 userid 来表示， android.os.UserHandle 中定义了一些特殊的 userid

    public final class UserHandle implements Parcelable {

        //代表所有的用户            
        public static final int USER_ALL = -1;
        
        //代表当前的用户
        public static final int USER_CURRENT = -2;
        
        //代表当前用户或者调用者用户
        public static final int USER_CURRENT_OR_SELF = -3;
        
        //代表用户不存在
        public static final int USER_NULL = -10000;
        
        //代表主用户
        public static final int USER_OWNER = 0;

        ......
    }
    
###4. 多用户与 uid

为了支持多用户， 将 linux 的 uid 划分为不同的区间， 分配给不同的 android user， android.os.UserHandle 中有如下的定义

    public final class UserHandle implements Parcelable {
        ......
        public static final int PER_USER_RANGE = 100000;
        ......
    }

例如 user 0 的 uid 区间为 [0, 99999), user 1 的 uid 区间为 [100000, 199999)

###5. 多用户与 appid

在单用户时代， 每个package单独分配一个uid/gid来进行管理(共享uid/gid的除外)， 而在支持多用户之后， 因为 uid 按区间分配给不同的 android user, 同一个 app， 在不同的 android user 下运行时， 其 uid 是不同的， 因此， 无法使用唯一的 uid 来标识一个 android  package， 因此， 使用 appid 来标识一个 android package， appid 在安装时确定， 与运行状态无关， 因此， 在不同的 android user 下， 同一个 package 的 appid 是不变的

在支持多用户之后， 如下的值保存的都是 package 的 appid

+ android.content.pm.ApplicationInfo.uid
+ /data/system/packages.xml 中对应的pcakage的信息中的 "userId=xxx" 中的值
+ 执行 `adb shell pm dump <package name>` 命令, 其中 "userId=xxx" 中的值

例如， 在一台有id分别为 0 和 10 的 2个 user 的 android 设备上， 

在一个有两个用户（用户id分别为0和10）的安卓设备上，在这2个用户下， 安装同一个app

+ 在用户 0 下:
   + 查看packages.xml，看到uid没有变化10078
   + Process.myUid()得到uid为”10078”
   + Process.myUserHandle()得到”userHandle{0}” 
   + 运行该 app， 使用 `ps` 命令查看 uid 为 "u0_a78" 
+ 在用户 10 下:
   + 从packages.xml查看此应用的uid：userId=”10078”
   + Process.myUid()得到uid为”1010078”
   + Process.myUserHandle()得到”userHandle{10}”
   + 运行该 app， 使用 `ps` 命令查看 uid 为 "u10_a78"
    
###6. userid uid 与 appid 的关系

在 android.os.UserHandle 中， 提供了转换这3者的方法

    public static final int getUid(int userId, int appId) {
        if (MU_ENABLED) {
            return userId * PER_USER_RANGE + (appId % PER_USER_RANGE);
        } else {
            return appId;
        }
    }
    
    public static final int getAppId(int uid) {
        return uid % PER_USER_RANGE;
    }
    
    public static final int getUserId(int uid) {
        if (MU_ENABLED) {
            return uid / PER_USER_RANGE;
        } else {
            return 0;
        }
    }

例如， 对于一台 android 设备上 user id 为 10 的一个 app

+ 从packages.xml查看此应用的uid：userId=”10078”
+ Process.myUid()得到uid为”1010078”
+ Process.myUserHandle()得到”userHandle{10}”

其中， appid 为 10078， uid 为 1010078， userid 为10， 按照它们的转换关系

    uid = userid * PER_USER_RANGE + appid
    1010078 = 10 * 100000 + 10078;
    
###7. uid 的格式化输出

很多时候， uid 并不是直接使用数字来显示， 比如， 对于 uid 1010078 ， 以类似于 "u0a78" 这样的形式来显示， 那么它们之间是怎样的转换关系呢

首先， 在  android.os.Process 中， 有如下的定义

    public class Process {    
        ......
        // 分配给 app 的 uid 的范围， 在支持多用户时， 这一值并不是绝对的 uid 值
        // 而是在 对应的 android user 的 uid 范围内的相对值
        public static final int FIRST_APPLICATION_UID = 10000;
        public static final int LAST_APPLICATION_UID = 19999;
        
        // 分配给被隔离的沙箱进程的 uid 的范围, 在支持多用户时， 这一值并不是绝对的 uid 值
        // 而是在 对应的 android user 的 uid 范围内的相对值
        public static final int FIRST_ISOLATED_UID = 99000;
        public static final int LAST_ISOLATED_UID = 99999;
        ......
    }

以 android.os.UserHandle 的 formatUid 的实现为例:

    public static void formatUid(StringBuilder sb, int uid) {
        if (uid < Process.FIRST_APPLICATION_UID) {
            sb.append(uid);
        } else {
            sb.append('u');
            sb.append(getUserId(uid));
            final int appId = getAppId(uid);
            if (appId >= Process.FIRST_ISOLATED_UID && appId <= Process.LAST_ISOLATED_UID) {
                sb.append('i');
                sb.append(appId - Process.FIRST_ISOLATED_UID);
            } else if (appId >= Process.FIRST_APPLICATION_UID) {
                sb.append('a');
                sb.append(appId - Process.FIRST_APPLICATION_UID);
            } else {
                sb.append('s');
                sb.append(appId);
            }
        }
    }

因此, 以 u10a78 和 uid 1010078 为例

    u*axxx = u{uid / PER_USER_RANGE}a{uid % PER_USER_RANGE - FIRST_ISOLATED_UID}
    u10a78 = u{1010078 / 100000}a{1010078 % 100000 - 10000}         