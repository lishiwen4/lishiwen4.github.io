---
layout: post
title: "openssl 证书/签名/加密"
description:
category: tool
tags: [tool, network, linux]
mathjax: 
chart:
comments: false
---

###1. openssl

OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用[openssl 项目网站](http://www.openssl.org/docs/manmaster/apps/req.html)
  
###2. 证书和加密的相关知识

####2.1 加密算法都有哪些

加密算法主要分为对称加密和非对称加密:

+ 对称加密 : 指双方拥有相同的密钥，用这个密钥加密的也可以用这个密钥解密。常见的有DES，DES3，RC4等
+ 非对称加密 : 指双方使用不同的密钥，分为公钥和私钥，公钥是发给别人，用于加密的，私钥是留给自己用于把别人加密的东西解密的。公钥和私钥是严格成对的。常见的算法有RSA，DSA，DH等

除了这些加密算法一般还会提到叫散列算法，这些算法不是用来加密的，而是把数据转换成一些特定长度的较验数据，用来检验原数据是否被篡改的。常见的有MD5，SHA，Base64，CRC等

####2.2 CA  

CA(certification authorit) 的缩写，即证书颂发机构，他是一个负责发放和维护证书的实体

CA的组织结构跟域名有些相似，有一个根CA，然后它派生了一些子的CA, 形成一个链，根证书是CA认证中心给自己颁发的证书,是信任链的起始点, 安装根证书意味着对这个CA认证中心的信任　可以交钱使用第３方的根证书, 也可以使用自签名的根证书

一个CA中心主要是做以下的工作：
　1)接受证书申请，验证申请人身份，签发证书，向用户提供证书的下载
　2)吊销证书，发布黑名单，发布吊销列表(CRL,certificate revocation list) 
  
**通常也把根证书称为CA证书**
  
数字证书则是由CA对证书申请者真实身份验证之后，用CA的根证书对申请人的一些基本信息以及申请人的公钥进行签名（相当于加盖发证书机构的公章）后形成的一个数字文件。数字证书包含证书中所标识的实体的公钥（就是说你的证书里有你的公钥），由于证书将公钥与特定的个人匹配，并且该证书的真实性由颁发机构保证（就是说可以让大家相信你的证书是真的），因此，数字证书为如何找到用户的公钥并知道它是否有效这一问题提供了解决方案  

####2.3 证书的作用

证书主要有两个作用，一个是加密通信，一个是数字签名:

+ 加密通信 : 保证数据不被别人截获并且不被知道通信内容的，主要是两个层次
   + 一个是通信双方身份确认，避免对方是冒充的
   + 另一个是数据通过公钥加密传输和使用私钥解密。这方面常见的具体应用就是SSL和HTTPS，比如说WebSphere的管理员登陆和一些重要的登陆操作。
+ 数字签名 : 用于识别签名者身份的，这个从字面就可以理解。你使用你的私钥进行签名，然后用户看到你的签名后用公钥检查，发现的确是你的数字签名，就可以了。这个常见的应用有代码发行商签名，就是签名控件的那种，邮件数字签名，电子公章等。 

####2.4 常见的证书协议

+ x509v3 : IETF的证书标准
+ x.500 : 目录的标准
+ SCEP : 简单证书申请协议，用http来进行申请，数据有PKCS#7封装，数据其实格式也是PKCS#10的
+ PKCS#7 : 是封装数据的标准，可以放置证书和一些请求信息
+ PKCS#10 : 用于离线证书申请的证书申请的数据格式，注意数据包是使用PKCS#7封装这个数据
+ PKCS#12 : 用于一个单一文件中交换公共和私有对象，就是公钥，私钥和证书，这些信息进行打包，加密放在存储目录中，CISCO放在NVRAM中，用户可以导出，以防证书服务器挂掉可以进行相应恢复。思科是.p12,微软是.pfx

X.509被广泛使用的数字证书标准，是由国际电联电信委员会（ITU-T）为单点登录（SSO-Single Sing-on）和授权管理基础设施（PMI-Privilege Management Infrastructure）制定的PKI(Public Key Infrastructure)标准, X.509定义了（但不仅限于）公钥证书、证书吊销清单、属性证书和证书路径验证算法等证书标准

