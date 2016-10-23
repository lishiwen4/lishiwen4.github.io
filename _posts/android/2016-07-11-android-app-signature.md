---
layout: post
title: "android 应用签名"
description:
category: android-app
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. APK

APK是AndroidPackage的缩写，即Android安装包(apk)，APK文件其实就是zip格式的文件，只是后缀被改为了apk，所以用任何能解压zip的软件都可以直接把一个APK文件解压出来

###2. 对称加密和非对称加密

**对称加密(Symmetric Cryptography**) : 加密（encryption）与解密（decryption）用的是同样的密钥（secret key）

+ 因为 加/解密 使用的是相同的密钥， 对称加密是最快速、最简单的一种加密方式
+ 对称加密有很多种算法，例如 DES、3DES、Blowfish、IDEA、RC4、RC5、RC6和AES， 由于它效率很高，所以被广泛使用在很多加密协议的核心当中
+ 对称加密通常使用的是相对较小的密钥，一般小于256 bit
   + 因为密钥越大，加密越强，但加密与解密的过程越慢。如果你只用1 bit来做这个密钥，那黑客们可以先试着用0来解密，不行的话就再用1解；但如果你的密钥有1 MB大，黑客们可能永远也无法破解，但加密和解密的过程要花费很长的时间。密钥的大小既要照顾到安全性，也要照顾到效率，是一个trade-off
+ 对称加密的一大缺点是密钥的管理与分配，即如何把密钥安全地发送到需要解密你的消息的人的手里是一个问题
   + 在发送密钥的过程中，密钥有很大的风险会被黑客们拦截, 现实中通常的做法是将对称加密的密钥进行非对称加密，然后传送给需要它的人

**非对称加密(Asymmetric Cryptography)** : 它使用了一对密钥，公钥（public key）和私钥（private key）， 使用这对密钥中的一个进行加密，而解密则需要另一个密钥

+ 私钥只能由一方安全保管，不能外泄，而公钥则可以发给任何请求它的人
   + 因为不需要传递公钥， 因此相较于对称加密， 安全性大大提高
+ 常用的非对称加密算法是 RSA ECC（椭圆曲线加密算法）、Diffie-Hellman、El Gamal(背包算法)、DSA（数字签名常用）
+ 因为 加/解密 使用不同的密钥， 因此速度较慢

非对称加密算法的这个重大缺点——加密速度慢, 往往不能直接对文件进行加密 因此需要通常配合其它的技术来使用:

+ 当进行加密传输文件时(确保数据不泄露) : 使用非对称加密技术来对“对称加密技术所使用的密钥进行加密”， 传输密钥后， 双方使用对称加密来进行沟通
+ 当进行数字签名时(确保数据来自可信的发送方且数据未被篡改) :  对文件使用摘要算法生成摘要， 再对摘要进行非对称加密
   + 接收方只需要对接收到的数据生成摘要， 在将接收到的摘要解密， 比较2者的值即可

###3. 摘要
   
摘要算法，又叫作Hash算法或散列算法，是一种将任意长度的输入浓缩成固定长度的字符串的算法，注意是“浓缩”而不是“压缩”，因为这个过程是不可逆的,它的特点是:

+ 同内容的文件生成的散列值一定不同；相同内容的文件生成的散列值一定相同
   + 很难找到一种模式，修改了文件，而它的摘要不会变化
   + 由于这个特性，摘要算法又被形象地称为文件的“数字指纹”
+ 不管文件多小（例如只有一个字节）或多大（例如几百GB），生成的散列值的长度都相同，而且一般都只有几十个字符
+ 消息摘要只能保证消息的完整性，并不能保证消息的不可篡改性， 这也是摘要和数字签名的差异
+ 目前比较流行的摘要算法主要有MD5和SHA-1

**这一算法被广泛应用于比较两个文件的内容是否相同——散列值相同，文件内容必然相同；散列值不同，文件内容必然不同**

