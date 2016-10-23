---
layout: post
title: "Intent"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###１. Intent

Intent是一种运行时绑定（runtime binding)机制，它能在程序运行的过程中连接两个不同的组件。通过Intent，你的程序可以向Android系统表达某种请求或者意愿，Android会根据意愿的内容选择适当的组件来响应,如果有多个条件符合的组件，就以列表的方式让用户手动选择一个.

Intent不仅可用于应用程序之间，也可用于应用程序内部的activity, service和broadcast receiver之间的交互

Intent 常见的几个应用场景如下:

1. 启动一个 Activity : `Context.startActivity(Intent intent)`
2. 启动一个 Service : `Context.startService(Intent intent）`
3. 绑定一个 Service :　`Context.bindService(Intent service, ServiceConnection conn, int flag)`
4. 发送一个 Broadcast : `Context.sendBroadcast(Intent intent)`

Android 的　4大组件中，　activity、service　和　broadcast receiver　之间是通过Intent进行通信的，而另外一个组件Content Provider，　本身就是一种通信机制，不需要通过Intent

###2. Intent 的相关属性

Intent　对象包含如下的属性：

1. action : 动作，　用来表现意图的行动
2. category : 类别，　用来表现动作的类别
3. extra : 扩展信息, 
4. type : 数据类型, data范例的描写
5. component :　组件，　目标组件
6. data :　数据，　表示要操纵的数据
7. flag :　标志位，　期望这个inten　运行的模式

###3. 显式Intent和隐式Intent

+ 显式 Intent : Intent　对象中指定了 component 属性　(即明确指定了由哪一个组件来处理该Intent)
+ 隐式 Intent : Intent　对象中未指定 component 属性, 而是指定一系列更为抽象的action和category等其它属性

相比与显式Intent，隐式Intnet则含蓄了许多，它并不明确指出目的组件，而是指定一系列更为抽象的信息，然后交由系统去分析这个Intent，并帮我们找出合适的组件

隐式 Intent 的匹配过程需要借助 IntentFilter

###4. component(组件)

Component属性明确指定Intent的目标组件的类名称，Intent对象中，　指定了　Component　属性以后，　其它的属性是可选的

Intent对象可以在构造时指定　component　的package和class

    public Intent(Context packageContext, Class<?> cls);
    
或者在实例化Intent对象后，实例化component对象并为Intent对象设置component

    public ComponentName(String pkg, String cls);
    public ComponentName(Context pkg, String cls);
    public ComponentName(Context pkg, Class<?> cls);
    public Intent setComponent(ComponentName component);

或者实例化Intent对象后, 分别指定component信息

    public Intent setPackage(String packageName)；
    public Intent setClassName(Context packageContext, String className)；
    public Intent setClassName(String packageName, String className);
    public Intent setClass(Context packageContext, Class<?> cls)；
    
例如，　直接指明要启动activity

    Intent intent = new Intent();
    ComponentName component = new ComponentName(MainActivity.this, SecondActivity.class);
    intent.setComponent(component);
    startActivity(intent);

    //和上面等价
    Intent intent = new Intent(MainActivity.this, SecondActivity.class);
    startActivity(intent);
    
###5. action(动作)

Action 是一个用户定义的字符串，用于描述一个 Android 应用程序组件, 每一个隐式 Intent　必须并且只能具有一个 action　属性

使用 Intent　的如下构造函数，　即可传递action参数

    public Intent(String action);
    
或者使用如下的方法为 Intent　对象设置 action

    public Intent setAction(String action);
    
可以使用如下的方法从 Intent　对象获取 action　属性

    public String getAction();
    
Intent对象中有预定义一些 action，　其中一个比较重要的是　Intent.ACTION_MAIN

    public static final String ACTION_MAIN = "android.intent.action.MAIN";
    
Intent.ACTION_MAIN action 指定了一个app的主入口，　即打开app时启动的第一个activity

###6. category(类别)

一个　Intent　对象可以有任意个 category

可以使用如下的方式来为 Intent　对象添加一个　category

    public Intent addCategory(String category)
    
可以使用如下的方式来为 Intent　对象移除一个　category

    public void removeCategory(String category)
    
可以使用如下的方式来获取 Intent　对象的所有　category

    public Set<String> getCategories()
    
Intent对象中有预定义一些 action，　其中几个比较重要的有

    public static final String CATEGORY_HOME = "android.intent.category.HOME";
    public static final String CATEGORY_DEFAULT = "android.intent.category.DEFAULT";
    public static final String CATEGORY_LAUNCHER = "android.intent.category.LAUNCHER";
    
+ Intent.CATEGORY_HOME : 设备启动后显示的第一个　activity，　
+ Intent.CATEGORY_DEFAULT : 每一个通过 startActivity() 方法发出的隐式 Intent 都至少有一个 category，　就是"android.intent.category.DEFAULT", 因此每一个想要接收隐式 Intent 的 Activity组件，　必须在　intent-filter　中声明该category
+ Intent.CATEGORY_LAUNCHER : 应用程序显示在app列表里面, 即launcher上显示图标, 必须和Intent.ACTION_MAIN　配合才有作用，　若一个app中有多个activity设置了 Intent.CATEGORY_LAUNCHER 和 Intent.ACTION_MAIN, 则在launcher上会有多个图标
，　但是点击任何图标，　开启的都是最先定义 Intent.CATEGORY_LAUNCHER 和 Intent.ACTION_MAIN　的那个 activity

###7. data(数据)

Data是用一个uri对象来表示的，uri代表数据的地址，属于一种标识符

使用 Intent　的如下构造函数，　可以直接传递 data

    public Intent(String action, Uri uri);
    
还可以使用如下的方法来为Intent对象添加data

    public Intent setData(Uri data);
    
例如, 调用浏览器打开百度的主页:

    Intent intent = new Intent(Intent.ACTION_VIEW);
    intent.setData(Uri.parse("http://www.baidu.com"));                
    startActivity(intent);

###8. type(数据类型)

Type属性用于明确指定Data属性的数据类型或MIME类型

可以使用如下的方法为 Intent　对象指定其data属性的type

    public Intent setType(String type);

####8.1 MIME type

多用途互联网邮件扩展（MIME，Multipurpose Internet Mail Extensions）是一个互联网标准，它扩展了电子邮件标准，使其能够支持非ASCII字符、二进制格式附件等多种格式的邮件消息。一般以下面的形式出现:

    [type]/[subtype]

type有下面的形式:

