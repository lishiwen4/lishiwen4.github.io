---
layout: post
title: "android permission"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. android permision 

android 采用了和Linux类型的权限隔离机制，每个应用使用独立的系统标识（uid 和）来运行，从而使得不同应用之间以及应用和系统之间隔离开来，　每一个Android 应用都是在一个进程沙箱中运行的，缺省情况下，应用无权执行可能对其它应用，操作系统或是用户造成影响的任何操作，这些操作可能包括读写用户个人数据（比如通讯录或是电子邮件），读写其它应用的文件，访问网络，拨打电话，发送短信, 录音, 拍照, 读取定位信息等

如有需要，应用必须明确指明需要和其它应用共享的资源或数据。这需要通过申明缺省“沙箱”未支持的“Permissions”来获得这些额外的功能, 应用在 AndroidManifest.xml 文件中静态声明所需permission，Android系统在安装应用时会询问用户是否赋予该应用所需权限。为避免用户决定安全问题的复杂性，Android系统不支持在运行时动态赋予应用所需permission(android M开始，　已经支持 runtime permission)

例如:

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.android.app.myapp" >
        <uses-permission android:name="android.permission.RECEIVE_SMS" />
        ...
    </manifest>

###2. PermissionInfo　／　PermissionGroupInfo

android framework 中使用　class PermissionInfo 来存储permission 相关的信息，　其中几个主要的成员如下:

+ public int protectionLevel;
   + 该permission的保护级别，　即获取该permission需要的条件
+ public String group;
   + 该permission所属的permission group
+ public int flags;
   + 该permission相关的flag, 共有如下几种
   + FLAG_COSTS_MONEY ：　该permission会涉及资费
   + FLAG_HIDDEN　: 该permission在dump时不显示给用户
   + FLAG_INSTALLED ：　该permission被安装到系统全局定义的权限中
+ public int descriptionRes;
   + 该permission相关的描述信息的字符串资源的id, 若没有则置为０

Android 6.0 支持多种permission保护级别，　在　permissionInfo　中有:

+ public static final int PROTECTION_NORMAL = 0;
   + AndroidManifest.xml 中使用　”dangerous“　来表示
   + 普通permission, 这些permission对于用户隐私和设备操作不会造成太多危险（比如开启手电筒的权限），apk在AndroidManifest中申明后， 系统会自动授予这些permission
