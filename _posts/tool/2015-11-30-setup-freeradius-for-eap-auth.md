---
layout: post
title: "使用freeradius搭建EAP认证环境"
description:
category: tool
tags: [tool, network, linux]
mathjax: 
chart:
comments: false
---

###1. freeradius

RADIUS：Remote Authentication Dial In User Service，远程用户拨号认证系统由RFC2865，RFC2866定义，是目前应用最广泛的AAA协议。AAA是一种管理框架，因此，它可以用多种协议来实现。在实践中，人们最常使用远程访问拨号用户服务（Remote Authentication Dial In User Service，RADIUS）来实现AAA

RADIUS是一种C/S结构的协议，它的客户端最初就是NAS（Net Access Server）服务器，任何运行RADIUS客户端软件的计算机都可以成为RADIUS的客户端。RADIUS协议认证机制灵活，可以采用PAP、CHAP或者Unix登录认证等多种方式。RADIUS是一种可扩展的协议，它进行的全部工作都是基于Attribute-Length-Value的向量进行的。RADIUS也支持厂商扩充厂家专有属性。

由于RADIUS协议简单明确，可扩充，因此得到了广泛应用，包括普通电话上网、ADSL上网、小区宽带上网、IP电话、VPDN（Virtual Private Dialup Networks，基于拨号用户的虚拟专用拨号网业务）、移动电话预付费等业务。IEEE提出了802.1x标准，这是一种基于端口的标准，用于对无线网络的接入认证，在认证时也采用RADIUS协议

freeradius 是一种开源的radius服务器的实现

###2. 安装freeradius

首先， 安装依赖的软件包

    $ sudo apt-get install openssl libssl-dev

在ubuntu上可以使用 apt-get 来安装freeradius

	$ sudo apt-get install freeradius

安装完成后， 可以看到系统中安装了如下的软件包

	$ sudo dpkg -l | grep radiu

正常情况下应该显示安装了如下的软件包:

+ freeradius
+ freeradius-common
+ freeradius-utils
+ libfreeradius2

如果缺少其中的文件， 可以使用 apt-get 单独安装

安装freeradius成功后， 执行如下的命令来测试

    $ sudo service freeradius stop
    $ sudo freeradius -x
    
如果看到有如下的输出：

    ......
     ... adding new socket proxy address * port 50963
    Listening on authentication address * port 1812
    Listening on accounting address * port 1813
    Listening on authentication address 127.0.0.1 port 18120 as server inner-tunnel
    Listening on proxy address * port 1814
    Ready to process requests.

则说明freeradius启动正常

###3. freeradius-utils中的测试工具

freeradius-utils 软件包包含一系列的测试工具用于测试不同的认证方式

+ radclient	    //用于向radius server发送radius packet并接收回复
+ raddebug      //用于显示radius server的debug信息
+ radeapclient	//用于向radius server发送EAP packet
+ radlast		//从wtmp 文件中获取最后的accounting log并显示
+ radmin		//freeradius demo的管理工具
+ radsniff      //wrapp libpcap， 用于dump radius协议消息
+ radsqlrelay	//传递SQL查询到一个集中的数据库服务器 
+ radtest		//radclient的前端， 用于向radius server发送访问请求并接收回复
+ radwatch      //用于启动和监控radius server， 当radius server退出时， 负责重启
+ radwho		//显示当前在线的user
+ radzap		//radwho从数据库或者文件中获取当前在线的user的信息， 有时候这些信息没有同步， 
                //radzap 用于同步这些信息

###4. 安装eapol_test
 