x509　证书有２种存储格式：

+ DER 编码(ASCII)  : 后缀是 ".der" ".cer" ".crt"
   + 这一类文件只包含证书，　不包含私钥
+ PEM 编码(Base64) : 后缀是 ".pem" ".cer" ".crt"
   + 通常使用后缀　".der", “.cer”, "crt"时，　文件只包含证书， 不包含私钥, 使用“.pem”后缀时，文件包含证书和私钥中的一种或者多种

还有一些其它类型的后缀:

+ .key ： 私钥文件, 也可以使用“.pem”　后缀
+ .csr : 证书签名请求（证书请求文件），含有公钥信息，certificate signing request的缩写
+ .crl : 证书吊销列表，Certificate Revocation List的缩写 

###3. openssl生成证书的步骤  
 
**生成根证书的步骤:**  
  
1. 生成CA私钥  
2. 生成CA证书请求  
3. 自签名得到根证书  
  
**生成证书(服务端/客户端):**  
  
1. 生成私钥  
2. 生成证书请求  
3. 通过CA签得到证书  
  
###4. 示例  
  
首先， 需要在生成证书的目录建立以下的目录和文件, 这些文件和目录在使用CA证书签名其它的证书时或被使用到
  
	$ mkdir -p demoCA/newcerts
    $ touch demoCA/index.txt
    $ touch demoCA/seria
    $ echo 01 > demoCA/serial
  
demoCA/seria 中的数值在生成证书时会依次递增  
  
####4.1 生成自签名x509格式的CA证书  

使用 “openssl req” 命令来生成 ca.crt 和 ca.key （可使用 `openssl req -h`来获取帮助信息 ）

    $ openssl req -new -x509 -keyout ca.key -out ca.crt
    
首先， 会生成ca.key, 这一步会要求输入密码(最少4个字符)， 后续使用ca.key时都要输入这一密码

    writing new private key to 'ca.key'
    Enter PEM pass phrase:
    Verifying - Enter PEM pass phrase:

继续, 输入完密码后，　还会要求输入一系列信息:  
  
    Country Name (2 letter code) [AU]: CN
	State or Province Name (full name) [Some-State]:JiangSu
	Locality Name (eg, city) []: SuZhou
	Organization Name (eg, company) [Internet Widgits Pty Ltd]: test
	Organizational Unit Name (eg, section) []: wifi
	Common Name (e.g. server FQDN or YOUR name) []:ca 
	Email Address []: sven@xxx.com 

最后，　生成的　ca.crt　为根证书，　ca.key　为密钥，　**需要注意的是，　rsa为非对称加密，　生成的rsa密钥ca.key中同时包含私钥和公钥**
	
对于上述过程中输入的DN字段的解释如下:

+ Country Name 	            : 缩写为“C” 	证书持有者所在国家 	要求填写国家代码，用2个字母表示
+ State or Province Name 	: 缩写为“ST“ 证书持有者所在州或省份 	填写全称，可省略不填
+ Locality Name             : 缩写为“L” 	证书持有者所在城市 	可省略不填
+ Organization Name         : 缩写为“O“ 	证书持有者所属组织或公司 	最好还是填一下
+ Organizational Unit Name  : 缩写为“OU” 证书持有者所属部门 	可省略不填
+ Common Name               : 缩写为“CN“ 证书持有者的通用名 	必填。
   + 对于非应用证书，它应该在一定程度上具有惟一性；
   + 对于应用证书，一般填写服务器域名或通配符样式的域名。
+ Email Address             : 证书持有者的通信邮箱 	可省略不填

#####4.1.1 上述步骤的改进

上述步骤需要进入交互模式，　等待输入密码和DN信息, 可以使用　“-passout” 和　“-subj”　参数在命令行中指定这些参数

    openssl req -new -x509 -passout pass:123456 -subj /CN=CN/ST=JiangSu/L=SuZhou/O=/test/OU=wifi/CN=ca/emailaddress=sven@xxx.com -keyout ca.key -out ca.crt