+ public static final int PROTECTION_DANGEROUS = 1;
   + AndroidManifest.xml 中使用　”dangerous“　来表示
   + 即 Runtime permission，　这些permission可能会影响用户隐私和设备的普通操作（请求获取用户联系人信息的权限）， 系统会明确的让用户决定是否授予这些permission(Android 6.0以前， 在安装apk时授予这些权限， Android 6.0后， 可在安装后动态开关这些permission， 因此又称为运行时权限
+ public static final int PROTECTION_SIGNATURE = 2;
   + AndroidManifest.xml 中使用　”signature“　来表示
   +　签名级别的permission， 即申请该permission的apk必须和声明该permission的apk具有相同的签名才能申请的permission， 若签名匹配， 系统会自动授予这些permission
+ public static final int PROTECTION_SIGNATURE_OR_SYSTEM = 3;
   + AndroidManifest.xml 中使用　”signatureOrSystem“　来表示
   +　已废弃，　使用　`PROTECTION_SIGNATURE | PROTECTION_FLAG_PRIVILEGED`　来代替

以上的几种为基本的保护级别，　不能同时使用，　另外还有若干　extral flag, 需要和以上的几种 flag　配合使用才有意义 


1. public static final int PROTECTION_FLAG_PRIVILEGED = 0x10;
   + AndroidManifest.xml 中使用　`| privileged`　来表示
   + /system/priv-app　目录下的app　才能获取该permission
2. public static final int PROTECTION_FLAG_SYSTEM = 0x10;
   + AndroidManifest.xml 中使用　`| system`　来表示
   + PROTECTION_FLAG_PRIVILEGED 的旧名称
3. public static final int PROTECTION_FLAG_DEVELOPMENT = 0x20;
   + AndroidManifest.xml 中使用　`| development`　来表示
   + 具有该保护级别的permission，　只有之前被授权过，　才能被授权
4. public static final int PROTECTION_FLAG_APPOP = 0x40;
   + AndroidManifest.xml 中使用 `| appop` 来表示
   + 该属性和　AppOps相关
5. public static final int PROTECTION_FLAG_PRE23 = 0x80;
   + AndroidManifest.xml 中使用　`| pre23`　来表示
   + pre SDK 23
6. public static final int PROTECTION_FLAG_INSTALLER = 0x100;
   + AndroidManifest.xml 中使用　`| installer`　来表示
   + 该permission被授予给package installer 组件
7. public static final int PROTECTION_FLAG_VERIFIER = 0x200;
   + AndroidManifest.xml 中使用　`| verifier`　来表示
   + 该permission将被授予给　system verifier 组件
8. public static final int PROTECTION_FLAG_PREINSTALLED = 0x400;
   + AndroidManifest.xml 中使用　`| preinstalled`　来表示
   + 该permission将被授予给　预安装 的app

不同的保护级别可以使用　"位或"　操作来同时使用，　例如　`PROTECTION_SIGNATURE | PROTECTION_FLAG_PRIVILEGED` 表示要满足　PROTECTION_SIGNATURE　或者　PROTECTION_FLAG_PRIVILEGED　即可获取该permission

android frameworks 中使用　class PermissionInfo 来存储permission group相关的信息

拥有system级别permission的使用者可以access其他普通permission　和　signature　permission

**权限检查中有些特例， 可以参见第1６节**

###3. PackageParse.Permission / PackageParse.PermissionGroup

PackageManagerService　会分析所有的package, 收集系统中所有定义的permission

PackageParse.Permission / PackageParse.PermissionGroup　是对　PermissionInfo　／　PermissionGroupInfo　的包装

com.android.server.pm.PackageManagerService 中保存了系统中所有的　permission group 的信息

    // Mapping from permission names to info about them.
    final ArrayMap<String, PackageParser.PermissionGroup> mPermissionGroups =
            new ArrayMap<String, PackageParser.PermissionGroup>();
            
通过如下的　api　可以获取到　PermissionGroupInfo

    public PermissionGroupInfo getPermissionGroupInfo(String name, int flags)
    public List<PermissionGroupInfo> getAllPermissionGroups(int flags)
    
com.android.server.pm.Settings 中存储了所有　permission　的信息

    // Mapping from permission names to info about them.
    final ArrayMap<String, BasePermission> mPermissions =
            new ArrayMap<String, BasePermission>();

通过如下的　api 可以获取到　PermissionInfo

    public PermissionInfo getPermissionInfo(String name, int flags)

###4. android　framework定义的permission

Android 系统定义的permission位于源码的 “framework/base/core/res/AndroidManifest.xml”中

    <permission-group android:name="android.permission-group.CONTACTS"
        android:icon="@drawable/perm_group_contacts"
        android:label="@string/permgrouplab_contacts"
        android:description="@string/permgroupdesc_contacts"
        android:priority="100" />

    <permission android:name="android.permission.READ_CONTACTS"
        android:permissionGroup="android.permission-group.CONTACTS"
        android:label="@string/permlab_readContacts"
        android:description="@string/permdesc_readContacts"
        android:protectionLevel="dangerous" />

    ......
    
    <permission android:name="android.permission.USE_FINGERPRINT"
        android:permissionGroup="android.permission-group.SENSORS"
        android:label="@string/permlab_useFingerprint"
        android:description="@string/permdesc_useFingerprint"
        android:protectionLevel="normal" />

    ......
    
    <permission android:name="android.permission.BLUETOOTH_PRIVILEGED"
        android:protectionLevel="system|signature" />

    ......
    
    <permission android:name="android.permission.RECEIVE_BLUETOOTH_MAP"
        android:protectionLevel="signature|privileged" />

    ......
    
+ &lt;permission-group&gt;　定义了一个　permission group, 其中
   + icon : 该permission group对应的图标
   + label : 该permission group的标题
   + priority　: 该permission group的优先级，　即列出这些permission group时的先后顺序
   + description : 该permission group的详细描述
   
+ &lt;permission&gt; 定义了一个permission, 其中
   + permissionGroup ：　该permission所属的permission group
   + label : 该permission的标题
   + priority　: 该permission的优先级，　即列出这些permission group时的先后顺序
   + description : 该permission的详细描述, 规则是两句话，第一句描述，第二句警告用户如果授权会发生什么后果
   + protectionLevel : 该permission的保护级别
    
可以看到， 有很多permission是未分组的

在编译android 源码的过程中，　会根据　“framework/base/core/res/AndroidManifest.xml”　生成　“out/target/common/R/android/Manifest.java”文件，　其中有

    package android;

    public final class Manifest {
        public static final class permission {
            ......
            public static final String ACCESS_FINE_LOCATION="android.permission.ACCESS_FINE_LOCATION";
            ......
            public static final String BLUETOOTH_ADMIN="android.permission.BLUETOOTH_ADMIN";
            ......
            public static final String CALL_PHONE="android.permission.CALL_PHONE";
            ......
        }
        public static final class permission_group {
            ......
            public static final String CAMERA="android.permission-group.CAMERA";
            ......
            public static final String PHONE="android.permission-group.PHONE";
            ......
        }

因此，　在app开发中可以直接使用　android.Manifest.permission.CALL_PHONE　／　android.Manifest.permission_group.PHONE　这样的方式来引用　permission / permission group　的名称

###5. 自定义permission / permission-group

在package 的　AndroidManifest.xml　中：

+ 可使用　&lt;permission-group&gt;标签来自定义一个　permission group
   + android:name　属于必填项，　且必须至少包含２个"."
+ 可使用&lt;permission&gt;标签 来自定义一个　permission
   + android:name　属于必填项，　且必须至少包含２个"."
   + android:protectionLevel : 属于必填项，　且取值范围为"normal", "dangerous", "signature", "signatureOrSystem"

例如

    <permission-group
        android:name="com.whut3.sven.GROUP"
        android:icon="@mipmap/ic_perm_group"
        android:label="@string/perm_group_label"
        android:description="@string/perm_group_desc"
        android:priority="100"/>

    <permission
        android:permissionGroup="com.whut3.sven.GROUP"
        android:name="com.whut3.sven.test"
        android:icon="@mipmap/ic_perm_test"
        android:label="@string/perm_test_label"
        android:description="@string/perm_test_desc"
        android:protectionLevel="dangerous"/>

定义了一个　permission group “com.whut3.sven.GROUP”，　一个　permission　“com.whut3.sven.test”，　并且将该 permission 划分为归属　"permission group “com.whut3.sven.GROUP"　这一permission group

同样，　在编译 package　的过程中，　也会生成　Manifest.java，　例如　“out/target/common/obj/APPS/Bluetooth_intermediates／src/com/android/bluetooth/Manifest.java”

    package com.android.bluetooth;

    public final class Manifest {
        public static final class permission {
            /**  Allows access to the Bluetooth Share Manager 
             */
            public static final String ACCESS_BLUETOOTH_SHARE="android.permission.ACCESS_BLUETOOTH_SHARE";
            ......
        }
    }onSetUserDataEnabled

在相应的　package　内，可以使用　com.android.bluetooth.Manifest.permission.ACCESS_BLUETOOTH_SHARE 这样的方式直接引用permission的名称

还可以可以使用　&lt;permission-tree&gt;　标签配合　PackageManager.addPermission()　来动态添加 permission

    <permission-tree android:icon="drawable resource"
                     android:label="string resource"
                     android:name="string" />

+ name : 定义该命名空间内的 permission / permission group 的基础名称
+ icon : 代表该命名空间里面所有permission的图标, 当然，定义　permission / permission group　时还可以单独指定　icon
+ label ：　用户可见的权限名称, 当然，定义　permission / permission group　时还可以单独指定　label

&lt;permission-tree&gt;标签的作用类似于定义一个命名空间，　声明该命名空间中的　permission　都属于定义该　&lt;permission&gt;　的package的uid

例如, 某个 package　中定义了名为　"com.test.whut3" 的　permission-tree, 则　在package中可以　调用 PackageManager.addPermission() 添加以 "com.test.whut3"　为前缀名的　permission, 以这种方式添加的 permission 具有如下的特征

+ 类型为　TYPE_DYNAMIC
+ 没有　protect　level 和　所属的　group, 只保存　package name 和　permission name
+ 必须在任何其它　package 使用它之前被添加
+ 被添加后，　一直到设备重启，　或者调用　PackageManager.removePermission()　移除该　permission 之前都存在


###6. runtime permission

runtime permission 即 protect level 为　dangerous 的　permission 

在　android M 之前，　app在　AndroidManifest.xml 中声明所需的permission，　然后在app的安装过程中，　系统会提示用户所需的permission，　若用户同意安装，　则授予 app　所请求的所有permission，用户不能单独只对一部分 permission　进行授权并且安装完成后不能进行更改

在　android M 中，　引入了　runtime permission 的概念, 可在　app　运行阶段由　app　请求用户授权， 并且用户在授权后在可以在　“设置”　> “应用” > “权限”　中取消授权

runtime permission　及其所属的　permission group:

1. CALENDAR 	
   + READ_CALENDAR
   + WRITE_CALENDAR
   
2. CAMERA 	
   + CAMERA
   
3. CONTACTS 	
   + READ_CONTACTS
   + WRITE_CONTACTS
   + GET_ACCOUNTS

4. LOCATION 	
   + ACCESS_FINE_LOCATION
   + ACCESS_COARSE_LOCATION

5. MICROPHONE 	
   + RECORD_AUDIO

6. PHONE 	
   + READ_PHONE_STATE
   + CALL_PHONE
   + READ_CALL_LOG
   + WRITE_CALL_LOG
   + ADD_VOICEMAIL
   + USE_SIP
   + PROCESS_OUTGOING_CALLS

7. SENSORS 	
   + BODY_SENSORS

8. SMS 	
   + SEND_SMS
   + RECEIVE_SMS
   + READ_SMS
   + RECEIVE_WAP_PUSH
   + RECEIVE_MMS

9. STORAGE     	
   + READ_EXTERNAL_STORAGE
   + WRITE_EXTERNAL_STORAGE

“runtime permission”的控制粒度为组， 即只要获取到了某一 permission group 中的任意一个permission， 即系统会默认为已授予了该 permission group 内的所有permission， 后续申请该 permission group 内的其它 permission 时， 不再弹出提示框

有两个 permission 比较特殊，分别是SYSTEM_ALERT_WINDOW 和 WRITE_SETTINGS，　都属于比较敏感的permission，很多时候都用不到，如果需要这个 permission，应该首先声明，然后发送Intent去请求用户授权，然后系统会显示详细的管理界面

###７. 查看系统中的permission

使用　`adb shell pm list permission-groups` 命令可以查看系统中所有的permission　group，　例如

    $ pm list permission-groups                              
    permission group:android.permission-group.CONTACTS
    permission group:android.permission-group.OTHER
    permission group:android.permission-group.PHONE
    permission group:android.permission-group.CALENDAR
    permission group:android.permission-group.CAMERA
    permission group:android.permission-group.SENSORS
    ......
    
使用 `adb shell pm list permissions` 命令可以查看系统中所有声明的permission， 可以使用如下选项：

+ -g ： 按权限组显示，　若无　'-g'　选项则只列出未分组的　permission
+ -f ： 显示详细信息， 包括protect level， description， 以及定义该permission的package等信息
+ -d : 只列出 dangerous permission

例如：

    $ adb shell pm list permissions -g
    All Permissions:

    ......
    
    group:android.permission-group.CAMERA
        permission:android.permission.CAMERA
    ......
    
    group:android.permission-group.SENSORS
        permission:android.permission.BODY_SENSORS
        permission:android.permission.USE_FINGERPRINT

    group:android.permission-group.LOCATION
        permission:android.permission.ACCESS_FINE_LOCATION
        permission:android.permission.ACCESS_COARSE_LOCATION
    ......


    $ adb shell pm list permissions -f
    All Permissions:
    
    ......
    
    + permission:asus.permission.GET_LOCATION
      package:com.asus.taskwidget
      label:null
      description:null
      protectionLevel:signature
    + permission:android.permission.INSTALL_LOCATION_PROVIDER
      package:android
      label:null
      description:null
      protectionLevel:signature|privileged
    + permission:com.asus.gallery.filtershow.permission.WRITE
      package:com.asus.gallery
      label:null
      description:null
      protectionLevel:signature

    ......
    
'-g' 选项能够按group 来列出permission
'-f' 选项能够列出permission的　名称　／　定义该permissiond的package / 标签 / 描述信息　／　保护级别

###8. PermissionsState

android　系统中定义了众多的　permission, 每一个 app　可以申请不同的　permission，　一个 app　内部的　permission 的状态 使用　PermissionState　来表示

![PermissionState](/images/android/PermissionState.png)

+ PackageManagerService
   + mPackages　保存了　package name  到 PackageParser.Package　的映射
+ PackageParser.Package 
   + 保存了　PackageSetting　的实例
   + 如果该　package　有 shared　user id，　则　PackageSetting.sharedUser　将指向对应的　SharedUserSetting
+ SettingsBase.mPermissionState
   + 指向一个　PermissionState，　存储一个　package 或者　SharedUserId　的所有的　permission　的状态
+ PermissionState.mPermissions
   + 保存了pcakage内　permission name 到　PermissionState.PermisdsionData　的映射
+ PermissionState.PermissionData
   + mPerm保存了　一个permission 的　基本信息, 例如　name, type　等
   + mUserStates　保存了package内各个user的permission的状态信息
   + 提供了　isGranted(), grant(), revoke(), getFlags(), updateFlags(), getPermissionState() 等接口 来 查询/修改 permission　的状态
+ PermissionState.PermissionState
   + 最终存储了　packege　内一个permission对应某个user的配置状态
   
对于 BasePermission.type　可以分为３种情况:

1. TYPE_BUILTIN　：　"android"　package 中定义的　permission
2. TYPE_NORMAL : 在 AndroidManifest.xml　中静态定义的　permission
3. TYPE_DYNAMIC : 使用　PackageManagerService.addPermission() 接口添加的permission

对于某个　package　或者　Shared User Id, PackageManager　中某个　permission　的授权状态有２种表示

+ PackageManager.PERMISSION_GRANTED : 已获得该　permission　的授权
+ PackageManager.PERMISSION_DENIED　: 未获得该　permission　的授权

另外，　还有一组 flag　来标识　permission　的相关状态, 取值范围如下：

+ PackageManager.FLAG_PERMISSION_USER_SET
   + 表示 permission 当前的授权状态是用户设置的， app可以请求修改该 permission 的授权状态
+ PackageManager.FLAG_PERMISSION_USER_FIXED
   + 表示 permission 当前的授权状态是用户固定的， app可以不能再请求修改该 permission 的授权状态
   + android runtime permission　的权限请求弹框上的　"拒绝后不再询问"　框和该标记相关
+ PackageManager.FLAG_PERMISSION_POLICY_FIXED
   + 表示 permission　当前的授权状态是 device policy 固定的，　app 和　用户都不能改变该 permission 的授权状态
   + 具有该标记的　permission, 只有　system　用户才能修改它的授权状态和该　flag
+ PackageManager.FLAG_PERMISSION_REVOKE_ON_UPGRADE
   + 表示 app　是不支持 runtime permission　的，　在升级后开始支持 runtime　permission，　因此，　在升级过程中, 取消了该　permission　的 授权
+ PackageManager.FLAG_PERMISSION_SYSTEM_FIXED
   + 表示 permission　当前的授权状态是因为该 package　是系统组件
   + 具有该标记的　permission, 只有　system　用户才能修改它的授权状态和该　flag
+ PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT
   + 表示 permission　当前的授权状态是因为默认授权该　permission，　以便提供更好的 用户体验


###9. DefaultPermissionGrantPolicy 和　runtime permission

DefaultPermissionGrantPolicy 用于为系统组件或者　system handler　授予　runtime permission　授权
        
    PackageManagerService.SystemReady()
        DefaultPermissionPolicy.grantDefaultPermissions()
            grantPermissionsToSysComponentsAndPrivApps()
            grantDefaultSystemHandlerPermissions()
    
    PackageManagerService.newUserCreated()
        DefaultPermissionPolicy.grantDefaultPermissions()
            grantPermissionsToSysComponentsAndPrivApps()
            grantDefaultSystemHandlerPermissions()
            
DefaultPermissionPolicy.grantDefaultPermissions()　会为　需要预授权的package　的需要预授权的 permission　进行授权
            
+ DefaultPermissionGrantPolicy.grantPermissionsToSysComponentsAndPrivApps()　: 用于为系统组件或者使用　platform key sign且 persistent 的　priv　app　请求的所有的runtime　permission　授权
   + UID　小于　FIRST_APPLICATION_UID　的　package　会被视为　系统组件
   + 如果某个　permission　的 flag　包含　PackageManager.FLAG_PERMISSION_POLICY_FIXED　或者　PackageManager.FLAG_PERMISSION_SYSTEM_FIXED，　则不处理该　permission
   + 通过该接口设置权限状态成功的 permission　会被设置　PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT　和　PackageManager.FLAG_PERMISSION_SYSTEM_FIXED　flag
+ DefaultPermissionPolicy.grantDefaultSystemHandlerPermissions()　为需要预授权的system　handler　的runtime permission　进行授权
   + 通过该接口设置权限状态成功的 permission　会被设置　PackageManager.FLAG_PERMISSION_GRANTED_BY_DEFAULT flag，　但不会被设置　PackageManager.FLAG_PERMISSION_SYSTEM_FIXED　flag
   + 为 installer pacakge 授予　storage　group　的　runtime permission
   + 为 verifier package　授予　storage phone sms ３个group　的　runtime permission
   + 为 SetupWizard　(即　category 为　Intent.CATEGORY_SETUP_WIZARD　的所有　activity)　授予　phone　和　contacts　group　权限
   + 为 camera　package　授予　camera microphone storage　3 个 group 的 runtime permission
   + 为 Media provider 授予　storage １个　group 的 runtime permision
   + 为 Downloads provider　授予　storage １个　group 的 runtime permission
   + 为 Downloads UI 授予　storage １个　group 的 runtime permission
   + 为 Storage provider　授予　storage １个　group 的 runtime permission
   + 为 CertInstaller 授予　storage １个　group 的 runtime permission
   + 为 Dialer 授予 phone contacts sms microphone 4 个 group　的 runtime permission
   + 为 Sim call manager 授予　phone 和　microphone ２　个　group 的 runtime permission
   + 为 SMS 授予　phone contacts sms ３ 个 group　的 runtime permission
   + 为 Cell Broadcast Receiver　授予 sms 1个group　的　runtime permission
   + 为 Carrier Provisioning Service　授予　sms １个 group　的　runtime permission
   + 为 Calendar 授予　calendar 和　contacts　2个 group　的　runtime permission
   + 为 Calendar provider 授予 contacts calendar storage　３个 group　的　runtime permission
   + 为 Calendar provider sync adapters 授予　calendar　１个 group　的　runtime permission
   + 为 Contacts 授予　contacts phone　２个　group　的　runtime permission
   + 为 Contacts provider sync adapters　授予　contact 1个　group　的　runtime permission
   + 为 Contacts provider 授予　contacts phone storage 3个　group　的　runtime permission
   + 为 Device provisioning 授予　contact 1个　group　的　runtime permission
   + 为 Maps 授予　location　一个group　的　runtime permission
   + 为 Gallery　授予　storage　１个 group　的　runtime permission
   + 为 Email　授予 contacts　１个 group　的　runtime permission
   + 为 Browser 授予　location　１个 group　的　runtime permission
   + 为 IME　授予 contacts １个 group　的　runtime permission
   + 为 Voice interaction  授予　contacts calendar microphone phone sms location 6个 group　的　runtime permission
   + 为 PackageManagerService.PackageManagerInternalImpl.setLocationPackagesProvider()
   + 为 Voice recognition　授予　microphone 1个 group 的　runtime permission
   + 为 Location 授予　contacts calendar microphone phone sms location camera sensors storage 9 个　group　的　permission
   + 为 Music 授予　storage　１个 group 的　runtime permission
   
###10. DevicePolicyManagerService 和　runtime　permission

android DevicePolicyManagerService　中也可以　修改　android runtime permission　的状态

    DevicePolicyManagerService.setPermissionGrantState()
    
通过该接口将　runtime permission 设置为　grant　或者　deny 状态 时，　会设置　PackageManager.FLAG_PERMISSION_POLICY_FIXED　flag, 设置为 default　状态时，　会清除　DevicePolicyManager.PERMISSION_GRANT_STATE_DEFAULT flag 
   
###11. 查看apk的权限

目前已有多个工具可以检测Android app所请求的Permission，这类工具有：aapt、apktool、androguard等等， 以 aapt (android 源码可以build出该host tool)为例

    $ ./aapt dump permissions xxx.apk
    package: com.xxx.wallpaper
    uses-permission: name='android.permission.READ_PHONE_STATE'
    uses-permission: name='android.permission.INTERNET'
    uses-permission: name='android.permission.ACCESS_NETWORK_STATE'
    uses-permission: name='android.permission.WRITE_EXTERNAL_STORAGE'
    ......
    
pm 命令可以检查检系统中的package所请求的permission及授权状态

    $ pm dump com.android.phone
    ......
    requested permissions:
        android.permission.BROADCAST_STICKY
        android.permission.CALL_PHONE
        android.permission.CALL_PRIVILEGED
        android.permission.WRITE_SETTINGS
        android.permission.WRITE_SECURE_SETTINGS
        android.permission.READ_CONTACTS
        android.permission.READ_CALL_LOG
    ......
    install permissions:
        android.permission.SEND_RECEIVE_STK_INTENT: granted=true
        android.permission.WRITE_SETTINGS: granted=true
        android.permission.MODIFY_AUDIO_SETTINGS: granted=true
        android.permission.MANAGE_ACCOUNTS: granted=true
        android.permission.SYSTEM_ALERT_WINDOW: granted=true
    ......
    
    Shared users:
    SharedUser [android.uid.phone] (6df2b55):
      userId=1001
      install permissions:
        android.permission.SEND_RECEIVE_STK_INTENT: granted=true
        android.permission.WRITE_SETTINGS: granted=true
        android.permission.MODIFY_AUDIO_SETTINGS: granted=true
        android.permission.MANAGE_ACCOUNTS: granted=true
        android.permission.SYSTEM_ALERT_WINDOW: granted=true
    ......
    User 0: 
        gids=[1005, 3002, 3003, 3001, 3009]
        runtime permissions:
          android.permission.READ_SMS: granted=true, flags=[ GRANTED_BY_DEFAULT ]
          android.permission.READ_CALL_LOG: granted=true, flags=[ GRANTED_BY_DEFAULT ]
          android.permission.RECEIVE_SMS: granted=true, flags=[ GRANTED_BY_DEFAULT ]
          android.permission.READ_EXTERNAL_STORAGE: granted=true, flags=[ GRANTED_BY_DEFAULT ]

+ requested permissions: 该package申请的所有的　permission
+ install permissions: 安装时授予的权限(非runtime permission)
+ runtime permissions: runtime permission的授权状态

**runtime permission　后的　"flags=[ xxx ]"　值，　列出了该 permission 的flag**

####12. 在app中请求runtime permission

app中可以使用如下的方式来　检查/申请 runtime permission

    //检查给定的permission是否被授予
    Contex.checkSelfPermission()
    
    //请求获取指定的permission
    Activity.requestPermissions()

    //用于检查在请求permission前是否需要给出提示
    Activity.shouldShowRequestPermissionRationale()

    //permission请求的授权结果的回调方法
    Activity.onRequestPermissionsResult()
    
这些接口是在 android M 中新添加的，　如果app需要在　targetSdk　小于 23　来编译，　就会不通过，　最简单粗暴的解决方法可能就是利用Build.VERSION.SDK_INT >= 23这个判断语句来判断了, 方便的是,   support-v4 兼容库提供了如下的api,　可以支持同时在 targetsdk23 或旧的 sdk　编译运行

    //检查给定的permission是否被授予
    contextCompat.checkSelfPermission()
    
    //请求获取指定的permission
    ActivityCompat.requestPermission()
    
    //用于检查在请求permission前是否需要给出提示
    ActivityCompat.shouldShowRequestPermissionRationale()

    //permission请求的授权结果的回调方法
    ActivityCompat.onRequestPermissionsResult()    

示例

    // Here, thisActivity is the current activity
    if (ContextCompat.checkSelfPermission(thisActivity,
                    Manifest.permission.READ_CONTACTS)
            != PackageManager.PERMISSION_GRANTED) {

        // Should we show an explanation?
        if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
                Manifest.permission.READ_CONTACTS)) {

            // Show an expanation to the user *asynchronously* -- don't block
            // this thread waiting for the user's response! After the user
            // sees the explanation, try again to request the permission.

        } else {

            // No explanation needed, we can request the permission.

            ActivityCompat.requestPermissions(thisActivity,
                    new String[]{Manifest.permission.READ_CONTACTS},
                    MY_PERMISSIONS_REQUEST_READ_CONTACTS);

            // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
            // app-defined int constant. The callback method gets the
            // result of the request.
        }
    }
    
    
    @Override
    public void onRequestPermissionsResult(int requestCode,
            String permissions[], int[] grantResults) {
        switch (requestCode) {
            case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
                // If request is cancelled, the result arrays are empty.
                if (grantResults.length > 0
                    && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                    // permission was granted, yay! Do the
                    // contacts-related task you need to do.

                } else {

                    // permission denied, boo! Disable the
                    // functionality that depends on this permission.
                }
                return;
            }

            // other 'case' lines to check for other
            // permissions this app might request
        }
    }