在测试eap认证时， 还需要安装 eapol_test 工具， eapol_test 是 wpa_supplicant 中的一个测试程式，在[wpa_supplicant 官网](http://w1.fi/wpa_supplicant/) 可下载wpa_supplicant 源码，自行编译,例如

	$ wget http://w1.fi/releases/wpa_supplicant-2.5.tar.gz

wpa_supplicant 默认使用netlink， 因此， 需要先安装netlink的lib库，以使用的ubuntu 15.04为例

    $ sudo apt-get install libnl-3-200
    $ sudo apt-get install libnl-3-dev
    $ sudo apt-get install libnl-genl-3-200
    $ sudo apt-get install libnl-genl-3-dev
    
然后， 进入wpa_supplicant 的源码目录进行编译

	$ tar xzvf wpa_supplicant-2.5.tar.gz
	$ cd wpa_supplicant-2.5/wpa_supplicant
	$ cp defconfigf .config
	$ vim .config
	
	//去掉 “#CONFIG_EAPOL_TEST=y” 前的注释
	//去掉 "#CONFIG_LIBNL32=y" 前的注释

	$ make eapol_test

然后安装编译出的 eapol_test

    $ sudo cp eapol_test /usr/local/bin
    $ sudo chmod a+x /usr/local/bin/eapol_test
    
###5. freeradius的配置

先看最为常见的radius认证的模型

![radius认证模型](/images/tool/radius-certificate-model.png)

其中路由器扮演了NAS的角色

freeradius的配置文件为存放在 "/etc/freeradius/" 目录下的 *.conf， 不同的配置文件有着不同的作用， 其中常用的有

+ clients.conf : 对radius client进行配置
+ eap.conf : 对eap认证方式进行配置
+ radiusd.conf : 对radius server进行配置
+ sql.conf ： freeradius可以使用文件和数据库来保存user信息， 当使用数据库来保存user信息时， sql.conf用于配置数据库
+ users : freeradius可以使用文件和数据库来保存user信息， 当使用文件来保存user信息时， 在此文件中保存

###5.1 radius.conf

radius.conf 中的内容基本上保持默认值就可以使用， 需要注意如下几个部分

    listen{
        type = auth     #可取值有 auth, acct, proxy, detail, status, coa
        ipaddr = *      #可以是域名，或者ip， 或者*代表通配 
        port = 0        #端口号， 0代表从 /etc/services 中读取 (默认为1812)
    }

freeradius支持 认证/计费 等功能， 我们只关注认证功能， 因此只关注 auth 类型的 listen块即可

ipaddr和port用于指定freeradius绑定的IP地址和端口号， 有些系统上面支持在一个网络接口上设置多个ip， 此时需要详细指定 ipaddr 和 port， 一般情况下不需要修改
    
    modules {
            ......
            #$INCLUDE eap.conf
            #$INCLUDE sql.conf
            ......
    }
    
当需要使用eap认证的功能时， 则需要去掉 “#$INCLUDE eap.conf” 前的注释， 当需要使用数据库来存储user信息时， 则需要去掉 “#$INCLUDE sql.conf” 前的注释

后续的测试将会只使用文件的方式来存储user信息

###5.2 clients.conf

client.conf 文件中， 使用 client 块来配置一个 radius cleint

    client Name {       //Name 为client的名称， 自行定义
        ipaddr =        //client 的ip地址 或者主机名
        #ipv6addr =     //ipv6 地址
        
        #netmask =      //子网掩码， 和ipaddr 不在一个网段的ip的packet讲被忽略, 默认为32
        secret =        //client的密码
        
        require_message_authenticator =     //老式的client在Access-Request中不包含Message-Authenticator， 而 RFC 5080 强制要求包含这一项
                                            //这一配置项定义client是否包含Message-Authenticator， 如果定义为yes， 但是实际上没有包含， 则请求会被忽略
        nastype =       //NAS type, 普通的路由设置为 other 即可
    }

client.conf中预定义了一个client “localhost” 用于测试

    client localhost {
        ipaddr = 127.0.0.1
        secret          = testing123
        require_message_authenticator = no
        nastype     = other
    }


####5.3 users

users文件用于添加user， 例如

    testuser Cleartext-Password := "testpswd"
    
即添加了一个名为 testuser 的user， 其密码为 testpswd

还可以在users文件中指定某一个user的认证结果， 例如

    lameuser        Auth-Type := Reject
                    Reply-Message = "Your account has been disabled."
                    
即针对user “lameuser”， 直接拒绝其认证请求

###6. 生成EAP认证所需的证书

EAP认证最多需要3份证书(EAP-TLS)：

+ CA 证书
+ server 端证书
+ client 端证书

在linux上可以使用 openssl 命令来生成这些证书， 详细内容可以参考“tools”分类下的“openssl 证书／签名／加密”一文, 可以使用“make-certs.sh来协助生成所需的证书”

    ＃file make-certs.sh
    
    #!/bin/sh

    rsa_bits=2048
    pswd_in=123456
    pswd_out=123456

    dn_C=CN
    dn_ST=JS
    dn_L=SZ
    dn_O=radius
    dn_OU=freeradius
    dn_EA="sven@xxx.com"
    dn_CN_ca=ca
    dn_CN_server=server
    dn_CN_client=client

    rm demoCA -rf
    mkdir -p demoCA/newcerts/
    touch demoCA/index.txt
    echo 01 > demoCA/serial

    #generate self signed CA cert
    openssl req -new -x509 -passout pass:$pswd_out -subj /C=$dn_C/ST=$dn_ST/L=$dn_L/O=$dn_O/OU=$dn_OU/CN=$dn_CN_ca/emailAddress=$dn_EA -keyout ca.key -out ca.pem 

    #generate server and client cert request
    openssl req -new -passout pass:$pswd_out -subj /C=$dn_C/ST=$dn_ST/L=$dn_L/O=$dn_O/OU=$dn_OU/CN=$dn_CN_server/emailAddress=$dn_EA -keyout server.key -out server.csr 
    openssl req -new -passout pass:$pswd_out -subj /C=$dn_C/ST=$dn_ST/L=$dn_L/O=$dn_O/OU=$dn_OU/CN=$dn_CN_client/emailAddress=$dn_EA -keyout client.key -out client.csr

    #sign server and client request whith CA cert
    openssl ca -in server.csr -out server.pem -cert ca.pem -keyfile ca.key -passin pass:$pswd_in
    openssl ca -in client.csr -out client.pem -cert ca.pem -keyfile ca.key -passin pass:$pswd_in

运行过程中， 需要确认是否对server端证书和client端证书进行签名， 输入“y”即可

运行完成后， 生成的证书如下：

+ ca.key   : CA证书的公私钥对
+ ca.pem   : CA证书， PEM格式
+ server.key ： server端证书的公私钥对
+ server.pem ： server端证书， PEM格式
+ client.key ： client端证书的公私钥对
+ client.pem ： erver端证书， PEM格式

在 “/etc/freeradius/certs/”下新建“test”目录， 然后讲上述的 ca.pem, server.pem, server.key 拷贝到该目录中， 然后修改eap.conf 中的 tls 块：

    - certdir = ${confdir}/certs
    + certdir = ${confdir}/certs/test
    
    - cadir = ${confdir}/certs
    + cadir = ${confdir}/certs/test

    - private_key_password = whatever
    + private_key_password = 123456

**frreradius中 PEAP 和 TTLS 模块都依赖于 TLS，　因此只要使用证书认证时，　就需要配置号TLS**

###7. 本地测试环境

在 "/etc/freeradius/users"　中添加一个测试用户 testuser， 密码为 userpswd

    testuser Cleartext-Password := "userpswd"
    
在notebook上同时运行　radius　server　和　测试程序:

+ client为　localhost　，　密码为　testing123
+ user 为　testuser，　密码为　userpswd

####7.1 测试添加的 testuser

这一步目的是测试上一步中添加的 testuser 可用, 通过radtest命令， 使用预置的 client "localhost" 向radius server发送radius packet来进行测试, radtes的usage如下

    $ radtest [OPTIONS] user passwd radius-server[:port] nas-port-number secret
    
其中， nas-port-number 没有用到时， 置0即可

首先， 在notebook上以调试模式启动freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
然后新开一个终端， 执行测试程序

    $ radtest testuser userpswd 127.0.0.1 0 testing123

返回如下的内容

    rad_recv: Access-Accept packet from host 127.0.0.1 port 1812　......

说明认证请求被通过

###8. 本地测试EAP-MD5

测试EAP-MD5前， 请确保 /etc/freeradius/radiusd.conf 中的 modules 块中的 "$INCLUDE eap.conf" 语句未被注释掉

首先， 修改 eap.conf , 指定默认的eap 方式为md5

    eap {
        ......
        default_eap_type = md5
        ......
    }
    
测试eap-md5 需要使用 radeapclient 命令， 其usage为

    radeapclient [options] server[:port] <command> [<secret>]
    
    # command 的取值有 auth, acct, status, disconnect
    
radeapclient会进入交互模式， 等待输入， 为了方便测试， 可以将内容保存到一个文件中， 编写一个 eap-md5.txt 文件， 内容如下：

    User-Name="testuser"
    ClearText-Password="userpswd"
    EAP-Code=Response
    EAP-Id=210
    EAP-Type-Identity="testuser"
    Message-Authenticatior=0x00
    
在一个终端中以调试模式打开 freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
在另一个终端中执行

    $ radeapclient -x 127.0.0.1 auth testing123 < eap-md5.txt
    
成功后， 可看到输出内容如下

    ......
    
    Received Access-Accept packet from host 127.0.0.1 port 1812 ......
        ......
        EAP-Code = Success
    
###9. 测试EAP-TLS

测试EAP-peap前， 请确保 /etc/freeradius/radiusd.conf 中的 modules 块中的 "$INCLUDE eap.conf" 语句未被注释掉

首先， 修改 eap.conf , 指定默认的eap 方式为tls

    eap {
        ......
        default_eap_type = tls
        ......
    }

测试eap-peap 需要使用 eapol_test 命令, 还需要编写一个config file “eap-tls.conf”

    network={
	   eap=TLS
	   eapol_flags=0
	   key_mgmt=IEEE8021X
	   identity="testuser"
	   password="userpswd"

	   ca_cert="/etc/freeradius/certs/test/ca.pem"
	   client_cert="/etc/freeradius/certs/test/client.pem"
	   private_key="/etc/freeradius/certs/test/client.key"
	   private_key_passwd="123456"
    }
    
在一个终端中以调试模式打开 freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
在另一个终端中执行eapol_test

    $ eapol_test -c eap-tls.conf -a 127.0.0.1 -p 1812 -s testing123 -r 1
    
若最后返回SUCCESS则说明认证成功

###10. 测试 EAP-PEAP

测试EAP-peap前， 请确保 /etc/freeradius/radiusd.conf 中的 modules 块中的 "$INCLUDE eap.conf" 语句未被注释掉

首先， 修改 eap.conf , 指定默认的eap 方式为md5

    eap {
        ......
        default_eap_type = peap
        ......
    }
    
测试eap-peap 需要使用 eapol_test 命令, 还需要编写一个config file “eap-peap.conf”

    network {
        ssid="example"
        key_mgmt=WPA_EAP
        eap=PEAP
        identity="testuser"
        password="userpswd"
        anonymous_identity="anonymouse"
        phase2="autheap=MSCHAPV2"
        
        #ca_cert="/etc/freeradius/certs/test/ca.pem"
    }
    
在一个终端中以调试模式打开 freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
在另一个终端中执行eapol_test

    $ eapol_test -c eap-peap.conf -s testing123
    
若最后返回SUCCESS则说明认证成功

另外还需要注意的是：

+ EAP-PEAP认证可以选择在client端使用根证书验证server端证书， 只需要取消 “ca_cert="/etc/freeradius/certs/test/ca.pem” 前的注释即可

###11. 测试 EAP-TTLS

首先， 修改 eap.conf , 指定默认的eap 方式为ttls

    eap {
        ......
        default_eap_type = ttls
        ......
    }

####11.1 EAP-TTLS 之 PAP

编写一个config file “eap-ttls-pap.conf”

    network={
	   ssid="TP-LINK_19AB"
	   key_mgmt=WPA-EAP
	   eap=TTLS
	   identity="testuser"
	   anonymous_identity="anonymous"
	   password="userpswd"
	   phase2="auth=PAP"

	   #ca_cert="/etc/freeradius/certs/test/ca.pem"
    }

若最后返回SUCCESS则说明认证成功

EAP-TTLS认证可以选择在client端使用根证书验证server端证书， 只需要取消 “ca_cert="/etc/freeradius/certs/test/ca.pem” 前的注释即可


####11.2 EAP_TTLS 之 CHAP

编写一个config file “eap-ttls-chap.conf”

    network={
	   ssid="TP-LINK_19AB"
	   key_mgmt=WPA-EAP
	   eap=TTLS
	   identity="testuser"
	   anonymous_identity="anonymous"
	   password="userpswd"
	   phase2="auth=CHAP"

	   #ca_cert="/etc/freeradius/certs/test/ca.pem"
    }

若最后返回SUCCESS则说明认证成功

EAP-TTLS认证可以选择在client端使用根证书验证server端证书， 只需要取消 “ca_cert="/etc/freeradius/certs/test/ca.pem” 前的注释即可

####11.3 EAP_TTLS 之 MSCHAPv2

编写一个config file “eap-ttls-mschapv2.conf”

    network={
	   ssid="TP-LINK_19AB"
	   key_mgmt=WPA-EAP
	   eap=TTLS
	   identity="testuser"
	   anonymous_identity="anonymous"
	   password="userpswd"
	   phase2="auth=MSCHAPV2"

	   #ca_cert="/etc/freeradius/certs/test/ca.pem"
    }

若最后返回SUCCESS则说明认证成功

EAP-TTLS认证可以选择在client端使用根证书验证server端证书， 只需要取消 “ca_cert="/etc/freeradius/certs/test/ca.pem” 前的注释即可

####11.4 EAP_TTLS 之 EAP-MD5

编写一个config file “eap-ttls-eap-md5.conf”

    network={
	   ssid="TP-LINK_19AB"
	   key_mgmt=WPA-EAP
	   eap=TTLS
	   identity="testuser"
	   anonymous_identity="anonymous"
	   password="userpswd"
	   phase2="autheap=MD5"

	   #ca_cert="/etc/freeradius/certs/test/ca.pem"
    }

若最后返回SUCCESS则说明认证成功

EAP-TTLS认证可以选择在client端使用根证书验证server端证书， 只需要取消 “ca_cert="/etc/freeradius/certs/test/ca.pem” 前的注释即可

####11.5 EAP-TTLS 之 EAP-MSCHAPv2

编写一个config file “eap-ttls-eap-mschapv2.conf”

    network={
	   ssid="TP-LINK_19AB"
	   key_mgmt=WPA-EAP
	   eap=TTLS
	   identity="testuser"
	   anonymous_identity="anonymous"
	   password="userpswd"
	   phase2="autheap=MSCHAPV2"

	   ca_cert="/etc/freeradius/certs/test/ca.pem"
    }

若最后返回SUCCESS则说明认证成功

EAP-TTLS认证可以选择在client端使用根证书验证server端证书， 只需要取消 “ca_cert="/etc/freeradius/certs/test/ca.pem” 前的注释即可

###12. 使用android手机和无线路由进行EAP认证测试

向 /etc/freeradius/client.conf 中添加一个client testclient (由图中的路由来扮演)

    client testclient{
        ipaddr = 192.168.1.1
        secret = clientpswd
        nastype = other
    }

使用之前添加的“testuser"作为测试user

配置的测试环境如下图

![freeradius 测试环境](/images/tool/freeradius-test-enviroment.png)

+ 在 notebook 上安装freeradius， 作为 radius server， IP 为 192.168.1.10
+ TPL-INK 无线路由器， 作为 radius client， IP 为 192.168.1.1
+ android 手机作为radius user
+ 使用的freeradius client账号为　testclient　密码为　clientpswd
+ 使用的freeradius user账号为　testuser 密码为　userpswd

修改无线路由器的无线安全设定， 设置加密方式为 WPA/WPA2-EAP， 填写radius server的ip(192.168.1.10)，client("testclient"), secret("clientpswd"), 重启后生效

将前述步骤生成的　ca.pem  client.key client.pem　安装到android 手机中

####12.1 测试 TLS

首先， 修改 eap.conf , 指定默认的eap 方式为ttls

    eap {
        ......
        default_eap_type = tls
        ......
    }

在一个终端中以调试模式打开 freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
在android的settings界面中点击AP， 选择EAP-TLS加密方式， 输入"testuser" 和 ”userpswd“， 选择CA证书和client证书， 点击连接

####12.2 测试 PEAP

首先， 修改 eap.conf , 指定默认的eap 方式为peap

    eap {
        ......
        default_eap_type = peap
        ......
    }

在一个终端中以调试模式打开 freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
在android的settings界面中点击AP， 选择EAP-PEAP加密方式，选择所需的phase2认证， 输入"testuser" 和 ”userpswd“， 如果要验证server端， 还需要选择CA证书, 最后点击连接

####12.3 测试 TTLS

首先， 修改 eap.conf , 指定默认的eap 方式为ttls

    eap {
        ......
        default_eap_type = ttls
        ......
    }

在一个终端中以调试模式打开 freeradius

    $ sudo service freeradius stop
    $ sudo freeradius -X
    
在android的settings界面中点击AP， 选择EAP-TTLS加密方式，选择所需的phase2认证， 输入"testuser" 和 ”userpswd“， 如果要验证server端， 还需要选择CA证书, 最后点击连接