+ text : 用于标准化地表示的文本信息，文本消息可以是多种字符集和或者多种格式的；
+ multipart : 用于连接消息体的多个部分构成一个消息，这些部分可以是不同类型的数据；
+ application : 用于传输应用程序数据或者二进制数据；
+ message : 用于包装一个E-mail消息；
+ image : 用于传输静态图片数据；
+ audio : 用于传输音频或者音声数据；
+ video : 用于传输动态影像数据，可以是与音频编辑在一起的视频数据格式

而subtype用于指定type的详细形式, android 支持多种MIME, 定义在源码的　libcore　中的　MimeUtils　中, 见末尾附表

####8.2 Content provider type

无论数据的来源是什么，android  content provider　都会将数据组织成表格， 在从　content provider　获取数据时，content provider　的 getType()　方法可以返回指定的URI的数据的MIME type,　对于返回的MIME (type/subtype)，　有如下的规则

+ type : 取值为如下的２者之一:
   + "vnd.android.cursor.item" : 返回的是单条数据 (数据表中的一行)
   + "vnd.android.cursor.dir" : 返回的是多条数据 (数据表中的多行) 的集合
   + “vnd”　其实是　vendor-specific，　代表时厂商自定义的数据类型
+ subtype : 根据实际情况，　可以是标准的类型，也可以是自定义的类型

###9. data 和 type

data和type属性一般只需要一个，通过setData方法会把type属性设置为null，相反设置setType方法会把data设置为null, 在设置data时， Android系统将会根据Data属性值来分析出type，所以无需指定type属性

在某些情况下，　需要同时设置 data　和　type，　如果想要两个属性同时设置，要使用Intent.setDataAndType()方法

    public Intent setDataAndType(Uri data, String type);
    
例如:

    intent.setAction(Intent.ACTION_VIEW);
    Uri data = Uri.parse("file:///storage/sdcard0/abcdefg.mp3");
    intent.setDataAndType(data, "audio/mp3");
    startActivity(intent);                

###10. extras(扩展信息)

是其它所有附加信息的集合, 使用extras可以为组件提供扩展信息,诸如用户名等信息, extra信息类似于一个　键-值 对，　由"名称"和"值"构成，　extra信息并不是必须的, 每个Intent可以根据需要附加多个extra信息,　接收该 Intent的组件可以取出这些extra信息

Intent　对象提供了如下的方法用于添加　不同数据类型的　extra：

    public Intent putExtra(String name, boolean value);
    public Intent putExtra(String name, boolean[] value);
    
    public Intent putExtra(String name, byte value);
    public Intent putExtra(String name, byte[] value);
    
    public Intent putExtra(String name, char value);
    public Intent putExtra(String name, char[] value);
    
    public Intent putExtra(String name, short value);
    public Intent putExtra(String name, short[] value);
    
    public Intent putExtra(String name, int value);
    public Intent putExtra(String name, int[] value);
    public Intent putIntegerArrayListExtra(String name, ArrayList<Integer> value);
    
    public Intent putExtra(String name, long value);
    public Intent putExtra(String name, long[] value);
    
    public Intent putExtra(String name, float value);
    public Intent putExtra(String name, float[] value);
    
    public Intent putExtra(String name, double value);
    public Intent putExtra(String name, double[] value);
        
    public Intent putExtra(String name, String value);
    public Intent putExtra(String name, String[] value);
    public Intent putStringArrayListExtra(String name, ArrayList<String> value);
    
    public Intent putExtra(String name, CharSequence value);
    public Intent putExtra(String name, CharSequence[] value);
    public Intent putCharSequenceArrayListExtra(String name, ArrayList<CharSequence> value);
    
    public Intent putExtra(String name, Parcelable value);
    public Intent putExtra(String name, Parcelable[] value);    
    public Intent putParcelableArrayListExtra(String name, ArrayList<? extends Parcelable> value);
    
    public Intent putExtra(String name, Serializable value)
    
    public Intent putExtra(String name, Bundle value)
    public Intent putExtra(String name, IBinder value)
    public Intent putExtras(Intent src)
    public Intent putExtras(Bundle extras)
    public Intent putExtras(Intent src)

Intent 对象提供了如下的方法用于移除某个　extra:

    public void removeExtra(String name);
    
Intent 对象提供了如下的方法用于所有非基本数据类型的 extra　信息
    
    public void removeUnsafeExtras()
    
Intent 对象提供了如下的方法用于检查是否具有指定名称的 extra信息

    public boolean hasExtra(String name)
    
Intent　对象提供了如下的方法用于获取不同数据类型的　extra:

    public Object getExtra(String name)
    
    public boolean getBooleanExtra(String name, boolean defaultValue);
    public boolean[] getBooleanArrayExtra(String name);
        
    public byte getByteExtra(String name, byte defaultValue)
    public byte[] getByteArrayExtra(String name);
        
    public short getShortExtra(String name, short defaultValue);
    public short[] getShortArrayExtra(String name);
    
    public char getCharExtra(String name, char defaultValue)
    public char[] getCharArrayExtra(String name)
        
    public int getIntExtra(String name, int defaultValue)
    public int[] getIntArrayExtra(String name)
    public ArrayList<Integer> getIntegerArrayListExtra(String name)
        
    public long getLongExtra(String name, long defaultValue)
    public long[] getLongArrayExtra(String name)
        
    public float getFloatExtra(String name, float defaultValue)
    public float[] getFloatArrayExtra(String name)
        
    public double getDoubleExtra(String name, double defaultValue)
    public double[] getDoubleArrayExtra(String name)
        
    public String getStringExtra(String name)
    public String[] getStringArrayExtra(String name)
    public ArrayList<String> getStringArrayListExtra(String name)
        
    public CharSequence getCharSequenceExtra(String name)
    public CharSequence[] getCharSequenceArrayExtra(String name)
    public ArrayList<CharSequence> getCharSequenceArrayListExtra(String name)
        
    public <T extends Parcelable> T getParcelableExtra(String name)
    public Parcelable[] getParcelableArrayExtra(String name)
    public <T extends Parcelable> ArrayList<T> getParcelableArrayListExtra(String name)
    
    public Serializable getSerializableExtra(String name)
    
    public Bundle getBundleExtra(String name)
    
    public IBinder getIBinderExtra(String name)
    
    public Object getExtra(String name, Object defaultValue)
    
    public Bundle getExtras()
    
###11. flags(标志位)

flags　属性用于为目标组件设置相关的flag

Intent对象提供了如下的方法来设置和添加flag :

    public Intent setFlags(int flags);
    public Intent addFlags(int flags);