上述步骤生成的ca.key是1024bits的RSA公私钥对，　如果希望成密钥长度不为1024bits的RSA公私钥对或者其它类型加密(例如　des, des3, aes128, aes192, aes256)的公私钥对，　则要使用"-newkey"参数来代替”-new“参数，　例如:

    $ openssl req -newkey -des3:2048 -x509 -keyout ca.key -out ca.crt
    $ openssl req -newkey -rsa:2048 -x509 -keyout ca.key -out ca.crt

生成的ca.key被加密， 后续使用openssl读写ca.key时都要输入这一密码， 可使用如下的方式来清除ca.key的密码保护

    $ openssl rsa -in ca.key -out ca.key 
    
默认生成的ca.crt使用PEM格式，　可以使用 “-outform” 参数来指定格式，　例如，　指定为DER

    $ openssl req -new -x509 -keyout ca.key　-outform　DER -out ca.crt
        
证书的默认有效期为365天，可以使用”-days“参数来指定有效的天数, 例如

    $ openssl req -new -x509 -keyout ca.key -out ca.crt -days 100

**想使用openssl生成一个只能用于签名的证书，即证书的扩展属性：密钥用途，只能用于数字签名, 但是我们一般使用openssl生成证书时，生成的证书都是v1证书，是不带扩展属性的, v1的CA证书有可能会被视作user证书而不是CA证书, 因此要生成v3的证书，**

    $ openssl req -new -x509 -keyout ca.key -out ca.crt -extensions v3_req

默认使用的config文件是　"/etc/ssl/openssl.cnf"，　”v3_req“　是其中定义的一个扩展块

#####4.1.2 使用config file来生成证书

可以将上述的DN信息保存在一个文件 "ca.cnf"　中来生成ca.crt 和 ca.key, 参照 /etc/ssl/openssl.cnf

    #file ca.cnf
    [req]
    prompt = no
    
    distinguished_name = my_ca
    default_bits = 2048
    input_password = 123456       #根据需要输入密码
    output_password = 123456      #根据需要输入密码
    x509_extensions = ca_extensions

    [my_ca]
    countryName = CN
    stateOrProvinceName = JiangSu
    localityName = SuZhou
    organizationName =test
    organizationalUnitName = wifi
    emailAddress = sven@xxx.com
    commonName = ca
    
    #just needed by self signed CA
    [ca_extensions]
    subjectKeyIdentifier=hash  
    authorityKeyIdentifier=keyid:always,issuer:always  
    basicConstraints = CA:true     
    
然后执行
    
    $ openssl req -new -x509 -keyout ca.key -out ca.crt -config ./ca.cnf
    
注意在生成CA时， config文件中需要有　“ca_extensions” 块的内容，　这是因为，　CA证书必须带有扩展属性(密钥用途，只能用于数字签名), 只有v3证书才有扩展属性，使用默认的config　file "/etc／ssl/openssl.conf"　时，　其内部声明了x509 v3扩展　"v3_ca"，　因此在使用自定义的config生成CA时，　需要自己添加　v3　扩展的内容, 否则生成的是　v1　证书，不能
作为CA去签发下级证书

####4.2 生成密钥    
  
例如， 要为server和client生密钥， 分别以“server”和“client”命名  
  
	$ openssl genrsa -des3 -out server.key 2048 
    $ openssl genrsa -des3 -out client.key 2048 
   
生成的“server.key”和"client.key"即为RSA密钥文件(包含公钥和私钥)， 最后的参数 “2048” 指定了的bit位数, "-des3"　为密钥的加密类型，有如下的加密类型:

+ des
+ des3
+ aes128
+ aes192
+ aes256

如果不希望对密钥进行加密，　可以使用如下命令类生成密钥

    $ openssl genrsa 2048 > server.key 
    $ openssl genrsa 1024 > server.key
  
###4.3 生成证书请求  
  
例如， 要为server和client生成证书请求， 分别以“server”和“client”命名

    $ openssl req -new -key server.key -out server.csr
    $ openssl req -new -key client.key -out client.csr
    
生成的“server.csr”和"client.csr"即为证书请求文件， CSR文件必须有CA的签名才可形成证书

