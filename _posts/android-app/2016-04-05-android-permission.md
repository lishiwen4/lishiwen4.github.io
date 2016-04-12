---
layout: post
title: "android permission"
description:
category: android-app
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. Security Architecture(安全体系结构)

Android安全体系结构的核心是：

默认情况下没有任何应用有权限去执行对其他应用、操作系统、用户有不利影响的操作。这是一个核心的设计理念

因为每一个Android 应用都是在一个进程沙盒中运行的，应用必须明确分享的资源和数据，通过申明需要的额外权限这种方式（这些额外权限不由基本沙河提供）。应用静态的声明这些权限(在Manifest里面),然后Android系统会请求用户同意这些权限


###2. Application signing(应用签名)

所有的Apk文件都必须由他的开发者使用私有的签名证书签名，这个证书是开发者身份的唯一标识，这个证书是由开发者自己生成的开放式的证书，用于自己签名应用。证书的目的是标识应用的身份，限制对于程序的修改仅限于同一来源， 签名相关文件可以从 apk 包中的 META-INF 目录下找到

系统中主要有两个地方会检查 ：

+ 如果是程序升级的安装，则要检查新旧程序的签名证书 是否一致，如果不一致则会安装失败
+ 对于申请权限的 protectedlevel 为 signature 或者 signatureorsystem 的，会检查权限申请者和权限声明者的证书是否是一致的。

####2.1 签名步骤

签名工具 signapk.jar 的源代码在 build/tools/signapk ，签名主要有以下几步

1. 将除去 CERT.RSA ， CERT.SF ， MANIFEST.MF 的所有文件生成 SHA1 签名
   + 首先将除了 CERT.RSA ， CERT.SF ， MANIFEST.MF 之外的所有非目录文件分别用 SHA-1 计算摘要信息
   + 然后使用 base64 进行编码，存入 MANIFEST.MF 中，如果 MANIFEST.MF 不存在，则需要创建。存放格式是 entry name 以及对应的摘要
2. 根据 之前计算的 SHA1 摘要信息，以及 私钥生成 一系列的 signature 并写入 CERT.SF
   + 对 整个 MANIFEST.MF 进行 SHA1 计算，并将摘要信息存入 CERT.SF 中 
   + 然后对之前计算的所有摘要信息使用 SHA1 再次计算数字签名，并写入 CERT.SF 中
3. 把公钥和签名信息写入 CERT.RST
   + 把之前整个的签名输出文件 使用私有密钥计算签名
   + 将签名结果，以及之前声称的公钥信息写入 CERT.RSA 中保存
   
####2.2 签名的验证步骤
   
安装时对一个 package 的签名验证的主要逻辑在 JarVerifier.java 文件的 verifyCertificate() 函数中实现。 其主要的思路是通过提取 cert.rsa 中的证书和签名信息，获取签名算法等信息，然后按照之前对 apk 签名的方法进行计算，比较得到的签名和摘要信息与 apk 中保存的匹配

+ 第一步 提取证书信息，并对 cert.sf 进行完整性验证 ： 
   1. 先找到是否有 DSA 和 RSA 文件 ，如果找到则对其进行 decode ，然后读取其中的所有的证书列表（这些证书会被保存在 Package 信息中，供后续使用）。
   2. 读取这个文件中的签名数据信息块列表，只取第一个签名数据块。读取其中的发布者和证书序列号。
   3. 根据证书序列号，去匹配之前得到的所有证书，找到与之匹配的证书。
   4. 从之前得到的签名数据块中读取签名算法和编码方式等信息
   5. 读取 cert.sf 文件，并计算整个的签名，与数据块中的签名（编码格式的）进行比较，如果相同则完整性校验成功。
+ 第二步使用 cert.sf 中的摘要信息，验证 MANIFEST.MF 的完整性 ： 在 cert.sf 中提取 SHA1-Digest-Manifest 或者 SHA1-Digest 开头的签名 数据块 （ -Digest-Manifest 这个是整个 MANIFEST.MF 的摘要 信息，其它的是 jar 包中其它文件的摘要信息 ）， 并逐个对这些数据块 进行验证。验证的方法是，现将 cert.sf 看做是很多的 entries ，每个 entries 包含了一些基本信息，如这个 entry 中使用的摘要算法（ SHA1 等），对 jar 包中的哪个文件计算了摘要，摘要结果是什么。 处理时先找到每个摘要数据开中的文件信息，然后从 jar 包中读取，然后使用 -Digest 之前的摘要算法进行计算，如果计算结果与摘要数据块中保存的信息的相匹配，那么就完成验证。

####2.3 android系统默认的4把key

build/target/product/security目录中有四组默认签名供Android.mk在编译APK时使用

