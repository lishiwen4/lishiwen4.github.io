---
layout: post
title: "linux内核模块签名"
description: 
category: linux-kernel
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. linux内核模块签名

linux内核从3.7 开始加入模块签名检查机制， 校验签名是否与已编译的内核公钥匹配。目前只支持RSA X.509验证， 模块签名验证并非强制使用， 可在编译内核时配置是否开启

linux 内核中与模块签名验证相关的config项有：  

+ CONFIG_MODULE_SIG ： 模块签名验证功能， 配置为‘y’后， 还有如下的子选项可以配置
+ CONFIG_MODULE_SIG_FORCE ：是否验证模块的签名， 配置为‘n’时， 对于未签名或者使用无效的key签名的模块， 内核都允许加载， 但是kernel会被标记为被感染(tainted), 并且使用商业许可的模块也会被标记为被感染, 若配置为‘y’， 则只有正确签名的模块才能被允许加载， 其他的模块将被拒绝加载， 并且kernel会打出错误信息  
+ CONFIG_MODULE_SIG_ALL ： 若配置为'n', 则在build kernel的过程中， 所有的模块都不会被签名， 需要自己手动签名， 若被配置为‘y’， 则在build kernel的 “modules_install”阶段， 所有的模块都会被自动签名  

若不希望在加载模块的过程中进行签名验证， 只需要在build kernel 时关闭相关config项即可
**若build kernel 时打开了CONFIG_MODULE_SIG， 但是没有打开CONFIG_MODULE_SIG_FORCE， 则还可以在启动kernel时差undineihe参数”module.sig_enforce“来打开模块签名验证**
内核提供了多种签名算法供使用， 可在编译内核时进行配置  

+ CONFIG_MODULE_SIG_SHA1
+ CONFIG_MODULE_SIG_SHA224
+ CONFIG_MODULE_SIG_SHA256
+ CONFIG_MODULE_SIG_SHA384
+ CONFIG_MODULE_SIG_SHA512

当选中某一中算法时， 该算法相关的代码也会被built-in到内核中

###2. 签名所用的key

在编译内核时， 需要一对公钥和私钥文件

+ signing_key.priv : 私钥用于对模块进行签名
+ signing_key.x509 : 公钥则会被编译进内核中， 用于为模块进行签名验证

公钥和私钥文件需要放置在编译内核时的out目录的根目录下， 内核的Makefile中会使用“MODPUBKEY”和“MODSECKEY”两个变量来指定公钥和私钥文件

默认情况下， 在编译内核的过程中， 会利用 /dev/random 中的数据自动生成公钥和私钥， 当然， 在生成公钥和私钥前， 还会生成必需的根证书文件 x509.genkey, 这些文件的生成过程是在 KERNEL_SRC/kernel/Makefile 中完成的  

    signing_key.priv signing_key.x509: x509.genkey
        @echo "###"
        @echo "### Now generating an X.509 key pair to be used for signing modules."
        @echo "###"
        @echo "### If this takes a long time, you might wish to run rngd in the"
        @echo "### background to keep the supply of entropy topped up.  It"
        @echo "### needs to be run as root, and uses a hardware random"
        @echo "### number generator if one is available."
        @echo "###"
        openssl req -new -nodes -utf8 -$(CONFIG_MODULE_SIG_HASH) -days 36500 \
                -batch -x509 -config x509.genkey \
                -outform DER -out signing_key.x509 \
                -keyout signing_key.priv 2>&1
        @echo "###"
        @echo "### Key pair generated."
        @echo "###"

    x509.genkey:
        @echo Generating X.509 key generation config
        @echo  >x509.genkey "[ req ]"
        @echo >>x509.genkey "default_bits = 4096"
        @echo >>x509.genkey "distinguished_name = req_distinguished_name"
        @echo >>x509.genkey "prompt = no"
        @echo >>x509.genkey "string_mask = utf8only"
        @echo >>x509.genkey "x509_extensions = myexts"
        @echo >>x509.genkey
        @echo >>x509.genkey "[ req_distinguished_name ]"
        @echo >>x509.genkey "O = Magrathea"
        @echo >>x509.genkey "CN = Glacier signing key"
        @echo >>x509.genkey "emailAddress = slartibartfast@magrathea.h2g2"
        @echo >>x509.genkey
        @echo >>x509.genkey "[ myexts ]"
        @echo >>x509.genkey "basicConstraints=critical,CA:FALSE"
        @echo >>x509.genkey "keyUsage=digitalSignature"
        @echo >>x509.genkey "subjectKeyIdentifier=hash"
        @echo >>x509.genkey "authorityKeyIdentifier=keyid"