Intent对象提供了如下的方法来获取flag :
     
    public int getFlags();

Intent 中定义了多个 flag

URI permission　相关的 flag：

+ FLAG_GRANT_READ_URI_PERMISSION : 授予目标组件对　Intent　中的data属性携带的uri的读权限
+ FLAG_GRANT_WRITE_URI_PERMISSION : 授予目标组件对　Intent　中的data属性携带的uri的写权限
+ FLAG_GRANT_PERSISTABLE_URI_PERMISSION : 和上２个flag配合使用时，　授予的权限在重启后仍然生效，　直到明确取消授权
+ FLAG_GRANT_PREFIX_URI_PERMISSION　: 和前２个flag配合使用时，　授予目标组件对所有以　Intent　中的data属性携带的uri开头的uri的读/写权限
  
activity相关的flag:
    
+ FLAG_ACTIVITY_NO_HISTORY : 目标activity将不会被保存在 history statck 中，当从该Activity中离开时，　该Activity即被finish
+ FLAG_ACTIVITY_SINGLE_TOP :　若目标Activity已经在history stack的最上层运行，　则不再重新launch
+ FLAG_ACTIVITY_NEW_TASK :　目标Activity会成为history stack中一个新Task的开始，一个Task（从启动它的Activity到下一个Task中的Activity）定义了用户可以迁移的Activity原子组。Task可以移动到前台和后台；在某个特定Task中的所有Activity总是保持相同的次序
   + 这个标志通常被launcher所使用：它们提供用户一系列可以单独完成的事情，与启动它们的Activity完全无关
   + 使用这个标志，如果正在启动的Activity的Task已经在运行的话，那么，新的Activity将不会启动；代替的，当前Task会简单的移入前台。参考FLAG_ACTIVITY_MULTIPLE_TASK标志，可以禁用这一行为
   + 这个标志不能用于调用方对已经启动的Activity请求结果
+ FLAG_ACTIVITY_MULTIPLE_TASK :　用于建立一个新的task并且launche一个activity
   + 通常和　FLAG_ACTIVITY_NEW_DOCUMENT　或者　FLAG_ACTIVITY_NEW_TASK　配合使用
   + 除非是在实现launcher，　否则，　不要使用该flag
+ FLAG_ACTIVITY_CLEAR_TOP :　设置该flag后，　若目标activity已经在当前的Task中运行，那么，将不会重新启动一个目标Activity的实例，而是在将标Activity上方的所有Activity都关闭，然后这个Intent会作为一个新的Intent投递到目标Activity（现在位于顶端）中
+ FLAG_ACTIVITY_FORWARD_RESULT :设置该flag后，　目标activty的回传值将传递给设置这一flag的activity的前一个activity，　例如 activity　Ａ -> B -> C, B作为过渡页面,启动C时设置了该flag，然后自己finish，　则在C中设置的回传值将传递给Ａ,当然,这其中可以存在多个类似于B的过渡页面
+ FLAG_ACTIVITY_PREVIOUS_IS_TOP :　设置该flag后，　当前activity将不会被视作栈顶的activity来进行判断，　而是用之前的activity来作为栈顶的activity，当前的activity将会立即自己finish掉。典型的场景是在应用选择页面，如果在文本中点击一个网址要跳转到浏览器，而系统中又装了不止一个浏览器应用，此时会弹出应用选择页面。在应用选择页面选择某一款浏览器启动时，就会用到这个Flag。然后应用选择页面将自己finish，以保证从浏览器返回时不会在回到选择页面
+ FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS :　设置该flag之后, 目标activity将不会在recent　activities列表中保存
+ FLAG_ACTIVITY_BROUGHT_TO_FRONT :　这一flag一般不是由程序代码设置的，如在launchMode中设置singleTask模式时系统帮你设定
+ FLAG_ACTIVITY_RESET_TASK_IF_NEEDED :　设置该flag后，无论目标activity是在已给新的task中被launch, 还是从一个已存在的task中被移动到最上层,系统会根据affinity对指定的task进行重置操作, task会压入某些activity实例或者移除activity实例
+ FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY : 之一flag由系统而非应用来设定, 在从历史记录里面启动activity时会被设定该flag
+ FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET :　设置该flag后，将在Task的Activity stack中设置一个还原点，当Task恢复时，需要清理Activity，也就是说，下一次Task带着FLAG_ACTIVITY_RESET_TASK_IF_NEEDED标记进入前台时（典型的操作是用户在主画面重启它），这个Activity和它之上的都将关闭，以至于用户不能再返回到它们，但是可以回到之前的Activity。      这在你的程序有分割点的时候很有用。例如，一个e-mail应用程序可能有一个操作是查看一个附件，需要启动图片浏览Activity来显示。这个Activity应该作为e-mail应用程序Task的一部分，因为这是用户在这个Task中触发的操作。然而，当用户离开这个Task，然后从主画面选择e-mail app，我们可能希望回到查看的会话中，但不是查看图片附件，因为这让人困惑。通过在启动图片浏览时设定这个标志，浏览及其它启动的Activity在下次用户返回到mail程序时都将全部清除
+ FLAG_ACTIVITY_NEW_DOCUMENT :
+ FLAG_ACTIVITY_NO_USER_ACTION :　设置这一flag后，可以在避免用户离开当前Activity时回调到 onUserLeaveHint(). 通常，Activity可以通过这个回调表明有明确的用户行为将当前activity切出前台。 这个回调标记了activity生命周期中的一个恰当的点，可以用来“在用户看过通知之后”将它们清除，如闪烁LED灯，　如果Activity是由非用户驱动的事件（如电话呼入或闹钟响铃）启动的，那这个标志就应该被传入Context.startActivity，以确保被打断的activity不会认为用户已经看过了通知
+ FLAG_ACTIVITY_REORDER_TO_FRONT :　设置这一flag后，　如果目标activity已经在运行，那么这个activity将被调到activity stack的顶部, 比如，一个任务中有4个Activity：A，B，C，D。如果D调用了startActivity() 来启动B时使用了这个标志，那B就会被调到历史栈的栈顶，结果顺序：A，C，D，B，否则顺序会是：A，B，C，D，B
   + 如果使用了标志 FLAG_ACTIVITY_CLEAR_TOP，那这个FLAG_ACTIVITY_REORDER_TO_FRONT标志会被忽略