1. testkey：普通APK，默认情况下使用
2. platform：该APK完成一些系统的核心功能， 如需要share user ID 为 system时， 就需要使用platform key 来签名
   + 例如 Settings, Bluetooth, Mms 等使用 platform key 来签名
3. shared：该APK需要和home/contacts进程共享数据
   + 例如 Dialer， Contacts， QuickSearchBox， Launcher 等使用 shared key 签名
4. media：该APK是media/download系统中的一环 
   + 例如 Gallery， MediaProvider， DownloadProvider 等使用 media key 签名

应用程序的Android.mk中有一个LOCAL_CERTIFICATE字段，由它指定用哪个key签名，未指定的默认用testkey


###3. User ID

在安装一个app package的时候，android系统会给每一个package一个独立的Linux user ID。这个User ID在这个应用在当前设备的生命周期内都是固定不变的，在不同的设备，相同的package的用户ID可能各不相同，但可以确定的是在一台设备上一个package的用户ID是固定不变的

“/data/system/packages.xml” 记录了每一个apk文件的uid， 例如

    # grep systemui packages.xml

        <item name="com.android.systemui.permission.SELF" package="com.android.systemui" protection="2" />
    <package name="com.android.systemui" codePath="/system/priv-app/SystemUI" nativeLibraryPath="/system/priv-app/SystemUI/lib" publicFlags="945307149" privateFlags="8" ft="11e8dc5d800" it="11e8dc5d800" ut="11e8dc5d800" version="1510603000" sharedUserId="10015">
            <item name="com.android.systemui.permission.SELF" granted="true" flags="0" />
    <shared-user name="android.uid.systemui" userId="10015">
            <item name="com.android.systemui.permission.SELF" granted="true" flags="0" />

    可见systemui的UID是10015， ps查看的结果如下：
    
    # ps | grep systemui
    u0_a15    2547  350   1115292 92780 SyS_epoll_ b6c67e74 S com.android.systemui
    
普通 Android 应用程序的 uid 是从 10000 开始分配 （参见 Process.FIRST_APPLICATION_UID ）， 10000 以下是系统进程的 uid， 因此u0_a15 正好对应UID 10015

###4. shared User ID

通过Shared User id,拥有同一个User id的多个APK可以

+ 配置成运行在同一个进程中. 默认就是可以互相访问任意数据
+ 配置成运行在不同的进程中, 同时可以访问其他APK的数据目录下的数据库和文件， 就像访问本程序的数据一样

一个apk如果希望和其他的apk共享UID， 需要做如下的工作

1. 在Manifest节点中增加&lt;android:sharedUserId=xxxxx&gt;属性。
2. 在Android.mk中增加 "LOCAL_CERTIFICATE:=xxxx" 的定义

share同一UID的apk必须具有相同的签名， 否则将不能安装， 提示类似于如下的信息

    Package com.test.MyTest has no signatures that match those in shared user android.uid.system; ignoring!
    
当然为了保证安全，只有两个APP签名一致且申明了相同的sharedUserId才会被给予相同的User ID


###5. Using Permission

一个新建的Android应用默认是没有权限的，这意味着它不能执行任何可能对用户体验有不利影响的操作或者访问设备数据。为了使用受保护的功能，你必须包含一个或者多个<uses-permission>标签在你的app manifest中， 例如

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.android.app.myapp" >
        <uses-permission android:name="android.permission.RECEIVE_SMS" />
        ...
    </manifest>

Android 6.0中权限分为4种

+ 低级权限
  1. normal : 普通权限, 这些权限对于用户隐私和设备操作不会造成太多危险（比如开启手电筒的权限），apk在AndroidManifest中申明后， 系统会自动授予这些权限
  2. dangerous : 危险权限， 这些权限可能会影响用户隐私和设备的普通操作（请求获取用户联系人信息的权限）， 系统会明确的让用户决定是否授予这些权限(Android 6.0以前， 在安装apk时授予这些权限， Android 6.0后， 可在安装后动态开关这些权限， 因此又称为运行时权限)
+ 高级权限
  1. Signature : 签名级别的权限， 即申请该权限的apk必须和声明该权限的apk具有相同的签名才能申请的权限， 若签名匹配， 系统会自动授予这些权限
  2. Signature or System ： 当apk被内置在System Image或者和System Image使用相同的签名才能申请的权限

拥有system级别权限的使用者可以access其他普通signature权限声明设定过的功能

**权限检查中有些特例， 可以参见第13节**

###6. 系统定义的权限

Android 系统定义的权限位于源码的 “framework/base/core/res/AndroidManifest.xml”中


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
    
可以看到， 系统定义了权限组， 但是有些权限是未分组的


###7. 自定义权限


为了自定义权限，你必须在你的manifest中声明一个或多个<permission>标签