runtime permission 请求弹框由 PackageInstaller 弹出：

1. 第一次请求permission时， 不会有 "不再提醒" 选项， 第二次请求permission时，才会有“不再提醒”的选项， 若用户一直拒绝， 并且没有选择“不再提醒”的选项， 下次请求permission时， 会继续有"不再提醒的选项"
2. 第一次请求permission时， 若用户拒绝了， 后续若继续点击'拒绝'， 则每一次调用　shouldShowRequestPermissionRationale() 均将返回true
3. 若在runtime permision请求弹框上勾选了“不再提醒”的选项时，并且点击了'拒绝', 后续调用 shouldShowRequestPermissionRationale() 将返回false
4. 若设备的策略禁止当前的应用获取这个permission的授权， 则 shouldShowRequestPermissionRationale() 一直返回false

###13. 系统组件的权限

通过在 AndroidManifest 使用“android:permission”属性， 可以设置所需的permission， 以限制对系统或者应用程序的组件的访问

####13.1 Activity permission

Activity permission 在 AndroidManifest.xml 中的 “&lt;activity&gt;”标签中声明， 用于限定启动该activity所需要的permission， permission检查在 `Context.startActivity()`和 `Activity.startActivityForResult()`中进行

####13.2 Service permission

Service permission 在 AndroidManifest.xml 中的 “&lt;service&gt;”标签中声明， 用于限定启动/连接到该service所需要的permission， permission检查在 `Context.startService()` ， `Context.stopService()` 和 `Context.bindService()`中进行