###4. 数字签名

数字签名就是附加在数据单元上的一些数据,或是对数据单元所作的密码变换。这种数据或变换允许数据单元的接收者用以确认数据单元的来源和数据单元的完整性并保护数据,防止被人(例如接收者)进行伪造

简单来说， 数字签名技术就是:

1. 发送方将消息生成摘要， 并用私钥对消息摘要进行加密
2. 发送方将加密后的摘要和消息原文发送给接收方
3. 接收方对接收到的消息原文使用相同的摘要算法生成消息摘要
4. 接收方使用公钥对接收到的加密后的消息摘要进行解密
5. 接收方比较 3 4 二步得到的消息摘要， 则可说明2点
   + 消息确实时发送者发送的， 因为只有发送者才有加密摘要的私钥， 若是伪造的私钥， 则接收者在 3 4 二步得到的消息摘要一定不相同
   + 消息确实未被篡改， 因为若传递过程中消息内容发生变化， 则接收者在 3 4 二步得到的消息摘要一定不相同

因此一次数字签名涉及到一个摘要算法、发送者的私钥钥、接收方的公钥,即 “数字签名是非对称密钥加密技术 + 数字摘要技术 的结合”

简单来说， 数字签名有两种功效：

1. 确定消息确实是由发送方签名并发出来的，因为别人假冒不了发送方的签名
2. 确定消息的完整性，因为数字签名的特点是它代表了文件的特征，文件如果发生改变，数字摘要的值也将发生变化，不同的文件将得到不同的数字摘要

###5. 数字证书

数字证书是一个经证书授权中心(CA, Certificate Authority)数字签名的包含公开密钥拥有者信息以及公开密钥的文件。最简单的证书包含一个公开密钥、名称以及证书授权中心的数字签名, 可以由权威公正的第三方机构，即CA（例如中国各地方的CA公司）中心签发的证书，也可以由企业级CA系统进行签发， 或者自己制作自签名证书

**用于APK签名时用到的数字证书是用户自己在本地生成的，并不需要权威机构的发布或认证，是一个自签名的数字证书**

数字证书还有一个重要的特征就是只在特定的时间段内有效

数字证书的格式普遍采用的是X.509V3国际标准，一个标准的X.509数字证书包含以下一些内容：

+ 证书的版本信息；
+ 证书的序列号，每个证书都有一个唯一的证书序列号；
+ 证书所使用的签名算法；
+ 证书的发行机构名称，命名规则一般采用X.500格式；
+ 证书的有效期，通用的证书一般采用UTC时间格式，它的计时范围为1950-2049；
+ 证书所有人的名称，命名规则一般采用X.500格式；
+ 证书所有人的公开密钥；
+ 证书发行者对证书的签名

即

    CA《A》=CA{V，SN，AI，CA，UCA，A，UA，Ap，Ta}
    
数字证书通常有 cer 和 pfx 2种文件存储格式， 其区别如下

+ DER 编码二进制格式的证书文件， 以cer作为证书文件后缀名，证书中不包含私钥
+ BASE64 编码格式的证书文件， 以cer作为证书文件后缀名，证书中不包含私钥
+ 带有私钥的证书由（Public Key Cryptography Standards #12，PKCS#12）标准定义，以pfx作为证书文件后缀名， 包含了公钥和私钥的二进制格式的证书形式

###6. keytool 与 keystore

keytool 是一个JAVA环境下的安全钥匙与证书的管理工具, 用于管理 keystore

keystore 是一个证书库(相当一个数据库，里面可存放多个X.509标准的证书)， 可以存储多条证书(采用唯一的， 不区分大小写的别名来区分不同的证书)， 每一条证书包含如下的信息:

+ 私钥
+ 公钥
+ 对应的数字证书的信息

keystore 中的每一条证书可以导出数字证书文件, 到出的数字证书文件只包含 主体信息 和 对应的公钥

默任 keystore 实现是一个文件， 它利用一个密码来保护私钥