比如：

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.me.app.myapp" >
        <permission android:name="com.me.app.myapp.permission.DEADLY_ACTIVITY"
            android:label="@string/permlab_deadlyActivity"
            android:description="@string/permdesc_deadlyActivity"
            android:permissionGroup="android.permission-group.COST_MONEY"
            android:protectionLevel="dangerous" />
    ...
    
    </manifest>

+ protectionLevel ： 属性是必须的，告诉系统当app申请该权限的时候，要怎样通知用户。
+ permissionGroup ： 属性是可选的，可以帮助系统显示自定义属性属于哪个权限组，当通知用户弹出框的时候，当然你可以选择某一个自定义权限属于已知的权限组，也可以属于某一个自定义权限组，建议属于已知的权限组。
+ android:label ： 相当于权限组的提示，要简短
+ android:description ： 是某一个特定权限的描述，规则是两句话，第一句描述，第二句警告用户如果授权会发生什么后果

###8. 查看权限

####8.1 查看系统中的权限
使用 `adb shell pm list permissions` 命令可以查看系统中所有声明的权限， 可以使用如下选项：

+ -g ： 按权限组显示
+ -f ： 显示详细信息， 包括protect level， description， 以及定义该权限的package等信息

例如

    # pm pm list permissions -g
    All Permissions:

    group:com.google.android.gms.permission.CAR_INFORMATION
        permission:com.google.android.gms.permission.CAR_VENDOR_EXTENSION
        ......
    
    group:android.permission-group.CONTACTS
        permission:android.permission.WRITE_CONTACTS
        permission:android.permission.GET_ACCOUNTS
        permission:android.permission.READ_CONTACTS
        ......
    
    group:android.permission-group.PHONE
        ......

    group:android.permission-group.CALENDAR
        ......
        
    group:android.permission-group.CAMERA
        ......

    group:android.permission-group.SENSORS
        ......

    group:android.permission-group.LOCATION
        ......

    group:android.permission-group.STORAGE
        ......
        
    group:android.permission-group.MICROPHONE
        ......
        
    group:android.permission-group.SMS
        ......

    ungrouped:
        ......
        

`pm list permission-groups` 命令可以只列出权限组


####8.2 查看apk的权限

目前已有多个工具可以静态检测Android app所具有的Permissions，这类工具有：aapt、apktool、androguard等等， 以 aapt (android 源码可以build出该host tool)为例

    $ ./aapt dump permissions xxx.apk
    package: com.xxx.wallpaper
    uses-permission: name='android.permission.READ_PHONE_STATE'
    uses-permission: name='android.permission.INTERNET'
    uses-permission: name='android.permission.ACCESS_NETWORK_STATE'
    uses-permission: name='android.permission.WRITE_EXTERNAL_STORAGE'
    ......
    
###9. 运行时权限

运行时权限的权限组及组内权限:

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

“运行时权限”的控制粒度为组， 即只要获取到了某一权限组中的任意一个权限， 即系统会默认为已授予了该组内的所有权限， 后续申请该组内的其它权限时， 不再弹出提示框

有两个权限比较特殊，分别是SYSTEM_ALERT_WINDOW 和 WRITE_SETTINGS

都属于比较敏感的权限，很多时候都用不到，如果需要这个权限，应该首先声明，然后发送Intent去请求用户授权，然后系统会显示详细的管理界面


###10.  系统组件的权限

通过在 AndroidManifest 使用“android:permission”属性， 可以设置高级权限， 以限制对系统或者应用程序的组件的访问

####10.1 Activity permission

Activity permission 在 AndroidManifest.xml 中的 “&lt;activity&gt;”标签中声明， 用于限定启动该activity所需要的权限， 权限检查在 `Context.startActivity()`和 `Activity.startActivityForResult()`中进行

####10.2 Service permission

Service permission 在 AndroidManifest.xml 中的 “&lt;service&gt;”标签中声明， 用于限定启动/连接到该service所需要的权限， 权限检查在 `Context.startService()` ， `Context.stopService()` 和 `Context.bindService()`中进行

####10.3 BroadcastReceiver permission

BroadcastReceiver permission 在 AndroidManifest.xml 中的 “&lt;receiver&gt;”标签中声明, 用于限制谁能够发送广播到相关的receiver

BroadcastReceiver permission可以分为2种情况：

1. 有时候一些敏感的广播并不想让第三方的应用收到
2. 有时后需要限定广播的发送方， 防止伪造广播

由于 BroadcastReceiver permission 的检查将在`Context.sendBroadcast()`返回后进行，若权限检查失败， 也不会引发异常， 只是广播不会被分发到receiver， （若Sender 和 Receiver 有一方对某一要求BroadcastReceiver permission， 则必须双方都满足权限检查才能成功分发该广播）