+ FLAG_ACTIVITY_NO_ANIMATION : 设置这一flag后，　启动目标activity时，将禁用系统默认的activity切换动画
+ FLAG_ACTIVITY_CLEAR_TASK : 设置这一flag后，　含有目标activity的task会在activity被启动前清空，　就是说，这个Activity会成为一个新的root，并且所有旧的activity都被finish掉
   + 这个标志只能与FLAG_ACTIVITY_NEW_TASK 一起使用
+ FLAG_ACTIVITY_TASK_ON_HOME : 设置这一flag之后, 目标activity将会被置于当前的home任务(home activity task)之上（如果有的话）。也就是说，在任务中按back键总是会回到home界面，而不是回到他们之前看到的activity
   + 个标志只能与FLAG_ACTIVITY_NEW_TASK标志一起用
+ FLAG_ACTIVITY_RETAIN_IN_RECENTS : 设置该标记后，　当目标activity所在的task被finish后，　在 recent task　中的项将会被继续保留

broadcast相关的flag
    
+ FLAG_RECEIVER_REGISTERED_ONLY : 设置该flag后, 在 AndroidManifest.xml　中定义的receiver将不能接收到该广播，只有在代码中注册的receiver才能收到该广播
+ FLAG_RECEIVER_REPLACE_PENDING : 设置该flag后, 在发送广播时, 如果还存在相同的(Intent.filterEquals()返回true)，　且未被处理的广播，　则会将这些广播替换掉，　由于粘性广播只关注于把最新的值发给他们的接收者，所以粘性广播一般使用这个标记
+ FLAG_RECEIVER_FOREGROUND : 设置该flag后，当发送广播时，允许其接受者 在前台运行的拥有更高的优先级，更短的超时间隔
+ FLAG_RECEIVER_NO_ABORT : 设置该flag后, 如果是有序广播，　则不要允许接收者中断广播播
+ FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT :设置该flag后, 那么在boot completed 之前，　只有在代码中注册的Receiver才能接收到该广播
+ FLAG_RECEIVER_BOOT_UPGRADE :　 如果这条广播是用于系统升级的，设置这个标志，一种特许的状态允许在system ready前发送广播
        
其它的flag：

+ FLAG_FROM_BACKGROUND : 这个flag表示这个Intent是从后台服务发起的, 
+ FLAG_DEBUG_LOG_RESOLUTION : 用来调试，当设置这个flag的时候，在解析这个intent的时候，将会打出打印信息
+ FLAG_EXCLUDE_STOPPED_PACKAGES : 用来控制Intent是否要对处于停止状态的App起作用, 当设置这一flag的时候，表示包含未启动的App
+ FLAG_EXCLUDE_STOPPED_PACKAGES : 用来控制Intent是否要对处于停止状态的App起作用, 当设置这一flag的时候，表示不包含未启动的App
    
###12. IntentFilter

使用显式Intent时，　Intent　中指定了component, 因此，　系统可以准确查找到目标组件，　但是，　对于隐式的　Intent，　其中只包含了抽象的信息，　Android 系统需要根据这些抽象信息来匹配出 component

Android 中使用　IntentFilter 来过滤隐式 Intent, **IntentFilter中具有和Intent对应的用于过滤Action，Data和Category的字段，一个隐式Intent要想被一个组件处理，必须通过这三个环节的检查**,如果任何一方面不匹配，Android都不会将该隐式Intent传递给 目标组件

IntentFilter　的匹配顺序为: action ->  data ->  category

####12.1 声明 IntentFilter

在 AndroidManifest.xml　中的组件标签中，　可以使用 &lt;intent-filter&gt;来声明　IntentFilter

    <activity android:name="ApnEditor"
        android:label="@string/apn_edit"
        android:theme="@style/apn_editor_theme">
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <action android:name="android.intent.action.EDIT" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="vnd.android.cursor.item/telephony-carrier" />
        </intent-filter>

        <intent-filter>
            <action android:name="android.intent.action.INSERT" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="vnd.android.cursor.dir/telephony-carrier" />
        </intent-filter>
    </activity>

    <receiver android:name="CalendarProviderBroadcastReceiver"
        android:exported="false">
        <intent-filter>
            <action android:name="com.android.providers.calendar.intent.CalendarProvider2"/>
            <category android:name="com.android.providers.calendar"/>
        </intent-filter>
        <intent-filter>
            <action android:name="android.intent.action.EVENT_REMINDER"/>
            <data android:scheme="content" />
        </intent-filter>
    </receiver>

    <service android:name="DeviceAdminServiceForEAS">
        <intent-filter>
            <action android:name="android.app.action.DEVICE_ADMIN_SERVICE" />
        </intent-filter>
    </service>

    <provider
        android:name=".DownloadStorageProvider"
        android:authorities="com.android.providers.downloads.documents"
        android:grantUriPermissions="true"
        android:exported="true"
        android:permission="android.permission.MANAGE_DOCUMENTS">
        <intent-filter>
            <action android:name="android.content.action.DOCUMENTS_PROVIDER" />
        </intent-filter>
    </provider>
    
另外，　用于过滤广播的 IntentFilter　还可以在代码中创建，　例如:

    final IntentFilter filter = new IntentFilter(ALARM_SNOOZE_ACTION);
    filter.addAction(ALARM_DISMISS_ACTION);
    registerReceiver(mActionsReceiver, filter);

####12.2 IntentFilter 的 action　检查

尽管一个Intent只可以设置一个Action，但一个IntentFilter可以持有一个或多个Action用于过滤, 到达的Intent只需要匹配其中一个Action即可

如果一个IntentFilter没有设置Action的值，那么，它将不会匹配到任何Intent

如果一个Intent对象没有设置Action值，那么它能通过任何IntentFilter的Action检查

####12.3 IntentFilter 的　data　检查

IntentFilter中的Data部分也可以是一个或者多个，而且可以没有(只有action是必须的), 每个Data包含的内容为URL和数据类型，进行Data检查时主要也是对这两点进行比较，比较规则如下:

1. 如果一个Intent对象没有设置Data，只有Intentfilter也没有设置Data时才可通过检查
2. 如果一个Intent对象包含URI，但不包含数据类型，　则仅当Intentfilter也不指定数据类型，同时它们的URI匹配，才能通过检测
3. 如果一个Intent对象包含数据类型，但不包含URI，则仅当Intentfilter也没指定URL，而只包含数据类型且与Intent相同，才通过检测
4. 如果一个Intent对象既包含URI，也包含数据类型（或数据类型能够从URI推断出），只有当其数据类型匹配Intentfilter中的数据类型，并且通过了URL检查时，该Intent对象才能通过检查

data的URL由４个部分组成, 它有四个属性scheme、host、port、path对应于URI的每个部分, 例如:

    content://com.android.example1:121/files
    