按照上面Makefile中的步骤， 我们可以很容易生成自己定制的的key pairs， 如果需要使用自己的key pairs而不是随机生成，请按照如下的步骤

+ 若编译内核时， src目录和out目录重合， 则直接将生成的key pairs文件放置在 kernel src 目录的根目录下
+ 若编译内核时， src目录和out目录不重合， 则直接将生成的key pairs文件放置在 kernel out 目录的根目录下， 且不要在kernel src 目录下放置这两个key文件以及任何和这两个文件同名的文件

###3. 内核中的公钥

模块签名所需要的公钥会被编译到内核中， 事实上内核中可以保存多份公钥(编译内核时， 在src或out目录根目录中的所有以".x509"为后缀的文件都会作为公钥被编译进内核), 这涉及到linux 内核的密钥暂存服务， 会单独讲解

为查看内核中的所有公钥， 可以使用

    $ cat /proc/keys
    
或者查看linux boot时加载的模块签名的公钥的log

    $ dmesg | grep “MODSIGN: Loaded cer”
    
###4. 模块签名

linux 源码中提供了模块签名的工具 “ scripts/sign-file”， 在Makefile 中可以看到其使用方法

    mod_sign_cmd = perl $(srctree)/scripts/sign-file $(CONFIG_MODULE_SIG_HASH) $(MODSECKEY) $(MODPUBKEY)
    
 + CONFIG_MODULE_SIG_HASH ：  可以取值1, 224, 256, 384, 512
 + MODSECKEY ： 私钥文件
 + MODPUBKEY ： 公钥文件
 
 例如：
    
    $ scripts/sign-file sha512 kernel-signkey.priv kernel-signkey.x509 module.ko
    