和生成 ca.crt 一样, 生成证书请求时， 也要求输入 country 等DN信息， 同样可以将这些信息存储在文件中
  
###4.4 使用根证书来签名证书请求
  
例如， 要使用根证书为server和client的证书请求签名， 生成证书  
	
    $ openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key 
    $ openssl ca -in client.csr -out client.crt -cert ca.crt -keyfile ca.key
    
生成的 “server.crt” 和 “client.crt”即为x509证书  
在使用根证书来签名生成证书时，注意以下几点：  
  
1. 一个证书请求不能被多次签名， demoCA/index.txt中会有记录  
2. 进行CA签名获取证书时，需要注意国家、省、单位需要与CA证书相同，否则会报：The countryName field needed to be the same in the CA certificate (cn) and the request (sh)
3. 进行CA签名获取证书时，如果信息完全和已有证书信息相同会报错，即不能生成相同的证书(一般保持commonName不同)，报错信息为：failed to update database  TXT_DB error number 2
4. 如出现：unable to access the ./demoCA/newcerts directory 则需要自己建立 demoCA目录或者 修改 /usr/lib/ssl/openssl.cnf 中dir的值  
  
  
**生成server和client端证书的步骤可以简化为:**

    $ openssl req -new -passout pass:123456 -subj /CN=CN/ST=JiangSu/L=SuZhou/O=/test/OU=test.wifi/CN=server/emailaddress=sven@xxx.com -keyout server.key -out server.csr
    
    $ openssl req -new -passout pass:123456 -subj /CN=CN/ST=JiangSu/L=SuZhou/O=/test/OU=test.wifi/CN=client/emailaddress=sven@xxx.com -keyout client.key -out client.csr
    
    $ openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -passin pass:123456
    $ openssl ca -in client.csr -out client.crt -cert ca.crt -keyfile ca.key -passin pass:123456
    
###5. 合并x509证书文件和密钥钥  
  
有时需要分发证书和密钥, 可以使用 `cat` 命令将证书和密钥合并到一个　".pem"　文件中，例如
	
    $ cat client.crt client.key > client.pem 
    $ cat server.crt server.key > server.pem
    
###6. 查看证书和密钥

可以使用openssl　查看x509证书和私钥(对于单独的证书/密钥文件以及同时包含证书和密钥的 ".pem"　文件均能查看)

从 .pem 查看证书  
  
	$ openssl x509 -in client.crt -noout -text
	
从　.pem　查看密钥

    $ openssl rsa -in server.pem -noout -text
    
从　.pem 分离密钥

    $ openssl rsa -in server.pem -out pub.key
    
使用这一方法可以检查文件中是否包含有效的证书或者私钥

###7. 提取公钥

openssl 生成的rsa密钥文件包含公钥和私钥对，　使用如下的方式可以从密钥文件或者包含密钥文件的　.pem　文件中提取公钥

    $ openssl rsa -in ca.key -pubout -out pub.key

###８. 验证证书有效性

openssl verify 命令对证书的有效性进行验证，verify 指令会沿着证书链一直向上验证，直到一个自签名的CA

在生成证书的示例中，　生成了３组证书：　ca.crt, server.crt, client.crt, 使用　ca 证书验证　server证书和client证书的方法如下：

    $ openssl verify -CAfile ca.crt server.crt
    $ openssl verify -CAfile ca.crt client.crt
    
如果返回值为OK, 则说明证书验证成功

示例针对的是只有一层根证书的情况，　若存在多级根证书，　则验证证书时，　需要指定该证书链中的所有证书

###９. 使用密钥签名／加解密

以前述生成的　client.key　为例(其中包含公私钥), 提取其公钥

    $ openssl rsa -in client.key -pubout -out client.pub

####9.1 使用密钥签名／验证签名 小文件

假设在Ａ　Ｂ两端(A拥有公私钥，　B拥有公钥)对一个文件进行签名／验证签名

在Ａ端使用私钥签名(生成的　test-A.sig　为签名文件)test-A.txt

    $ openssl rsautl -sign -inkey client.key -out test-A.sig test-A.txt
    
A将签名文件test-A.sig　发送给Ｂ端
    