其中:

+ scheme 部分: content
   + 在　AndroidManifest.xml　中的　<data>　标签中使用 "android:scheme" 来表示
+ host 部分: com.android.example1
   + 在　AndroidManifest.xml　中的　<data>　标签中使用 "android:host" 来表示
+ port 部分: 121
   + 在　AndroidManifest.xml　中的　<data>　标签中使用 "android:path" 来表示
+ path 部分: files
   +

host和port部分一起构成URI的凭据(authority),如果host没有指定，那port也会被忽略

这四个属性是可选的，但他们之间并不是完全独立的。要让authority有意义，scheme必须要指定。要让path有意思，scheme和authority必须指定。 Intentfilter中的path可以使用通配符来匹配path字段，Intent和Intentfilter都可以用通配符来指定MIME类型

####12.4 IntentFilter 的　category　检查

Intentfilter中可以设置多个Category，Intent中也可以含有多个Category，只有Intent中的所有Category都能匹配到Intentfilter中的Category，Intent才能通过检查, 也就是说，如果Intent中的Category集合是Intentfilter中Category的集合的子集时，Intent才能通过检查

如果Intent中没有设置Category，则它能通过所有Intentfilter的Category检查

####　IntentFilter　priority

当隐式 Intent　用于启动　activity 或者 ordered　broadcast　时，　最终只能有一个　component　来处理 Intent，　那么, 可以在 IntentFilter　中定义优先级，　改变最终的选择顺序

在定义 IntentFilter　时，　可以设置一个优先级，　优先级的取值为　　-1000 ~ 1000，　**默认的priority是 0**

在 AndroidManifest.xml 中使用 "android:priority" 定义 IntentFilter　的priority：

    <intent-filter　android:priority="3">
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>

在 java　code　中，　使用 IntentFilter.setPriority()　来设置 priority

当一个　Intent 依次经过　“action”　> 　“data”　> "category" 检查后，仍匹配到多个　IntentFilter　并且需要检查　priority　时，　规则如下:

+ 若其中所有 IntentFilter　的　priority　>= 0, 则 这些　IntentFilter　的　priority 起作用, 且数值越高，　优先级越高
+ 只要其中某个 IntentFilter　的 priority　< 0 , 则这些　IntentFilter　的　priority　不起作用

线程一样，优先级的设置只是系统考虑的一个方面，但不是绝对，因此系统仍然会把所有为正的Intent Filter过滤出来让用户选择，而当一个设置为负的时候，系统可能认为此activity的Intent Filter有极其苛刻的过滤条件，此activity也只能适用于少数场合，因此系统按照我们设置的优先级来发挥作用

####12.5 查看 packages　定义的　IntentFilter

IntentFilter　在　AndroidManifest.xml　中定义, 若没有对应的　Package　的源码, 则可以在 android 系统中使用 `pm dump <package name>`　命令来查看指定的　package　的　IntenFilter

例如, 如下定义的 IntentFilter

    <activity　xxxxx>
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <action android:name="android.intent.action.EDIT" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="vnd.android.cursor.item/telephony-carrier" />
        </intent-filter>
    </activity>

使用　`pm dump`　命令显示的结果:

    Activity Resolver Table:
        ......
        Full MIME Types:
            ......
            vnd.android.cursor.item/telephony-carrier:
                c4acfe7 com.android.settings/.Settings$ApnEditorActivity filter 3b00277
                    Action: "android.intent.action.VIEW"
                    Action: "android.intent.action.EDIT"
                    Category: "android.intent.category.DEFAULT"
                    Type: "vnd.android.cursor.item/telephony-carrier"
                    AutoVerify=false

如下定义的 IntentFilter

    <activity　xxxxx>
        <intent-filter>
            <action android:name="android.provider.action.DOCUMENT_ROOT_SETTINGS" />
            <category android:name="android.intent.category.DEFAULT" />
            <data
                android:scheme="content"
                android:host="com.android.externalstorage.documents"
                android:mimeType="vnd.android.document/root" />
        </intent-filter>
    </activity>

使用　`pm dump`　命令显示的结果:

    Activity Resolver Table:
        ......
        Base MIME Types:
            ......
            vnd.android.document:
                add1794 com.android.settings/.Settings$PublicVolumeSettingsActivity filter d2adc76
                    Action: "android.provider.action.DOCUMENT_ROOT_SETTINGS"
                    Category: "android.intent.category.DEFAULT"
                    Scheme: "content"
                    Authority: "com.android.externalstorage.documents": -1
                    Type: "vnd.android.document/root"
                    AutoVerify=false

如下定义的 IntentFilter

    <activity　xxxxx>
        <intent-filter android:priority="1">
            <action android:name="android.settings.ACTION_PRINT_SETTINGS" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:scheme="printjob" android:pathPattern="*" />
        </intent-filter>
    </activity>

使用　`pm dump`　命令显示的结果:

    Activity Resolver Table:
        ......
        Schemes:
            ......
            printjob:
                dca523d com.android.settings/.Settings$PrintJobSettingsActivity filter 18d044e
                    Action: "android.settings.ACTION_PRINT_SETTINGS"
                    Category: "android.intent.category.DEFAULT"
                    Scheme: "printjob"
                    Path: "PatternMatcher{GLOB: *}"
                    mPriority=1, mHasPartialTypes=false
                    AutoVerify=false
                    
如下定义的 IntentFilter

    <activity　xxxxx>
        <intent-filter android:priority="1">
            <action android:name="android.net.wifi.PICK_WIFI_NETWORK" />
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </activity>
    
使用　`pm dump`　命令显示的结果:

    Activity Resolver Table:
        ......
        Non-Data Actions:
            ......
            android.net.wifi.PICK_WIFI_NETWORK:
                37e8bf5 com.android.settings/.wifi.WifiPickerActivity filter 9ffaff
                    Action: "android.net.wifi.PICK_WIFI_NETWORK"
                    Category: "android.intent.category.DEFAULT"
                    mPriority=1, mHasPartialTypes=false
                    AutoVerify=false

上述的示例列出的是用于启动 Activity　的Intent, 对于 braodcast receiver, service, provider 这些组件中的　IntentFilter，　｀pm dump <package name>｀　同样也会列出来

###13. PackageManagerService.resolveIntent()

当Intent用于启动　Activity　时, PackageManagerService.resolveIntent() 用于选择合适的 Activity, 其中主要的工作如下:

