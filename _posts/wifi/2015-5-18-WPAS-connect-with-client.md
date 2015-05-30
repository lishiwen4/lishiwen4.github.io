---
layout: post
title: "WPAS与client的通信"
description:
category: wifi
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. WPAS支持的连接方式  
  
WPAS支持client使用如下几种方式来进行连接： 
  
+ named pipe
+ udp socket
+ unix socket  

在Android和Linux上， wpa_cli通常使用unix socket来连接WPAS  
  
###2. WPAS的控制接口  
  
在socket通信时， server端在定义好的端口上监听， client端发起连接，而WPAS所监听的端口被称作WPAS的控制接口， 在unix socekt address中， 没有端口号， 由一个文件路径来标识地址， 这一控制接口， 因此， 在下文中也将这一文件的路径称作WPAS的控制接口， 用以下方式来指定  
  
1. 在WPAS启动参数中， 使用“-g”选项来指定全局的控制接口  
2. 在WPAS中为每一个无线网络接口单独使用“-C”选项来指定  
3. 在WPAS中使用“-O”来指定override的控制接口， 可以覆盖第2步指定的控制接口对应的文件路径    
  
在Android中, 启动WPAS时， 使用了 “-g” 和 “-O” 两个选项  
 
首先，在WPAS启动参数中使用“-g”指定的全局控制接口， 对应的文件路径为“/dev/sockets/wpa_wlan0”，在WPAS初始化时， 会监听该控制接口  
  
	static int wpas_global_ctrl_iface_open_sock(struct wpa_global *global,struct ctrl_iface_global_priv *priv)
	{
		......
		if (os_strncmp(ctrl, "@android:", 9) == 0) {
			priv->sock = android_get_control_socket(ctrl + 9);
		......
		eloop_register_read_sock(priv->sock, wpa_supplicant_global_ctrl_iface_receive, global, priv);
		......
	}
  
另外在启动WPAS时， 使用“-O”指定的控制接口文件路径为"/data/misc/wifi/sockets"， 这其实是一个目录， 在该目录下会根据无线网络接口的名称， 为每一个无线网络接口生成一个控制接口文件， 例如“/data/misc/wifi/sockets/wlan0”， “/data/misc/wifi/sockets/p2p0“  
	
	static char * wpa_supplicant_ctrl_iface_path(struct wpa_supplicant *wpa_s)
	{
		......
		pbuf = os_strdup(wpa_s->conf->ctrl_interface);
		......
		dir = pbuf;
		......
		res = os_snprintf(buf, len, "%s/%s", dir, wpa_s->ifname);
		......
		return buf;
	}

	static int wpas_ctrl_iface_open_sock(struct wpa_supplicant *wpa_s, struct ctrl_iface_priv *priv)
	{
		......
		fname = wpa_supplicant_ctrl_iface_path(wpa_s);
		......
		os_strlcpy(addr.sun_path, fname, sizeof(addr.sun_path));
		if (bind(priv->sock, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
		......
		eloop_register_read_sock(priv->sock, wpa_supplicant_ctrl_iface_receive, wpa_s, priv);
		......
	}
  
下面，使用netstat来检查一下， Android设备上， WPAS是否在这些控制接口上监听：  
  
1. 关闭wifi， 以及Settings -> advanced -> Scanning always avaliable  
2. reboot 设备  
3. adb shell
4. setprop ctl.start p2p_supplicant  
5. busybox netstat -axp \| grep wpa

输出的结果如下 : 
  
	unix  2      [ ]         DGRAM                    187045 17937/wpa_supplican /dev/socket/wpa_wlan0
	unix  2      [ ]         DGRAM                    185121 17937/wpa_supplican /data/misc/wifi/sockets/wlan0
	unix  2      [ ]         DGRAM                    185127 17937/wpa_supplican /data/misc/wifi/sockets/p2p0  

可以看到WPAS在3个unix socket(及WPAS的控制接口)上监听 ：  
  
1. 全局控制接口 “/dev/socket/wpa_wlan0”
2. wlan0 的控制接口 “/data/misc/wifi/sockets/wlan0”
3. p2p0的控制接口 “data/misc/wifi/sockets/p2p0”

###3. libwpa_client.so 
  
WPAS提供了2种client ： 
  
+ 字符模式下的wpa_cli
+ GUI模式下的wpa_gui

另外， WPAS源码还可以编译出libwpa_client.so 用于开发client  
  
在android上， wifi service 在native 层使用libwpa_client.so提供的功能来直接和WPAS通信，而在开发过程中， 也经常使用wpa_cli来进行调试  
  
libwpa_client.so 的源码为： "wpa_supplicant/src/common/wpa_ctrl.c" 和 "wpa_supplicant/src/utils/os_unix.c", 而wpa_cli和wpa_gui也是静态编译了这些代码，相当于也是使用libwpa_client.so来连接WPAS， libwpa_client.so 提供的接口为：  
  
	struct wpa_ctrl * wpa_ctrl_open(const char *ctrl_path);
	void wpa_ctrl_close(struct wpa_ctrl *ctrl);
	int wpa_ctrl_request(struct wpa_ctrl *ctrl, const char *cmd, size_t cmd_len,
		char *reply, size_t *reply_len, void (*msg_cb)(char *msg, size_t len));
	int wpa_ctrl_attach(struct wpa_ctrl *ctrl);
	int wpa_ctrl_detach(struct wpa_ctrl *ctrl);
	int wpa_ctrl_recv(struct wpa_ctrl *ctrl, char *reply, 
		size_t *reply_len);
	int wpa_ctrl_pending(struct wpa_ctrl *ctrl);
	int wpa_ctrl_get_fd(struct wpa_ctrl *ctrl);
	void wpa_ctrl_cleanup(void);

+ wpa_ctrl_open() ： 连接WPAS的控制接口, 可以是WPAS的全局控制接口，也可以是为每一个无线网络接口指定的控制接口
+ wpa_ctrl_close() : 关闭wpa_ctrl_open()打开的连接  
+ wpa_ctrl_request() : 通过建立的连接向WPAS发送消息  
+ wpa_ctrl_attach() : 使用wpa_ctrl_open()建立的连接， WPAS默认不会向这些连接的client端发送event， 必须显示调用wpa_ctrl_attach()，才能接收到消息  
+ wpa_ctrl_detach() : 取消wpa_ctrl_attach()
+ wpa_ctrl_recv() : 接收WPAS端发来的event， 必须要先在打开的连接上调用wpa_ctrl_attach()才能接收到event， 当无event可读时， 此调用会被block住  
+ wpa_ctrl_pending() : 检查是否有pending的event， 若有则可以调用wpa_ctrl_recv()来接收  
+ wpa_ctrl_get_fd() : 获取同WPAS的连接中， client端的fd， 获取的fd可以用于select， epoll等， 但是不能直接用于收发消息， 必须使用wpa_ctrl_request()和wpa_ctrl_recv（） 
+ wpa_ctrl_cleanup() : 当使用unix socekt 进行连接时，会建立socket文件， 若其carsh， 则会遗留这些文件， wpa_ctrl_cleanup()用于清理这些文件  
 
###4. wpa_ctrl_open()  
  
在wpa_supplicant/Android.mk中有  
  
    L_CFLAGS += -DCONFIG_CTRL_IFACE_CLIENT_DIR=\"/data/misc/wifi/sockets\"

再来看wpa_ctrl_open的源码  

    #ifndef CONFIG_CTRL_IFACE_CLIENT_PREFIX
    #define CONFIG_CTRL_IFACE_CLIENT_PREFIX "wpa_ctrl_"
    #endif /* CONFIG_CTRL_IFACE_CLIENT_PREFIX */
    
    struct wpa_ctrl * wpa_ctrl_open(const char *ctrl_path)
    {
        ......
        ret = os_snprintf(ctrl->local.sun_path, sizeof(ctrl->local.sun_path),
                CONFIG_CTRL_IFACE_CLIENT_DIR "/"
                CONFIG_CTRL_IFACE_CLIENT_PREFIX "%d-%d",
	       (int) getpid(), counter);
        ......
        if (bind(ctrl->s, (struct sockaddr *) &ctrl->local,
		    sizeof(ctrl->local)) < 0) {
        ......
        if (os_strncmp(ctrl_path, "@android:", 9) == 0) {
            if (socket_local_client_connect(
                    ctrl->s, ctrl_path + 9,
                    ANDROID_SOCKET_NAMESPACE_RESERVED,
                    SOCK_DGRAM) < 0) {
        ......
        res = os_strlcpy(ctrl->dest.sun_path, ctrl_path,
            sizeof(ctrl->dest.sun_path));
        ........
        if (connect(ctrl->s, (struct sockaddr *) &ctrl->dest,
            sizeof(ctrl->dest)) < 0) {
        ......
    }

android上， 调用wpa_ctrl_open()时，传递的WPAS的控制接口要么是"@android:wpa_wlan"， 要么是“/data/misc/wifi/sockets/wlan0”这样的控制接口， wpa_ctrl_open()会为client端绑定地址， 并生成对应的文件， 文件路径为“/data/misc/wifi/sockets/wpa_ctrl_xx_yy”， 其中xx为client端的pid， yy为client端的进程的client计数， 从1，2，3依次递增  
  
###5. wifi service连接WPAS  
  
wifi service 通过wifi HAL(链接libwpa_client.so)来同WPAS交互，且使用全局控制接口“/dev/sockets/wpa_wlan0”， 这一工作在“/hardware/libhadware_legacy/wifi/wifi.c”中完成，其会使用“@android:wpa_wlan0”作为参数来调用wpa_ctrl_open()， 在settings中打开wifi后通过netstat可以查看到：  

	unix  4      [ ]         DGRAM                     40347 8005/wpa_supplicant /dev/socket/wpa_wlan0
	unix  2      [ ]         DGRAM                     43667 8005/wpa_supplicant /data/misc/wifi/sockets/wlan0
	unix  2      [ ]         DGRAM                     43675 8005/wpa_supplicant /data/misc/wifi/sockets/p2p-dev-wlan0
	unix  2      [ ]         DGRAM                     43679 561/system_server   /data/misc/wifi/sockets/wpa_ctrl_561-5
	unix  2      [ ]         DGRAM                     43680 561/system_server   /data/misc/wifi/sockets/wpa_ctrl_561-6

wifi service运行在system_server进程中， “/dev/socket/wpa_wlan0“的refcount为4， 说明有两个client端的连接， 而系统中的client只有system_server， 在wifi HAL中会打开两个连接， 一个称为ctrl， 用于向WPAS发送cmd并获取结果， 一个称为monitor， 用于监听WPAS发送出来的event  
  
需要注意的是， 在使用全局控制接口来连接WPAS时， 一些sta相关的cmd(例如scan， scan_result)需要指定interface，在wifiNative的代码中可以看到   
	
	public WifiNative(String interfaceName) {
	    ......
        	    if (!interfaceName.equals("p2p0")) {
                 mInterfacePrefix = "IFNAME=" + interfaceName + " ";
        	    } else {
                 // commands for p2p0 interface don't need prefix
                 mInterfacePrefix = "";
        	    }
    	}

	private boolean doBooleanCommand(String command) {
	    ......
             boolean result = doBooleanCommandNative(mInterfacePrefix + command);
                 ......
                 return result;
        	    }
    	}
 
在使用全局控制接口来对STA类型的无线网络接口执行cmd时， 需要加“IFNAME=xxx”前缀来指明无线网络接口(有些cmd不许要指定， 但是指定了也不会有影响)，而对p2p类型的无线网络接口则不需要指明

###6. wpa_cli连接WPAS  
  
当使用wpa_cli连接WPAS的时候， 指定控制接口的时候， 有两种选择：  
  
+ 使用无线网络接口的特定控制接口 : 例如， “/data/misc/wifi/sockets/wlan0”, "/data/misc/wifi/sockets/p2p0", 然后在wpa_cli中直接执行命令
+ 使用全局的控制接口 : 例如， “/dev/sockets/wlan0”, "/dev/sockets/p2p0", 然后在wpa_cli中执行命令， 有些命令需要添加“IFNAME=xxx”前缀

cmd如下：

	wpa_cli -p /data/misc/wisi/socket/ -i wlan0  
	wpa_cli -g @android:wpa_wlan0  
  
###7. wpas处理cmd  
  
+ 在使用wpa_cli连接到WPAS时，若进入wpa_cli的shell中执行cmd， 则由wpa_cli_action()来调用wpa_ctrl_request()来请求WPAS处理cmd， 若不进入wpa_cli的shell， 只执行一条cmd， 则直接在wpa_cli的main()中调用wpa_ctrl_request()来请求WPAS处理cmd  
+ android上的wifi HAL会直接调用wpa_ctrl_request()来请求WPAS处理cmd  
  
因此，先来看wpa_ctrl_request()的实现  
  
    int wpa_ctrl_request(struct wpa_ctrl *ctrl, 
            const char *cmd, size_t cmd_len,
            char *reply, size_t *reply_len,
            void (*msg_cb)(char *msg, size_t len))
    {
        ......
        if (send(ctrl->s, _cmd, _cmd_len, 0) < 0) {
        ......
        res = select(ctrl->s + 1, &rfds, NULL, NULL, &tv);
        ......
        res = recv(ctrl->s, reply, *reply_len, 0);
        ......
    }

依次是 send(), select(), recv() 三步， client将cmd发送给WPAS处理， 然后读取wpas的返回， 接下来看WPAS对cmd的处理：  
  
+ WPAS的全局控制接口的read event由wpa_supplicant_global_ctrl_iface_receive()处理
+ WPAS的无线网络特定的接口的特定控制接口的read event由wpa_supplicant_ctrl_iface_receive()处理  
  
事实上， wpa_supplicant_global_ctrl_iface_receive() 和 wpa_supplicant_ctrl_iface_receive() 只会处理两条简单的cmd ：  
  
+ ATTACH ： 即wpa_ctrl_attach()的实现， WPAS会将相应的client端地址加入到某个链表中(WPAS在发送EVENT时， 会遍历该链表，给每一个地址发送一次)  
+ DETACH ： 即wpa_ctrl_detach()的实现， WPAS会将相应的client端地址移除出某个链表中(WPAS在发送EVENT时， 会遍历该链表，给每一个地址发送一次)  
  
其它的cmd分别由wpa_supplicant_global_ctrl_iface_process()和wpa_supplicant_ctrl_iface_process()来处理  
  
前面我们提到过全局控制接口和各无线网络接口的区别， 即在使用全局控制接口时， 有一些cmd前面需要添加“IFNAME=xxx”前缀来指定对哪一个无线网络接口来执行命令， wpa_supplicant_global_ctrl_iface_receive()在处理完“IFNAME=xxx”之后， 余下的处理工作还是调用wpa_supplicant_ctrl_iface_process()来完成， 事实上， 所有的cmd都可由wpa_supplicant_ctrl_iface_process()来完成， 下面， 只需要来关注wpa_supplicant_ctrl_iface_process()即可  
  
在wpa_supplicant_ctrl_iface_process()中会匹配传递进来的cmd， 依次调用相应的函数来处理， 因cmd太多， 这里不再举例， 请自行查看  
  
###END

