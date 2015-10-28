---
layout: post
title: "android wifi信号强度显示"
description:
category: android
tags: [android, framework, wifi]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. RSSI

wifi的信号强度使用RSSI表示， 在Android的Framework中会中会转换为对应的格数在状态栏来显示

RSSI(Received Signal Strength Indication) 代表接收的信号强度指示, 它与模块的发送功率以及天线的增益有关。单位为dBm， 发射端的发出的微波发射后通过空气传播衰减强烈， 在接收端接收到的功率必然会下降， 这时后的接收功率就是RSSI

信号的绝对功率的表示方式为：

	p(dBW) = 10 * lg ( xW / 1W)
	p(dBm) = 10 * lg ( x * 1000mW / 1mW)

可得：

	1W = 30dBm = 0dBW
	1mW = 0dBm

**因为无线信号多为mW级别(空中传输的过程中衰减较大)，所以对它进行了极化，转化为dBm而已，不表示信号是负的。1mW就是0dBm，小于1mW转换为dBm后就为负数**

RSSI的获取 ： 在104us内进行基带IQ功率积分得到RSSI的瞬时值，即RSSI（瞬时）=sum（I^2+Q^2）；然后在约1秒内对8192个RSSI的瞬时值进行平均得到RSSI的平均值，即RSSI（平均）=sum（RSSI(瞬时）)/8192，同时给出1秒内RSSI瞬时值的最大值和RSSI瞬时值大于某一门限时的比率（RSSI瞬时值大于某一门限的个数/8192）。由于 RSSI是通过在数字域进行功率积分而后反推到天线口得到的，反向通道信号传输特性的不一致会影响RSSI的精度

###2. 从wpa_supplicant获取RSSI

1. 对于当前连接的AP， 可以使用wpa_cli的‘signal_poll’命令来获取

	$ wpa_cli -p /data/misc/wifi/sockets/ -i wlan0 signal_poll                         <
	RSSI=-60
	LINKSPEED=72
	NOISE=9999
	FREQUENCY=2412

从输出结果中可以看到当前AP的 rssi， linkspeed， frequency

android framework 在fetchRssiLinkSpeedAndFrequencyNative()中通过WifiNative.signalPoll()来执行wpa_cli的“signal_poll”来获取当前的信号强度

wifiservice在初始化时， 会调用enableRssiPolling()向WifiStateMachine发送“CMD_ENABLE_RSSI_POLL”来打开 rssi poll， WifiStateMachine在处理该消息时调用fetchRssiLinkSpeedAndFrequencyNative()获取一次当前AP的信号强度，然后发送“CMD_RSSI_POLL”消息， 而WifiStateMachine在处理该消息时调用fetchRssiLinkSpeedAndFrequencyNative()获取一次当前AP的信号强度， 然后再延时3s发送“CMD_RSSI_POLL”消息， 此后， 开始循环获取当前AP的信号强度

2. 对于未连接的AP， 可以使用wpa_cli的“bss”命令从扫描结果中获取信息, 例如

	$ wpa_cli -p /data/misc/wifi/sockets/ -i wlan0 BSS RANGE=0- MASK=0x21987
	id=365
	bssid=10:c3:7b:d2:df:68
	freq=2412
	level=-59
	tsf=0000270980059560
	flags=[WPA2-PSK-CCMP][WPS][ESS]
	ssid=TDC-647-US

里面的level即代表rssi， 若将“bss”命令的mask置为0x0， 可以获得所有的信息， 但是android framework中只关注其中一部分， 因此设置为0x21987

android framework 中在每次wpa_supplicant有新的扫描结果时， 通过 WifiNative.scanResults()来执行wpa_cli的“bss”命令， 获取扫描的结果

###3. framework层对wifi信号强度的处理

在WifiInfo中定义的无效rssi阈值，以及最大rssi和最小rssi如下

	public static final int INVALID_RSSI = -127;
	public static final int MIN_RSSI = -126;
	public static final int MAX_RSSI = 200;
	
在fetchRssiLinkSpeedAndFrequencyNative()中， 通过WifiNative.signalPoll()获取当前AP的信号强度之后， 处理如下

1. 对于小于INVALID_RSSI和大于MAX_RSSI的值， 都直接修改为INVALID_RSSI， 
2. 对于大于0的RSSI， 减去256， 因为有些厂商的实现为了避免负值， 会将其加上256， 再上报给wpa_supplicant
3. 调用WifiManager.calculateSignalLevel()计算wifi信号的等级(即wifi信号的格数)
4. 若wifi信号的等级发生了变化， 则调用sendRssiChangeBroadcast()发送广播(”android.net.wifi.RSSI_CHANGED“)进行通知

再来看wifi信号等级的计算过程， WifiManager中定义了5个wifi信号强度的等级(0格到4格)
	
	public static final int RSSI_LEVELS = 5;
	private static final int MIN_RSSI = -100;
	private static final int MAX_RSSI = -55;
	

	public static int calculateSignalLevel(int rssi, int numLevels) {
        		if (rssi <= MIN_RSSI) {
            		return 0;
        		} else if (rssi >= MAX_RSSI) {
            		return numLevels - 1;
        		} else {
            		float inputRange = (MAX_RSSI - MIN_RSSI);
            		float outputRange = (numLevels - 1);
            		return (int)((float)(rssi - MIN_RSSI) * outputRange / inputRange);
        		}
    	}

1. RSSI小于等于-100的为0格
2. RSSI大于-55的为4格
3. 按照RSSI分为(-100, -88), [-88, -78), [-78, -67), [-67, -55), [-55, 0) 5 个等级