+ 调用 queryIntentActivities()　获取所有能够匹配到　Intent　的　activity　的　ResolveInfo
   + queryIntentActivities()会利用 mResolvePrioritySorte　按 priority　**降序排列**所有匹配到　Intent　的　Activity　的　ResolveInfo
+ 调用 chooseBestActivity()　从匹配到　Intent　的　Activity　中选择一个最终的 Activity　(即使有多个Activity匹配到了 Activity　最终也只会启动一个 Activity)
   1. 若上一步返回　单个　ResoleveInfo,　则最启动该　ResolveInfo　对应的　Activity 
   2. 若上一步返回　多个　ResoleveInfo,　对应的　IntentFilter　priority　各不相同，　并且　priority　起作用，　则最启动 IntentFilter　priority 最大的　Activity 
   3. 若上一步返回　多个　ResoleveInfo,  且这些　ResoleveInfo　中至少头部２个的 priority　相等, 则需查看系统中是否存在相关的偏好设置，　若有偏好设置，　则优先选择偏好设置中指定的 Activity, 这一部分调用　findPreferredActivity()　完成
   4. 若上一步返回　多个　ResoleveInfo,  且这些　ResoleveInfo　中至少头部２个的 priority　相等, 且系统中不存在相关的偏好设置则要跳出提示框，　由用户来决定最终需要启动的　Activity
   
上述第４步的 activity　选择器，　默认是 ResolverActivity，　其相关信息保存在　PackageManagerService.mResolveInfo 中, 当然, PackageManagerService　也提供了一个　`setUpCustomResolverActivity()`, 用于使用其它的　acticity　选择器来替换系统默认的　activity　选择器
   
###14. PackageManagerService.resolveService()

当Intent用于绑定 Service　时，　PackageManagerService.resolveService()　用于选择合适的　Service

PackageManagerService.resolveService()　中会调用　queryIntentServices()，　获取到所有匹配到的　Service　的　ResolveInfo (按照 IntentFilter priority 降序排列), 然后选择排在最前面的 ResoleveInfo 对应的　service

###15. 查看系统中的 Intent filter

若是需要检查系统中系统中有那些 package　会处理特定的　Intent，　可以使用如下的命令列出系统中所有注册的 filter　然后进行搜索

    $ adb shell dumpsys package -f resolvers

###16. 附1 - 使用Intent启动常见的系统组件
    
**打开网页**

    Intent intent = new Intent(Intent.ACTION_VIEW);
    intent.setData(Uri.parse("http://www.baidu.com"));                
    startActivity(intent);  

**拨打电话**

    //调出拨号界面
    Intent intent = new Intent(Intent.ACTION_DIAL);
    intent.setData(Uri.parse("tel:10086"));
    startActivity(intent);
    
    //直接拨出电话
    Intent intent = new Intent(Intent.ACTION_CALL);
    intent.setData(Uri.parse("tel:10086"));
    startActivity(intent);  

**发送短信**

    //打开发送短信的界面
    Intent intent = new Intent(Intent.ACTION_VIEW);
    intent.setType("vnd.android-dir/mms-sms");
    intent.putExtra("sms_body", "具体短信内容"); //"sms_body"为固定内容
    startActivity(intent);

    //打开发送短信的界面并指定号码
    Intent intent = new Intent(Intent.ACTION_SENDTO);
    intent.setData(Uri.parse("smsto:18780260012"));
    intent.putExtra("sms_body", "具体短信内容"); //"sms_body"为固定内容        
    startActivity(intent);

**播放指定路径音乐**

    Intent intent = new Intent(Intent.ACTION_VIEW);
    Uri uri = Uri.parse("file:///storage/sdcard0/abcd.mp3"); ////路径也可以写成："/storage/sdcard0/abcd.mp3"
    intent.setDataAndType(uri, "audio/mp3");
    startActivity(intent);

**卸载程序**

    Intent intent = new Intent(Intent.ACTION_DELETE);
    Uri data = Uri.parse("package:com.example.test");
    intent.setData(data);
    startActivity(intent);

**安装程序**
    
    Intent intent = new Intent(Intent.ACTION_VIEW);
    Uri data = Uri.fromFile(new File("/storage/sdcard0/AndroidTest/test.apk"));    //路径不能写成："file:///storage/sdcard0/···"
    intent.setDataAndType(data, "application/vnd.android.package-archive");  //Type的字符串为固定内容
    startActivity(intent);
 