####13.3 BroadcastReceiver permission

BroadcastReceiver permission 在 AndroidManifest.xml 中的 “&lt;receiver&gt;”标签中声明, 用于限制谁能够发送广播到相关的receiver

BroadcastReceiver permission可以分为2种情况：

1. 有时候一些敏感的广播并不想让第三方的应用收到
2. 有时后需要限定广播的发送方， 防止伪造广播

由于 BroadcastReceiver permission 的检查将在`Context.sendBroadcast()`返回后进行，若permission检查失败， 也不会引发异常， 只是广播不会被分发到receiver， （若Sender 和 Receiver 有一方对某一要求BroadcastReceiver permission， 则必须双方都满足权限检查才能成功分发该广播）

使用 BroadcastReceiver permission 的步骤大约如下:

1. 在 Sender 或者 Receiver 的AndroidManifest.xml 中自定义所需的permission(若使用系统中已有的permission则可以忽略这一步)
   + 例如 `<permission android:name = "com.android.permission.XXX"/>`
2. 在 Sender app 的 AndroidManifest.xml 中声明使用该permission
   + 例如 `<uses-permission android:name="com.android.permission.XXX"></uses-permission>`
3. 在 Sender app 中发送广播时， 将所需的permission作为参数
   + 例如 `sendBroadcast("com.android.XXX_ACTION", "com.android.permission.XXX");`