再结合wifiinfo中定义的RSSI限值， wifi RSSI的等级划分如下：

+ [-126, -88) 或者 [156, 168) 为 0 格
+ [-88, -78)  或者 [168, 178) 为 1 格
+ [-78, -67)  或者 [178, 189) 为 2 格
+ [-67, -55)  或者 [189, 200) 为 3 格
+ [-55, 0]    或者            为 4 格 

一般 0到-50表示信号最好，-50到-70表示信号偏差，小于-70表示最差，有可能连接不上或者掉线，一般Wifi已断则值为-200

**framwork中在为statusbar计算wifi信号格树时， 调用WifiManager.calculateSignalLevel()传递的第二个参数为5(即信号划分为5个等级0～4)， 而Settings在调用WifiManager.calculateSignalLevel()传递的第二个参数为4((即信号划分为5个等级0～3))**
###4. StatusBar 更新wifi信号强度

WifiStateMachine中在wifi信号等级变化后， 会发送“android.net.wifi.RSSI_CHANGED”并附带当前的rssi值， 而StatusBar中则会接收并处理这一广播

在StatusBar 的源码 ”frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/NetworkControllerImpl.java“ 中

	public NetworkControllerImpl(Context context) {
		......
		IntentFilter filter = new IntentFilter();
        		filter.addAction(WifiManager.RSSI_CHANGED_ACTION);
		......
		context.registerReceiver(this, filter);
		......
	}

	public void onReceive(Context context, Intent intent) {
		......
		if (action.equals(WifiManager.RSSI_CHANGED_ACTION)
                		|| action.equals(WifiManager.WIFI_STATE_CHANGED_ACTION)
                		|| action.equals(WifiManager.NETWORK_STATE_CHANGED_ACTION)) {
            	updateWifiState(intent);
            	refreshViews();
		......
	}

	protected void updateWifiState(Intent intent) {
		......
		} else if (action.equals(WifiManager.RSSI_CHANGED_ACTION)) {
            		mWifiRssi = intent.getIntExtra(WifiManager.EXTRA_NEW_RSSI, -200);
            		mWifiLevel = WifiManager.calculateSignalLevel(mWifiRssi, WifiIcons.WIFI_LEVEL_COUNT);
        		}

        		updateWifiIcons();
	}

	protected void updateWifiIcons() {
		int inetCondition = inetConditionForNetwork(ConnectivityManager.TYPE_WIFI);
		if (mWifiConnected) {
            		mWifiIconId = WifiIcons.WIFI_SIGNAL_STRENGTH[inetCondition][mWifiLevel];
            		mQSWifiIconId = WifiIcons.QS_WIFI_SIGNAL_STRENGTH[inetCondition][mWifiLevel];
            		mContentDescriptionWifi = mContext.getString(
                    	AccessibilityContentDescriptions.WIFI_CONNECTION_STRENGTH[mWifiLevel]);
        		} else {
		......
	}
	
###5. Settings中的wifi列表更新wifi信号强度

首先， 在settings中打开wifi列表时， WifiSettings内部会实例化一个内部类 Scanner

	private static class Scanner extends Handler {
		private int mRetry = 0;
        		private WifiSettings mWifiSettings = null;

        		Scanner(WifiSettings wifiSettings) {
            		mWifiSettings = wifiSettings;
        		}

        		void resume() {
            		if (!hasMessages(0)) {
                			sendEmptyMessage(0);
            		}
        		}

        		void forceScan() {
            		removeMessages(0);
            		sendEmptyMessage(0);
        		}

        		void pause() {
            		mRetry = 0;
            		removeMessages(0);
        		}

        		@Override
        		public void handleMessage(Message message) {
            		if (mWifiSettings.mWifiManager.startScan()) {
                			mRetry = 0;
            		} else if (++mRetry >= 3) {
                			mRetry = 0;
                			Activity activity = mWifiSettings.getActivity();
                			if (activity != null) {
                    			Toast.makeText(activity, R.string.wifi_fail_to_scan, Toast.LENGTH_LONG).show();
                			}
                			return;
            		}
            		sendEmptyMessageDelayed(0, WIFI_RESCAN_INTERVAL_MS);
        		}
    	}


可以看到， Scanner在被触发后， 会间隔WIFI_RESCAN_INTERVAL_MS(一般为10秒)发起一次扫描

在有扫描结果时， 系统会发出 WifiManager.SCAN_RESULTS_AVAILABLE_ACTION 广播， 而WifiSetting会监听此广播， 并且在WifiSettings.handleEvent()中会调用updateAccessPoints()来更新ui界面

在wifi打开的情况下， 中会调用updateAccessPoints()会调用constructAccessPoints()， 首先编译所有已经保存的网络，构造对应的AccessPoint， 以ssid作为键， 添加到一个HashMap apMap中去，  然后再遍历所有的扫描结果，添加到apMap中去或者更新apMap中的结果(包括ssid), 最后，带哦用refersh() 更新wifi信号强度的图片

**Settings中wifi列表的信号强度刷新主要依靠 WifiManager.SCAN_RESULTS_AVAILABLE_ACTION 广播来触发， 但是如下广播也会触发信号强度的刷新：**

+ WifiManager.CONFIGURED_NETWORKS_CHANGED_ACTION
+ WifiManager.LINK_CONFIGURATION_CHANGED_ACTION
+ WifiManager.NETWORK_STATE_CHANGED_ACTION
+ WifiManager.RSSI_CHANGED_ACTION 