B端使用公钥验证签名(输出的内容即为原始文件的内容, 可以使用”-out“参数将其输出到指定的文件中去)
 
    $ openssl rsautl -verify -pubin -inkey client.pub -in test-A.sig -out test-B.txt
    
若签名验证成功，　则会生成test-B.txt，　且其内容和　test-A.txt 一致

当然，　也可以使用私钥验证签名(输出的内容即为原始文件tes-A.txt的内容, 可以使用”-out“参数将其输出到指定的文件中去)

    $ openssl rsautl -verify -inkey client.key -in test-A.sig
    
####9.2 使用密钥签名／验证签名 大文件

签名只适用于较小的文件，　若需要对较大的文件进行签名，　则还要使用印章

印章和消息摘要可以对任意长度的消息产生固定长度（16或20个字节）的信息摘要，理论基于单向HASH函数，根据消息摘要无法恢复出原文，所以是安全的；消息原文和消息摘要是一一对应的，所以又被称作指纹

**对大文件的签名及验证的示例如下：**

在Ａ端生成文件印章(test-A.dgst　即为印章)

    $ openssl dgst -out test-A.dgst test-A.txt

在Ａ端使用私钥对印章签名(生成的　test.sig　为签名文件):

    $ openssl rsautl -sign -inkey client.key -in test-A.dgst -out test-A.sig    

Ａ将原始文件test-A.txt和签名后的印章test-A.sig传递给B

在B端使用公钥验证签名后的印章文件　test-A.sig

    $ openssl rsautl -verify -pubin -inkey client.pub -in test-A.sig -out test-B.dgst
    
若验证正常，　B端会得到印章文件　test-B.dgst，　且其和　test-Ａ.dgst　应当一致

在Ｂ端对test-A.txt生成印章　test.dgst

    $ openssl dgst -out test.dgst test-A.txt
    
在Ｂ端比较　test-A.dgst test.dgst，　若２者一致，　则说明接收到的　test-A.txt　的确来自A端，　且未被篡改

**上述步骤可以使用 openssl dgst　的　”-sign“ 和　”-verify“　来简化**

在A端利用私钥生成签名后的印章文件

    $ openssl dgst -sign client.key -out test-A.dgst.sig test-A.txt

A将签名后的印章文件　test-A.dgst.sig 和原始文件 test-A.txt　传递给B
    
在B端使用公钥验证签名后的印章文件和源文件

    $ openssl dgst -verify client.pub -signature test-A.dgst.sig test-A.txt

如果输出　”Verified OK“　则说明签名验证成功
        
当然，　也可以使用私钥验证签名后的印章文件和源文件

    $ openssl dgst -preverify client.key -signature test-A.dgst.sig test-A.txt

####9.3 使用密钥加／解密文件

假设在Ａ　Ｂ两端(A拥有公私钥，　B拥有公钥)对一个文件进行加／解密

在Ｂ端以公钥加密数据

    $ openssl rsautl -encrypt -pubin -inkey client.pub -in test-B.txt -out test-B.crp
    
B将加密后的文件　test-B.crp 传递给A

在A端以私钥解密数据

    $ openssl rsautl -decrpt -inkey client.key -in test-B.crp -out test-A.txt
    
解密成功后，　得到的　test-A.txt 和　test-B.txt　内容一致

**配合使用　加密和签名　可以保证数据来源的有效性以及数据内容不被泄露和篡改**

###10. 转换x509证书的存储格式

从　PEM 转换为　DER (请参照第５节，　确保PEM证书中包含证书和私钥)

    openssl x509 -in client.pem -inform PEM -out client.der -outform DER
    
从　DER　转换为　PAM

    openssl x509 -in client.der -inform DER -out client.pem -outform PEM
    
###11. PEM证书和PKCS#12证书互相转换  
  
PKCS#12证书有几种不同的后缀:

+ 思科的后缀为 ".p12"
+ 微软的后缀为 “.pfx”

从　PEM证书转换为PKCS#12证书，　例如

    $ openssl pkcs12 -export -inkey server.key -in server.crt -out server.pfx
      	
从　PKCS#12证书转换为　PEM证书

    openssl pkcs12 -nocerts -nodes -in client.p12 -out client.pem