4. 在 Receiver app的AndroidManifest.xml 中声明使用该permission
   + 例如 `<uses-permission android:name="com.android.permission.RECV_XXX"></uses-permission>`
5. 在 Receiver app的Androidmanifest.xml中的&lt;receiver&gt;tag里添加该权限的permission
   + 例如 `<receiver android:name=".XXXReceiver"   android:permission="com.android.permission.SEND_XXX"> `
   + 或者在 `Context.registerReceiver()` 中传入所需的参数也是等价的

####13.4 ContentProvider permissions

ContentProvider permissions AndroidManifest.xml 中的 “&lt;provider&gt;”标签中声明, 用于限定访问该provider中的数据所需要的permission

与上述的3种permission不同， provider具有2种分离的permission属性， 若一个 provider 受某个权限保护， 但是 accesser 并不具有该permission， 则会引发 accesser SecurityException 

1. android:readPermission
   + 在 `ContentResolver.query() ` 中检查
2. android:writePermission 
   + 在 `ContentResolver.insert()`, `ContentResolver.update()`, `ContentResolver.delete()` 中检查

###1４. URI permission

标准的permission系统对于content provider来说是不够的，一个content provider可能想保护它的读写权限，而同时与它对应的直属客户端也需要将特定的URI传递给其它应用程序，以便其它应用程序对该URI进行操作。一个典型的例子就是邮件程序处理带有附件的邮件。进入邮件需要使用permission来保护，因为这些是敏感的用户数据。然而，如果有一个指向图片附件的URI需要传递给图片浏览器，那个图片浏览器是不会有访问附件的权利的，因为他不可能拥有所有的邮件的访问权限。