因为 keystore 可以存储多个证书， 可以利用 KeyStore 可以存储一个 X.509 证书链

keytool 常用命令:

    //生成一个新的公/私鈅对到keystore中， duke 为别名， dukekeypasswd 为私钥密码
    //linux 下默认的keystore文件为 ~/.keystore 
    $ keytool -genkey -alias duke -validity 20000 -keypass dukekeypasswd
    Enter keystore password:
    
    //生成一个新的公/私鈅对到keystore中， duke 为别名， dukekeypasswd 为私钥密码
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore
    $ keytool -genkey -alias duke -validity 20000 -keypass dukekeypasswd -keystore test.keystore
    Enter keystore password:
        
    //修改证书的私钥密码， duke 为别名， dukekeypasswd 为旧私钥密码， newpass 为新私钥密码
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore   
    $ keytool -keypasswd -alias duke -keypass dukekeypasswd -new newpass -keystore test.keystore

    //修改 keystore 的密码
    $ keytool -storepasswd -new 123456  -storepass 789012 -keystore test.keystore
    
    //查看一个 keystore 的内容
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore   
    $ keytool -list -v -keystore test.keystore    
    Enter keystore password:

    //从 keystore 中导出证书文件, 以默认的二进制存储， 包含主题信息以及公钥， 不包含私钥
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore   
    $ keytool -export -alias duke -keystore test.keystore -file duke.cer 
    Enter keystore password:
    
    //从 keystore 中导出证书文件, 以 base64 存储， 包含主体信息以及公钥， 不包含私钥
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore   
    $ keytool -export -alias duke -keystore test.keystore -rfc -file duke.cer 
    Enter keystore password:
    
    //查看导出的证书文件
    $ keytool -printcert -file duke.cer 
     
    //导入证书到 keytore， 若指定的 keystore 不存在， 则会新建一个keystore
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore   
    $ keytool -import -alias duke -file testkey -keystore test.keystore
    Enter keystore password:
    
    //删除 KeyStore 中的一条证书
    //test.keystore 为指定的 keystore， linux 下默认的keystore文件为 ~/.keystore   
    $ keytool -delete -alias duke -keystore test.keystore
    
###7. jarsigner

jarsinger 是jar包签名/校验工具 (也可以用于 android apk 的签名/校验)，其usage 如下:

    //签名JAR文件
    jarsigner [ options ] jar-file alias
    
    //校验JAR文件的签名
    jarsigner -verify [ options ] jar-file
    
常见的 option 如下:

+ -keystore url : 指定keystore的 URL。缺省值是用户的宿主目录中的 .keystore 文件
+ -storepass password : 指定访问keystore所需的密码，这仅在签名（不是校验）JAR 文件时需要， 若不指定， 则会提示用户输入
+ -keypass password : 指定访问keystore中证书的私钥所需的密码， 这仅在签名（不是校验）JAR 文件时需要， 若不指定或者指定有误， 则会提示用户输入
+ -sigfile file : 指定用于生成 .SF 和 .DSA 文件的基本文件名。例如，如果 file 为“DUKESIGN”，则生成的 .SF 和 .DSA 文件将被命名为“DUKESIGN.SF”和“DUKESIGN.DSA”，并将其放到已签名的 JAR 文件的“META-INF”目录中, file 中的字符只允许字母、数字、下划线和连字符, 并且将被转换为大写字符， 若不指定该选项， 则.SF 和 .DSA 文件的基本文件名将是命令行中指定的别名的前 8 个字符，并全部被转换为大写。如果别名少于 8 个字符，将使用整个别名。如果别名中包含签名文件名所不允许的字符，则形成文件名时这样的字符将被转换为下划线 ("_")
+ -signedjar file ： 指定签名后的 JAR 文件的名称, 若不指定， 则会覆盖原始文件
+ -certs : 如果它与 -verify 和 -verbose 选项一起出现在命令行中，则输出将包括 JAR 文件的每个签名人的证书信息。该信息包括
   + 验证签名人公钥的证书的类型名（保存在 .DSA 文件中）
   + 如果该证书是 X.509 证书（更准确地说是 java.security.cert.X509Certificate 的实例）： 签名人的特征名
   + 密钥仓库也被检查。如果命令行中没有指定密钥仓库值，缺省密钥仓库文件（如果有）将被检查。如果签名人的公钥证书与密钥仓库中的项匹配，则还将显示下列信息：该签名人的密钥仓库项的别名，在圆括号中。如果该签名人实际上来自于 JDK 1.1 身份数据库而不是密钥仓库，则别名将显示在方括号而不是圆括号中