###17. android　扩展名和 MIME　对应关系

        add("application/andrew-inset", "ez");
        add("application/dsptype", "tsp");
        add("application/epub+zip", "epub");
        add("application/hta", "hta");
        add("application/mac-binhex40", "hqx");
        add("application/mathematica", "nb");
        add("application/msaccess", "mdb");
        add("application/oda", "oda");
        add("application/ogg", "ogg");
        add("application/ogg", "oga");
        add("application/pdf", "pdf");
        add("application/pgp-keys", "key");
        add("application/pgp-signature", "pgp");
        add("application/pics-rules", "prf");
        add("application/pkix-cert", "cer");
        add("application/rar", "rar");
        add("application/rdf+xml", "rdf");
        add("application/rss+xml", "rss");
        add("application/zip", "zip");
        add("application/vnd.android.package-archive", "apk");
        add("application/vnd.cinderella", "cdy");
        add("application/vnd.ms-pki.stl", "stl");
        add("application/vnd.oasis.opendocument.database", "odb");
        add("application/vnd.oasis.opendocument.formula", "odf");
        add("application/vnd.oasis.opendocument.graphics", "odg");
        add("application/vnd.oasis.opendocument.graphics-template", "otg");
        add("application/vnd.oasis.opendocument.image", "odi");
        add("application/vnd.oasis.opendocument.presentation", "odp");
        add("application/vnd.oasis.opendocument.presentation-template", "otp");
        add("application/vnd.oasis.opendocument.spreadsheet", "ods");
        add("application/vnd.oasis.opendocument.spreadsheet-template", "ots");
        add("application/vnd.oasis.opendocument.text", "odt");
        add("application/vnd.oasis.opendocument.text-master", "odm");
        add("application/vnd.oasis.opendocument.text-template", "ott");
        add("application/vnd.oasis.opendocument.text-web", "oth");
        add("application/vnd.google-earth.kml+xml", "kml");
        add("application/vnd.google-earth.kmz", "kmz");
        add("application/msword", "doc");
        add("application/msword", "dot");
        add("application/vnd.openxmlformats-officedocument.wordprocessingml.document", "docx");
        add("application/vnd.openxmlformats-officedocument.wordprocessingml.template", "dotx");
        add("application/vnd.ms-excel", "xls");
        add("application/vnd.ms-excel", "xlt");
        add("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", "xlsx");
        add("application/vnd.openxmlformats-officedocument.spreadsheetml.template", "xltx");
        add("application/vnd.ms-powerpoint", "ppt");
        add("application/vnd.ms-powerpoint", "pot");
        add("application/vnd.ms-powerpoint", "pps");
        add("application/vnd.openxmlformats-officedocument.presentationml.presentation", "pptx");
        add("application/vnd.openxmlformats-officedocument.presentationml.template", "potx");
        add("application/vnd.openxmlformats-officedocument.presentationml.slideshow", "ppsx");
        add("application/vnd.rim.cod", "cod");
        add("application/vnd.smaf", "mmf");
        add("application/vnd.stardivision.calc", "sdc");
        add("application/vnd.stardivision.draw", "sda");
        add("application/vnd.stardivision.impress", "sdd");
        add("application/vnd.stardivision.impress", "sdp");
        add("application/vnd.stardivision.math", "smf");
        add("application/vnd.stardivision.writer", "sdw");
        add("application/vnd.stardivision.writer", "vor");
        add("application/vnd.stardivision.writer-global", "sgl");
        add("application/vnd.sun.xml.calc", "sxc");
        add("application/vnd.sun.xml.calc.template", "stc");
        add("application/vnd.sun.xml.draw", "sxd");
        add("application/vnd.sun.xml.draw.template", "std");
        add("application/vnd.sun.xml.impress", "sxi");
        add("application/vnd.sun.xml.impress.template", "sti");
        add("application/vnd.sun.xml.math", "sxm");
        add("application/vnd.sun.xml.writer", "sxw");
        add("application/vnd.sun.xml.writer.global", "sxg");
        add("application/vnd.sun.xml.writer.template", "stw");
        add("application/vnd.visio", "vsd");
        add("application/x-abiword", "abw");
        add("application/x-apple-diskimage", "dmg");
        add("application/x-bcpio", "bcpio");
        add("application/x-bittorrent", "torrent");
        add("application/x-cdf", "cdf");
        add("application/x-cdlink", "vcd");
        add("application/x-chess-pgn", "pgn");
        add("application/x-cpio", "cpio");
        add("application/x-debian-package", "deb");
        add("application/x-debian-package", "udeb");
        add("application/x-director", "dcr");
        add("application/x-director", "dir");
        add("application/x-director", "dxr");
        add("application/x-dms", "dms");
        add("application/x-doom", "wad");
        add("application/x-dvi", "dvi");
        add("application/x-font", "pfa");
        add("application/x-font", "pfb");
        add("application/x-font", "gsf");
        add("application/x-font", "pcf");
        add("application/x-font", "pcf.Z");
        add("application/x-freemind", "mm");
        // application/futuresplash isn't IANA, so application/x-futuresplash should come first.
        add("application/x-futuresplash", "spl");
        add("application/futuresplash", "spl");
        add("application/x-gnumeric", "gnumeric");
        add("application/x-go-sgf", "sgf");
        add("application/x-graphing-calculator", "gcf");
        add("application/x-gtar", "tgz");
        add("application/x-gtar", "gtar");
        add("application/x-gtar", "taz");
        add("application/x-hdf", "hdf");
        add("application/x-hwp", "hwp"); // http://b/18788282.
        add("application/x-ica", "ica");
        add("application/x-internet-signup", "ins");
        add("application/x-internet-signup", "isp");
        add("application/x-iphone", "iii");
        add("application/x-iso9660-image", "iso");
        add("application/x-jmol", "jmz");
        add("application/x-kchart", "chrt");
        add("application/x-killustrator", "kil");
        add("application/x-koan", "skp");
        add("application/x-koan", "skd");
        add("application/x-koan", "skt");
        add("application/x-kpresenter", "kpr");
        add("application/x-kpresenter", "kpt");
        add("application/x-kspread", "ksp");
        add("application/x-kword", "kwd");
        add("application/x-kword", "kwt");
        add("application/x-latex", "latex");
        add("application/x-lha", "lha");
        add("application/x-lzh", "lzh");
        add("application/x-lzx", "lzx");
        add("application/x-maker", "frm");
        add("application/x-maker", "maker");
        add("application/x-maker", "frame");
        add("application/x-maker", "fb");
        add("application/x-maker", "book");
        add("application/x-maker", "fbdoc");
        add("application/x-mif", "mif");
        add("application/x-ms-wmd", "wmd");
        add("application/x-ms-wmz", "wmz");
        add("application/x-msi", "msi");
        add("application/x-ns-proxy-autoconfig", "pac");
        add("application/x-nwc", "nwc");
        add("application/x-object", "o");
        add("application/x-oz-application", "oza");
        add("application/x-pem-file", "pem");
        add("application/x-pkcs12", "p12");
        add("application/x-pkcs12", "pfx");
        add("application/x-pkcs7-certreqresp", "p7r");
        add("application/x-pkcs7-crl", "crl");
        add("application/x-quicktimeplayer", "qtl");
        add("application/x-shar", "shar");
        add("application/x-shockwave-flash", "swf");
        add("application/x-stuffit", "sit");
        add("application/x-sv4cpio", "sv4cpio");
        add("application/x-sv4crc", "sv4crc");
        add("application/x-tar", "tar");
        add("application/x-texinfo", "texinfo");
        add("application/x-texinfo", "texi");
        add("application/x-troff", "t");
        add("application/x-troff", "roff");
        add("application/x-troff-man", "man");
        add("application/x-ustar", "ustar");
        add("application/x-wais-source", "src");
        add("application/x-wingz", "wz");
        add("application/x-webarchive", "webarchive");
        add("application/x-webarchive-xml", "webarchivexml");
        add("application/x-x509-ca-cert", "crt");
        add("application/x-x509-user-cert", "crt");
        add("application/x-x509-server-cert", "crt");
        add("application/x-xcf", "xcf");
        add("application/x-xfig", "fig");
        add("application/xhtml+xml", "xhtml");
        add("audio/3gpp", "3gpp");
        add("audio/aac", "aac");
        add("audio/aac-adts", "aac");
        add("audio/amr", "amr");
        add("audio/amr-wb", "awb");
        add("audio/basic", "snd");
        add("audio/flac", "flac");
        add("application/x-flac", "flac");
        add("audio/imelody", "imy");
        add("audio/midi", "mid");
        add("audio/midi", "midi");
        add("audio/midi", "ota");
        add("audio/midi", "kar");
        add("audio/midi", "rtttl");
        add("audio/midi", "xmf");
        add("audio/mobile-xmf", "mxmf");
        // add ".mp3" first so it will be the default for guessExtensionFromMimeType
        add("audio/mpeg", "mp3");
        add("audio/mpeg", "mpga");
        add("audio/mpeg", "mpega");
        add("audio/mpeg", "mp2");
        add("audio/mpeg", "m4a");
        add("audio/mpegurl", "m3u");
        add("audio/prs.sid", "sid");
        add("audio/x-aiff", "aif");
        add("audio/x-aiff", "aiff");
        add("audio/x-aiff", "aifc");
        add("audio/x-gsm", "gsm");
        add("audio/x-matroska", "mka");
        add("audio/x-mpegurl", "m3u");
        add("audio/x-ms-wma", "wma");
        add("audio/x-ms-wax", "wax");
        add("audio/x-pn-realaudio", "ra");
        add("audio/x-pn-realaudio", "rm");
        add("audio/x-pn-realaudio", "ram");
        add("audio/x-realaudio", "ra");
        add("audio/x-scpls", "pls");
        add("audio/x-sd2", "sd2");
        add("audio/x-wav", "wav");
        // image/bmp isn't IANA, so image/x-ms-bmp should come first.
        add("image/x-ms-bmp", "bmp");
        add("image/bmp", "bmp");
        add("image/gif", "gif");
        // image/ico isn't IANA, so image/x-icon should come first.
        add("image/x-icon", "ico");
        add("image/ico", "cur");
        add("image/ico", "ico");
        add("image/ief", "ief");
        // add ".jpg" first so it will be the default for guessExtensionFromMimeType
        add("image/jpeg", "jpg");
        add("image/jpeg", "jpeg");
        add("image/jpeg", "jpe");
        add("image/pcx", "pcx");
        add("image/png", "png");
        add("image/svg+xml", "svg");
        add("image/svg+xml", "svgz");
        add("image/tiff", "tiff");
        add("image/tiff", "tif");
        add("image/vnd.djvu", "djvu");
        add("image/vnd.djvu", "djv");
        add("image/vnd.wap.wbmp", "wbmp");
        add("image/webp", "webp");
        add("image/x-cmu-raster", "ras");
        add("image/x-coreldraw", "cdr");
        add("image/x-coreldrawpattern", "pat");
        add("image/x-coreldrawtemplate", "cdt");
        add("image/x-corelphotopaint", "cpt");
        add("image/x-jg", "art");
        add("image/x-jng", "jng");
        add("image/x-photoshop", "psd");
        add("image/x-portable-anymap", "pnm");
        add("image/x-portable-bitmap", "pbm");
        add("image/x-portable-graymap", "pgm");
        add("image/x-portable-pixmap", "ppm");
        add("image/x-rgb", "rgb");
        add("image/x-xbitmap", "xbm");
        add("image/x-xpixmap", "xpm");
        add("image/x-xwindowdump", "xwd");
        add("model/iges", "igs");
        add("model/iges", "iges");
        add("model/mesh", "msh");
        add("model/mesh", "mesh");
        add("model/mesh", "silo");
        add("text/calendar", "ics");
        add("text/calendar", "icz");
        add("text/comma-separated-values", "csv");
        add("text/css", "css");
        add("text/html", "htm");
        add("text/html", "html");
        add("text/h323", "323");
        add("text/iuls", "uls");
        add("text/mathml", "mml");
        // add ".txt" first so it will be the default for guessExtensionFromMimeType
        add("text/plain", "txt");
        add("text/plain", "asc");
        add("text/plain", "text");
        add("text/plain", "diff");
        add("text/plain", "po");     // reserve "pot" for vnd.ms-powerpoint
        add("text/richtext", "rtx");
        add("text/rtf", "rtf");
        add("text/text", "phps");
        add("text/tab-separated-values", "tsv");
        add("text/xml", "xml");
        add("text/x-bibtex", "bib");
        add("text/x-boo", "boo");
        add("text/x-c++hdr", "hpp");
        add("text/x-c++hdr", "h++");
        add("text/x-c++hdr", "hxx");
        add("text/x-c++hdr", "hh");
        add("text/x-c++src", "cpp");
        add("text/x-c++src", "c++");
        add("text/x-c++src", "cc");
        add("text/x-c++src", "cxx");
        add("text/x-chdr", "h");
        add("text/x-component", "htc");
        add("text/x-csh", "csh");
        add("text/x-csrc", "c");
        add("text/x-dsrc", "d");
        add("text/x-haskell", "hs");
        add("text/x-java", "java");
        add("text/x-literate-haskell", "lhs");
        add("text/x-moc", "moc");
        add("text/x-pascal", "p");
        add("text/x-pascal", "pas");
        add("text/x-pcs-gcd", "gcd");
        add("text/x-setext", "etx");
        add("text/x-tcl", "tcl");
        add("text/x-tex", "tex");
        add("text/x-tex", "ltx");
        add("text/x-tex", "sty");
        add("text/x-tex", "cls");
        add("text/x-vcalendar", "vcs");
        add("text/x-vcard", "vcf");
        add("video/3gpp", "3gpp");
        add("video/3gpp", "3gp");
        add("video/3gpp2", "3gpp2");
        add("video/3gpp2", "3g2");
        add("video/avi", "avi");
        add("video/dl", "dl");
        add("video/dv", "dif");
        add("video/dv", "dv");
        add("video/fli", "fli");
        add("video/m4v", "m4v");
        add("video/mp2ts", "ts");
        add("video/mpeg", "mpeg");
        add("video/mpeg", "mpg");
        add("video/mpeg", "mpe");
        add("video/mp4", "mp4");
        add("video/mpeg", "VOB");
        add("video/quicktime", "qt");
        add("video/quicktime", "mov");
        add("video/vnd.mpegurl", "mxu");
        add("video/webm", "webm");
        add("video/x-la-asf", "lsf");
        add("video/x-la-asf", "lsx");
        add("video/x-matroska", "mkv");
        add("video/x-mng", "mng");
        add("video/x-ms-asf", "asf");
        add("video/x-ms-asf", "asx");
        add("video/x-ms-wm", "wm");
        add("video/x-ms-wmv", "wmv");
        add("video/x-ms-wmx", "wmx");
        add("video/x-ms-wvx", "wvx");
        add("video/x-sgi-movie", "movie");
        add("video/x-webex", "wrf");
        add("x-conference/x-cooltalk", "ice");
        add("x-epoc/x-sisx-app", "sisx");