针对这个问题的解决方案就是per-URI permission：当启动一个activity或者给一个activity返回结果的时候，呼叫方可以设置Intent.FLAG_GRANT_READ_URI_PERMISSION和/或 Intent.FLAG_GRANT_WRITE_URI_PERMISSION . 这会使接收该intent的activity获取到进入该Intent指定的URI的权限，而不论它是否有权限进入该intent对应的content provider

开放 URI permission 需要 Content Provider 在其 “&lt;provider&gt;” 标签中声明

+ `android:grantUriPermissions="true" ` 全局开关， 即该provider支持URI permission
+ ` <grant-uri-permissions>` 可以控制允许部分的URI访问权限， 例如 `<grant-uri-permission android:pathPrefix="/hello" />`

Context 提供2个方法

+ `public void grantUriPermission(String toPackage, Uri uri, int modeFlags)`
   + 为某个 package 添加访问 content Uri 的读或者写权限
+ `public void revokeUriPermission(Uri uri, int modeFlags)`
   + 移除所有对给定Uri的访问权限

###1５. permission检查的api

PackageManager 提供了一个接口用于检查指定的 package 是否具有给定的permission

     int checkPermission(String permName, String pkgName);
     
返回值为 `PackageManager.PERMISSION_GRANTED` 或者 `PackageManager.PERMISSION_DENIED`