+ -verbose : 代表“verbose”模式，它使 jarsigner 在 JAR 签名或校验过程中输出额外信息

示例如下:

    //使用 ~/mystore.keystore 中别名为 duke 的证书对 test.jar 文件进行签名， 生成 test-signed.jar 文件
    //并且 test-signed.jar 文件中 .SF 和 .DSA 文件的名为 CERT.SF 和 CERT.DSA
    $ jarsigner -keystore ~/mystore.keystore -storepass myspass -keypass j638klm -sigfile cert -signedjar test-signed.jar test.jar duke
    
    //若不指定密码， 由命令行提示用户输入， 且不指定 .SF .DSA 文件的名称， 则上述命令可缩短为
    $ jarsigner -keystore ~/mystore.keystore -signedjar test-signed.jar test.jar duke
    
    //校验已签名的 JAR 文件，验证签名合法且 JAR 文件未被更改过, 如果校验成功，将显示 "jar verified"
    $ jarsigner -verify test-signed.jar
    
    //校验已签名的 JAR 文件，验证签名合法且 JAR 文件未被更改过可以指定-verbose 选项和 -certs 选项， 获取更多信息
    $ jarsigner -verify -verbose -certs test-signed.jar        
    
###8. android 应用签名

应用签名机制在Android应用和框架中有着十分重要的作用， 通常应用在如下的场景

+ 程序升级 : 当新版程序和旧版程序的数字证书相同时，Android系统才会认为这两个程序是同一个程序的不同版本。如果新版程序和旧版程序的数字证书不相同，则Android系统认为他们是不同的程序，并产生冲突，要求新程序更改包名, 否则会安装失败
+ 程序的模块化设计和开发 : Android系统允许拥有同一个数字签名的程序运行在一个进程中，Android程序会将他们视为同一个程序。所以开发者可以将自己的程序分模块开发，而用户只需要在需要的时候下载适当的模块
+ 通过权限(permission)的方式在多个程序间共享数据和代码： Android 提供了基于数字证书的权限赋予机制，应用程序可以和其他的程序共享概功能或者数据给那那些与自己拥有相同数字证书的程序。如果某个权限 (permission)的protectionLevel是signature或者 signatureorsystem，则这个权限就只能授予那些跟该权限所在的包拥有同一个数字证书的程序

Android对应用签名的要求如下:

+ Android要求所有的应用程序都必须有数字证书，Android系统不会安装一个没有数字证书的应用程序
+ Android程序包使用的数字证书可以是自签名的，不需要一个权威的数字证书机构签名认证
+ 如果要正式发布一个Android ，必须使用一个合适的私钥生成的数字证书来给程序签名，而不能使用adt插件或者ant工具生成的调试证书来发布
+ 数字证书都是有有效期的，Android只是在应用程序安装的时候才会检查证书的有效期。如果程序已经安装在系统中，即使证书过期也不会影响程序的正常功能
+ Android使用标准的java工具 Keytool and Jarsigner 来生成数字证书，并给应用程序包签名(还可以使用android自己的signapk.jar工具来给 android package签名)
+ Android Market强制要求所有应用程序数字证书的有效期要持续到2033年10月22日以后