签名后的模块， 在文件尾部回附加“~Module signature appended~.”字符串， 可以使用二进制文本编辑器查看

    $ hexdump -C cpuid.ko  | tail -n 40

    00 00 00 00 00 00 00 00  4d 61 67 72 61 74 68 65  |........Magrathe|
    61 3a 20 47 6c 61 63 69  65 72 20 73 69 67 6e 69  |a: Glacier signi|
    6e 67 20 6b 65 79 00 a5  a6 57 59 de 47 4b c5 c4  |ng key...WY.GK..|
    31 20 88 0c 1b 94 a5 39  f4 31 02 00 29 18 64 dd  |1 .....9.1..).d.|
    8c a6 b7 db 18 a3 e1 82  f0 58 97 07 be 7c 97 4a  |.........X...|.J|
    3e 19 36 84 f8 c6 8d a9  92 3b 53 44 9a 9b f2 ab  |>.6......;SD....|
    9a 34 82 0c 87 f1 2b 29  8a 9b 24 ea 0a db 3b 4c  |.4....+)..$...;L|
    76 68 89 a7 d2 95 a9 9f  32 ff 69 ee 35 3a c4 35  |vh......2.i.5:.5|
    c3 c3 c4 66 17 bf 98 23  a1 25 80 8d 2f 4f 9d ef  |...f...#.%../O..|
    e0 24 77 d3 6b b5 62 b8  5e 90 3b 23 8c 88 4d d8  |.$w.k.b.^.;#..M.|
    a1 0c 71 08 01 92 18 ba  c0 86 da 0d 07 ee ba 02  |..q.............|
    c8 86 58 18 d0 0f 3b 90  c1 81 b6 aa d1 71 21 58  |..X...;......q!X|
    75 f2 38 b0 89 12 e4 8d  71 6f 2e d0 2e 34 b3 fe  |u.8.....qo...4..|
    a5 b8 46 b1 67 95 72 93  59 56 4a 69 30 d9 a7 24  |..F.g.r.YVJi0..$|
    cd 02 cd c3 01 cf ed 24  34 b9 7f 20 c8 df f4 22  |.......$4.. ..."|
    be 04 21 c4 38 aa 83 ba  b9 2a b1 59 c7 26 66 76  |..!.8....*.Y.&fv|
    8e 50 90 32 6b 14 46 60  bf f6 d5 f9 44 6e 0d 10  |.P.2k.F`....Dn..|
    3f de e8 59 3b d6 fd da  ea a6 d5 96 03 f7 27 ea  |?..Y;.........'.|
    34 b1 f9 67 96 7e 8d 87  46 a9 53 1f b0 bc 3f 5b  |4..g.~..F.S...?[|
    da 8e d1 fc 10 2a 91 d4  5a 88 91 38 47 54 33 90  |.....*..Z..8GT3.|
    b7 b2 fa 36 a5 c0 83 fd  a6 19 85 de ee f7 06 90  |...6............|
    6e 11 79 df bc 3b 6c fb  7a aa 52 a1 83 79 e2 13  |n.y..;l.z.R..y..|
    83 22 79 29 de e7 15 6a  b1 f5 15 a5 f3 23 71 91  |."y)...j.....#q.|
    de f7 2e 69 f1 d0 ee 56  ad 2f 5b 60 4b 42 65 ba  |...i...V./[`KBe.|
    d0 ef ac c6 82 04 de a7  80 5a f4 e8 23 ff c2 6c  |.........Z..#..l|
    a3 2b 8d d6 b8 26 e2 d2  95 7d 3e f5 d9 a0 36 10  |.+...&...}>...6.|
    16 7b 77 6a 19 0d d5 4d  fd ec d4 36 42 da 12 54  |.{wj...M...6B..T|
    9e a3 20 b6 11 e3 c3 cd  da b4 a5 1b 72 d7 24 9b  |.. .........r.$.|
    d4 10 03 95 6d 79 21 5d  91 56 de f3 40 29 a9 5e  |....my!].V..@).^|
    a6 9e a4 c9 be d4 09 09  28 44 11 30 6b 8c 64 4c  |........(D.0k.dL|
    94 aa 16 ec 04 68 b6 7f  c8 5d a5 25 27 b3 15 91  |.....h...].%'...|
    9b 49 81 8a 01 80 00 fd  9d 0a 3e d8 7f 9c 44 0b  |.I........>...D.|
    f0 32 b6 f0 45 06 83 9c  9a 12 c0 dc 40 18 28 15  |.2..E.......@.(.|
    98 c7 e0 46 09 d5 1c 69  eb 6e 80 44 9b de fb 4c  |...F...i.n.D...L|
    d2 a7 1c d2 b6 69 72 ae  71 5e 61 68 96 6e a2 91  |.....ir.q^ah.n..|
    d6 fe 44 e6 84 4b a3 b6  5a 72 e8 37 01 06 01 1e  |..D..K..Zr.7....|
    14 00 00 00 00 00 02 02  7e 4d 6f 64 75 6c 65 20  |........~Module |
    73 69 67 6e 61 74 75 72  65 20 61 70 70 65 6e 64  |signature append|
    65 64 7e 0a                                       |ed~.|
    
一个内核模块能够被多次签名， 这些签名信息会依次累加，但是在校验时， 只会使用最后一个签名信息， 若希望清除该模块的签名信息， 则可以使用

    $ stripe --strip--debug cpuid.ko
    
该命令会清除所有的签名信息

###5. linux内核模块的签名验证

若kernel打开了模块签名验证， 则在加载kernel module的过程中， 会进行模块的签名验证，由“kernel/module.c” 中的 “module_sign_check()” 来完成

事实上， linux 的kernel module 加载过程中存在诸多检查， 模块签名验证只是第一步， 后面还会有 vermagic 和 CRC验证

###END