ContextWrapper 提供了一些接口用来对外来的访问（包括自己）进行permission检查，具体实现在 ContextImpl 中 ，如果 package 接受到外来访问者的操作请求，那么可以调用这些接口进行permission检查。一般情况下可以把这些接口的检查接口分为两种

1. 返回错误的api
2. 抛出异常的api

####1５.1 返回错误的api

1. 检查权限, 返回值为 `PackageManager.PERMISSION_GRANTED` 或者 `PackageManager.PERMISSION_DENIED`
   + `int checkPermission(@NonNull String permission, int pid, int uid)`
      + 检查某个 uid 和 pid 是否有 permission 权限
   + ` int checkCallingPermission(String permission)`
      + 检查调用者是否有 permission 权限，如果调用者是自己那么返回 PackageManager.PERMISSION_DENIED
      + 通常用于在Service中检查调用者的permission
   + `int checkCallingOrSelfPermission(String permission)`
      + 检查自己或者其它调用者是否有 permission 权限

2. 检查Uri权限， 返回值为 `PackageManager.PERMISSION_GRANTED` 或者 `PackageManager.PERMISSION_DENIED`
   + `int checkUriPermission(Uri uri, int pid, int uid, int modeFlags)`
      + 检查给定 uid pid， 对于给定的URI是否具有给定的权限(读/写), 只有显式地授予权限才会检查成功， 否则， 即使默认具有权限， 也会报fail
   + `int checkCallingUriPermission(Uri uri, int modeFlags)`
      + 检查调用者是否对给定的uri具有给定的权限， 只有显式地授予权限才会检查成功， 否则， 即使默认具有权限， 也会报fail， 另外若调用者是自己， 也会fail
   + `int checkCallingOrSelfUriPermission(Uri uri, int modeFlags)`
      + 检查调用者或者自己， 是否对给定的uri具有给定的权限， 只有显式地授予权限才会检查成功， 否则， 即使默认具有权限， 也会报fail
   + `int checkUriPermission(Uri uri, String readPermission,String writePermission, int pid, int uid, int modeFlags)`
      + 相当于同时执行 checkPermission() 和 checkUriPermission()