**使用 Eclipse 和 Android Studio　这些 IDE　开发app时， debug版本会使用　~/.android/debug.keystore　来进行签名**, 该 keystore 的密码为 “android”, 包含一个 alias 为 androiddebugkey　的证书，　私钥密码为　“android”，　这一 keystore　在不同的机器上可能不一样

###9. andropid 应用签名的原理

同 JAR 签名一样， 签名后的 apk 文件解压缩后， 可得 META-INF 目录, 里面存在的3个文件都是 apk 在签名过程中生成

+ MANIFEST.MF : 保存的是除了 META-INF 目录中的 MANIFEST.MF  CERT.SF  CERT.RSA 这3个文件外， 所有其它文件的摘要信息
+ CERT.SF : 保存的是MANIFEST.MF的摘要值，以及MANIFEST.MF中每一个摘要项的摘要值
+ CERT.RSA : 保存的是利用密钥对CERT.SF进行加密生成的数字签名和签名时用到的数字证书，数字证书保存的就是公钥和签名算法
   + `keytool -printcert -file CERT.RSA` 命令可以查看签名的详细信息
   
**因此若需要除去一个android package的签名信息， 只需要删除 apk 文件中的 META-INF 目录 即可**

android 提供的应用签名工具 signapk.jar 的源代码在 build/tools/signapk ，签名主要有以下几步

1. 生成 MANIFEST.MF 文件 (不涉及证书/公/私钥)
   + 首先将除了 CERT.RSA ， CERT.SF ， MANIFEST.MF 之外的所有非目录文件分别用 SHA-1 计算摘要信息
   + 然后使用 base64 进行编码，存入 MANIFEST.MF 中，如果 MANIFEST.MF 不存在，则需要创建， 存放格式是 entry name 以及对应的摘要
2. 生成 CERT.SF (不涉及证书/公/私钥)
   + 对整个 MANIFEST.MF 文件进行 SHA-1 计算，获得摘要信息以base64存入 CERT.SF 中 
   + 对MANIFEST.MF 文件中的每一个摘要项分别使用 SHA-1 计算， 获得摘要信息以base64并存入 CERT.SF 中
3. 生成 CERT.RST 文件 (保存了签名和公钥证书)
   + 使用证书和和私钥对 CERT.SF 进行数字签名 (签名算法会在公钥中定义)
   + 将签名信息和公钥存放到  CERT.RSA 文件中
   
**这几个文件重要的是后缀名，　文件名不重要**
   
###10. android 签名的验证步骤
   
安装时对一个 package 的签名验证的主要逻辑在 JarVerifier.java 文件的 verifyCertificate() 函数中实现。 其主要的思路是通过提取 cert.rsa 中的证书和签名信息，获取签名算法等信息，然后按照之前对 apk 签名的方法进行计算，比较得到的签名和摘要信息与 apk 中保存的匹配


+ 验证 apk 文件的完整性
1. 检查 apk 中除了 CERT.RSA ， CERT.SF ， MANIFEST.MF 之外的所有非目录文件的摘要值与 MANIFEST.MF 中记载的摘要值是否一致
2. 若相同则说明 apk 文件未被篡改
+ 提取apk的证书信息，并对 CERT.SF 进行完整性验证 
1. 寻找是否有 CERT.DSA 或者 CERT.RSA 文件， 若找到则 decode， 提取里面的证书列表
2. 读取 CERT.DSA 或者 CERT.RSA 文件中的签名数据信息块列表， 只取第一个签名数据块。读取其中的发布者和证书序列号。
3. 根据证书序列号，去匹配第1步得到的所有证书，找到与之匹配的证书
4. 从第2步得到的签名数据块中读取签名算法和编码方式等信息
5. 使用第3步找到的证书以及第4步获得的签名算法和编码方式 对 CERT.SF 文件进行数字签名验证，确保CERT.SF 文件未被篡改过 
+ 使用 CERT.SF 中的摘要信息， 验证 MANIFEST.MF 未被篡改过
1. 计算 MANIFEST.MF 文件的摘要
2. 计算 MANIFEST.MF 文件中， 每一个摘要项的摘要
3. 读取 CERT.SF 中保存的摘要信息， 和 1, 2 二步获得的摘要比较， 若相同， 则说明 MANIFEST.MF 文件未被篡改