使用 BroadcastReceiver permission 的步骤大约如下:

1. 在 Sender 或者 Receiver 的AndroidManifest.xml 中自定义所需的权限(若系统中已有的权限则可以忽略这一步)
   + 例如 `<permission android:name = "com.android.permission.XXX"/>`
2. 在 Sender app 的 AndroidManifest.xml 中声明使用该权限
   + 例如 `<uses-permission android:name="com.android.permission.XXX"></uses-permission>`
3. 在 Sender app 中发送广播时， 将所需的权限作为参数
   + 例如 `sendBroadcast("com.android.XXX_ACTION", "com.android.permission.XXX");`
4. 在 Receiver app的AndroidManifest.xml 中声明使用该权限
   + 例如 `<uses-permission android:name="com.android.permission.RECV_XXX"></uses-permission>`
5. 在 Receiver app的Androidmanifest.xml中的&lt;receiver&gt;tag里添加该权限的声明
   + 例如 `<receiver android:name=".XXXReceiver"   android:permission="com.android.permission.SEND_XXX"> `
   + 或者在 `Context.registerReceiver()` 中传入所需的参数也是等价的

####10.4 ContentProvider permissions

ContentProvider permissions AndroidManifest.xml 中的 “&lt;provider&gt;”标签中声明, 用于限定访问该provider中的数据所需要的权限

与上述的3种permission不同， provider具有2种分离的permission属性， 若一个 provider 受某个权限保护， 但是 accesser 并不具有该权限， 则会引发 accesser SecurityException 

1. android:readPermission
   + 在 `ContentResolver.query() ` 中检查
2. android:writePermission 
   + 在 `ContentResolver.insert()`, `ContentResolver.update()`, `ContentResolver.delete()` 中检查

###11. URI permission

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

###12. 权限检查的api

PackageManager 提供了一个接口用于检查指定的 package 是否具有给定的权限

     int checkPermission(String permName, String pkgName);
     
返回值为 `PackageManager.PERMISSION_GRANTED` 或者 `PackageManager.PERMISSION_DENIED`


ContextWrapper 提供了一些接口用来对外来的访问（包括自己）进行权限检查，具体实现在 ContextImpl 中 ，如果 package 接受到外来访问者的操作请求，那么可以调用这些接口进行权限检查。一般情况下可以把这些接口的检查接口分为两种

1. 返回错误的api
2. 跑出异常的api

####12.1 返回错误的api

1. 检查权限, 返回值为 `PackageManager.PERMISSION_GRANTED` 或者 `PackageManager.PERMISSION_DENIED`
   + `int checkPermission(@NonNull String permission, int pid, int uid)`
      + 检查某个 uid 和 pid 是否有 permission 权限
   + ` int checkCallingPermission(String permission)`
      + 检查调用者是否有 permission 权限，如果调用者是自己那么返回 PackageManager.PERMISSION_DENIED
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

####12.2 抛出异常的api

主要有如下几组接口

1. 检查权限， 检查不通过则会抛出异常， 打印消息
   + `void enforcePermission(String permission, int pid, int uid, String message)`
      + 检查某个 uid 和 pid 是否有 permission 权限
   + `void enforceCallingPermission(String permission, String message)`
      + 检查调用者是否有 permission 权限，如果调用者是自己则不通过
   + `enforceCallingOrSelfPermission(String permission, String message)`    
      + 检查自己或者其它调用者是否有 permission 权限
2. 检查Uri权限，检查不通过则会抛出异常， 打印消息
   + `void enforceUriPermission(Uri uri, int pid, int uid, int modeFlags, String message)`
      + 检查给定 uid pid， 对于给定的URI是否具有给定的权限(读/写), 只有显式地授予权限才会检查成功， 否则， 即使默认具有权限， 也会检查不通过
   + `void enforceCallingUriPermission(Uri uri, int modeFlags, String message)`
      + 检查调用者是否对给定的uri具有给定的权限， 只有显式地授予权限才会检查成功， 否则， 即使默认具有权限， 也会检查不通过， 另外若调用者是自己， 也会检查不通过
   + `void enforceCallingOrSelfUriPermission(Uri uri, int modeFlags, String message)`
      + 检查调用者或者自己， 是否对给定的uri具有给定的权限， 只有显式地授予权限才会检查成功， 否则， 即使默认具有权限， 也会检查不通过
   + `void enforceUriPermission(Uri uri, String readPermission, String writePermission,int pid, int uid, int modeFlags, String message)`
      + 相当于同时执行 checkPermission() 和 checkUriPermission()
      
下面这一组和上面类似，如果遇到检查不通过时，会抛出异常，打印消息

###13. 权限检查的实现

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