####1５.2 抛出异常的api

主要有如下几组接口

1. 检查permission， 检查不通过则会抛出异常， 打印消息
   + `void enforcePermission(String permission, int pid, int uid, String message)`
      + 检查某个 uid 和 pid 是否有给定的 permission　授权 
   + `void enforceCallingPermission(String permission, String message)`
      + 检查调用者是否有给定 permission 授权，如果调用者是自己则不通过
   + `enforceCallingOrSelfPermission(String permission, String message)`    
      + 检查自己或者其它调用者是否有给定的 permission 授权
2. 检查Uri　permission，检查不通过则会抛出异常， 打印消息
   + `void enforceUriPermission(Uri uri, int pid, int uid, int modeFlags, String message)`
      + 检查给定 uid pid， 对于给定的URI是否具有给定的permission(读/写)　授权, 只有显式地授予permission才会检查成功， 否则， 即使默认具有permission， 也会检查不通过
   + `void enforceCallingUriPermission(Uri uri, int modeFlags, String message)`
      + 检查调用者是否对给定的uri具有给定的permission， 只有显式地授予permission才会检查成功， 否则， 即使默认具有permission， 也会检查不通过， 另外若调用者是自己， 也会检查不通过
   + `void enforceCallingOrSelfUriPermission(Uri uri, int modeFlags, String message)`
      + 检查调用者或者自己， 是否对给定的uri具有给定的permission， 只有显式地授予权限才会检查成功， 否则， 即使默认具有permission， 也会检查不通过
   + `void enforceUriPermission(Uri uri, String readPermission, String writePermission,int pid, int uid, int modeFlags, String message)`
      + 相当于同时执行 checkPermission() 和 checkUriPermission()
      
下面这一组和上面类似，如果遇到检查不通过时，会抛出异常，打印消息

###1６. permission检查的实现

ContextImpl.java 中提供的 API ，其实都是由 ActivityManagerService 中的如下几个接口进行的封装

    int checkPermission(String permission, int pid, int uid) throws RemoteException;
    int checkUriPermission(Uri uri, int pid, int uid, int mode) throws RemoteException；

以 checkPermission 为例， 其执行流程如下

1. ActivityManagerService.CheckPermission()
   + 如果 permission 为 null, 则返回 PackageManager.PERMISSION_DENIED
2. ActivityManagerService.checkComponentPermission()
   + 如果 pid 为system server 的pid， 则不做控制， 返回 PackageManager.PERMISSION_GRANTED
3. ActivityManager.checkComponentPermission()
   + 如果 uid 为 root 或者 system 则不做控制， 返回 PackageManager.PERMISSION_GRANTED
   + 如果是被隔离的uid (99000 ~ 99999) , 则不授予任何权限， 返回 PackageManager.PERMISSION_DENIED
   + 如果被检查权限的uid和 定义该权限的app的uid相同， 则不做控制， 返回 PackageManager.PERMISSION_DENIED
   + 如果 permissiion 为null则返回 PackageManager.PERMISSION_DENIED
4. PackageManagerService.checkUidPermission()
   + 调用 Settings.getUserIdLPr() 去 PackageManagerService.Setting.mUserIds 数组中，根据 uid 查找 uid （也就是 package ）的权限列表。一旦找到，就表示有相应的权限
   + 如果没有找到，那么再去 PackageManagerService.mSystemPermissions 中找。这些信息是启动时，从 /system/etc/permissions/platform.xml 中读取的。这里记录了一些系统级的应用的 uid 对应的 permission