apk 安装时，　PackageParse.collectCertificates()　会收集 package　的所有签名信息保存在　PackageParse.Package.mSignatures　数组中, 安装过程中, PakcageManagerService.verifySignaturesLP()　会检查 pcakage 的签名是否匹配

安装成功后可通过其对应的 PackageInfo.signatures 数组获取该package的所有签名信息(android允许对一个pcakage使用多个证书进行签名)

###11. android package 使用多个证书签名

Android中是允许使用多个证书对应用进行签名的, 若应用有多个签名，　则 apk 文件的 META-INFO　目录中将会有多个　.RSA 和　.SF　文件 (每一个证书签名会对应一个　.RSA 和　.SF　文件)

使用　signapk.jar　来对一个 android package 来进行多次签名时，　因此不能自己指定　生成的 .SF .RSA　文件的文件名，　因此需要一次指定所有的证书文件(若分开执行，　则后一次签名时会清除前一次的签名信息)

例如

    $ java -jar signapk.jar testkey.x509.pem testkey.pk8 platform.x509.pem platform.pk8 app-debug.apk app-debug-s2.apk
    
则　app-debug-s2.apk　文件的　META-INFO　目录中将会有 CERT0.RSA, CERT0.SF, CERT1.RSA, CERT1.SF 这些文件


使用　jarsigner 对应用进行签名时，　因为默认使用证书的　alias　作为 .RSA 和　.SF　的文件名，　并且还可以自己指定　.RSA 和　.SF　的文件名，　因此，　只需要保证每一次签名时，　生成的　.RSA 和　.SF　的文件名不相同，　就不会覆盖旧的签名信息，　因此，　jarsigner 可分开对 apk 文件使用不同的证书进行多次签名

**需要注意的是，　在升级安装apk时，　android　会检查apk所有的签名信息，　只有所有的签名都能匹配上时，　才能成功升级安装**

###1２. android 系统默认的4组证书

android 源码中， 自带了 development/tools/make_key 工具用于生成数字签名所需的证书

例如 

    $ development/tools/make_key testkey  '/C=US/ST=California/L=Mountain View/O=Android/OU=Android/CN=Android/emailAddress=android@android.com'

android 系统编译过程中， 会使用该 tool 生成4组test证书(用于 eng debug 版的image), 置于build/target/product/security目录中:

1. testkey：普通APK，即在android package不需要使用特殊的证书来签名时， 使用该组证书来签名(因为android package必须要签名)
2. platform：用于系统核心功能的android package， 如需要share user ID 为 system时， 就要使用platform key 来签名
   + 例如 Settings, Bluetooth, Mms 等使用 platform key 来签名
3. shared：用于需要和home/contacts进程共享数据的android package
   + 例如 Dialer， Contacts， QuickSearchBox， Launcher 等使用 shared key 签名
4. media：用于media/download 这一系统的android package 
   + 例如 Gallery， MediaProvider， DownloadProvider 等使用 media key 签名

**正式发布的 android rom， 应该使用自己生成的 release key， 不能使用 andriod source中生成的 testkey**

###1３. Android.mk 中配置应用签名

Android.mk中有一个 LOCAL_CERTIFICATE字段， 由它指定用哪个key签名

例如

    LOCAL_CERTIFICATE := platform
    
未指定该字段的情况下， 默认使用testkey 来签名

###1４. sinapk.jar

android 提供的应用签名工具, 直接使用公私钥文件来对android package来进行签名

signapk.jar 的源代码在 build/tools/signapk 目录下， build 后生成 out/host/linux-x86/framework/signapk.jar

signapk.jar 的 usage 如下:

    signapk [-w] [-a <alignment>] [-providerClass <className>] publickey.x509[.pem] privatekey.pk8 [publickey2.x509[.pem] privatekey2.pk8 ...] input.jar output.jar
    
通常的用法是:

    signapk publickey.x509[.pem] privatekey.pk8 input.jar output.jar
    
例如:

    $ java -jar signapk platform.x509.pem platform.pk8 test.jar test-s.jar
    
**signapk　签名生成的　.RSA　.ＳＦ 的文件名固定为　CERT，　而　jarsigner　则是使用签名所用的证书的　alias 作为这２个文件的文件名**

###1５. pem 证书转换 keystore

Eclipse 和 Android Studio 中配置应用签名, 需要使用 keystore 形式存储的证书文件， 而非直接使用 pem/pk8 形式的 公/私钥 文件， 因此， 若需要在 IDE 中使用指定的 pem/pk8 形式的 公/私钥 文件来为应用签名，则需要向将其转换为 keystore 形式

以 android 的 platform key 为例

    // 把pkcs8格式的私钥转换为pkcs12格式
    $ openssl pkcs8 -in platform.pk8 -inform DER -outform PEM -out platform.priv.pem -nocrypt
    
    // 生成pkcs12格式的证书文件, 需为生成的证书文件指定密钥密码
    $ openssl pkcs12 -export -in platform.x509.pem -inkey platform.priv.pem -out platform.pk12 -name releasekey
    Enter Export Password:
    
    // 生成keystore， 需要为 keystore 指定密码， 以及输入 pkcs12 证书文件的密码
    $ keytool -importkeystore -deststorepass storepaswd -destkeypass keypaswd -destkeystore debug.keystore -srckeystore platform.pk12 -srcstoretype PKCS12 -srcstorepass android -alias androiddebugkey
    Enter destination keystore password:  
    Re-enter new password: 
    Enter source keystore password:
    
###1６. Gradle 配置应用签名

使用 Gradle 构建 android project 时， 可以配置数字签名， 必须使用 keystore 形式存储的证书， 可以使用 keytool 直接生成 keystore 或者是将数字证书转换为 keystore 的形式

**Gradle build type **, 默认情况下，Gradle 的 Android Plugin会自动给项目设置同时构建应用程序的 debug 和 release 版本 (在 Android Studio 中， 可以从左下角打开 Build Variants 视窗， 选择 build type)， debug 和 release 这两个版本之间的不同主要围绕着能否在一个安全设备上调试，以及APK如何签名:
jarsigner
+ Debug版本采用使用通用的name/password键值对自动创建的数字证书进行签名，以防止构建过程中出现请求信息
+ Release版本在构建过程中没有签名，需要稍后再签名

这些配置通过一个BuildType对象来配置， Android plugin允许像创建其他构建类型一样定制debug和release实例， 这需要在buildTypes的DSL容器中配置， 例如， 在 app 的 build.gradle 文件里面

    android {
    
        ......
        buildTypes {
            debug {
                //打包时， 给 package name加上后缀 ".debug"
                applicationIdSuffix ".debug"
            }
            
            release {
                ......
            }
            ......
        }
        ......
    }

**Gradle signingConfigs**: 若需要为不同的build type， 指定不同的签名方式， 则可以在 gradle 配置文件的 “signingConfigs{}” 中配置好签名方式， 然后在 “buildTypes{}” 里对应的type里面指定使用不同的签名配置即可， 例如

    android {
        ......
        signingConfigs{
            //type1 也可以自行命名为其它
            type1{
                storeFile file("release.kerystore")
                storePassword "******"
                keyAlias "release"
                keyPassword "********"
            }
            ......
        }
        ......
        buildTypes {
            ......
            release {
                ......
                //type1 需根据实际的名称来替换
                signingConfigs signingConfigs.type1
                ......
            }
            ......
        }
        ......
    }