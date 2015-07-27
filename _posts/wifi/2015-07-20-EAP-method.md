---
layout: post
title: "EAP 认证"
description:
category: wifi
tags: [Data]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. PAP 认证

PAP（PPP Authentication Protocol）密码认证协议，在RFC-1334中定义，起源于PPP的认证(正是PPP的认证和计费， 后续还出现了PPPoE)，是典型的2次握手简单认证协议，PAP 并不是一种强有效的认证方法，其密码以文本格式在电路上进行发送，对于窃听、重放或重复尝试和错误攻击没有任何保护

PAP的标准化RFC-1334是在1992年提出的，那个时候还没有AAA服务器，所以只涉及到PC和NAS之间通信，这里的PAP（包括后面的CHAP）定义标准的时候都是指的PC和NAS间的认证，PAP和CHAP都是NAS来验证PC，而PC没有验证NAS，也就是所谓的单向验证

PAP协议很简单，其实就是在PPP建立过程中，PC需要将用户名和密码发送给NAS，NAS来验证用户名和密码的合法性，这个传送过程是明文传送的，这也是大家所谓的不安全，当然这部分确实是不安全的，IETF也建议不要使用PAP认证，在RFC-1334中明确了其已经被1994取代，也就是后面要讲到的CHAP

PAP 认证的流程

1. PC <——> NAS：这部分是不是明文的那要看你PAP是在什么链路上运行的，如果是PPP协议，那可能是明文传送密码给NAS的，但是如果PAP认证协议是跑在TTLS（802.1X安全隧道中），那么外层隧道是加密的，所以即使PAP使用的是明文密码，对于隧道外来说仍然是密文的，所以简单的说PC和NAS之间的PAP是明文密码不安全的，是比较片面的；
2. NAS <——>AAA：这部分就更不会是明文的了，NAS如果使用PAP认证协议找Radius服务器进行认证，那么他会将密码首先加密，放到User-Password属性中，交给Radius服务器，Radius服务器再进行解密获取原始密码，跟数据库中的密码进行比对得到认证结果；
所以我们常说的PAP认证使用明文密码，是不安全的，一般指的是1中PC和NAS之间的PAP认证协议，而Radius的PAP认证中并没有使用明文密码，是比较安全的（当然Radius的PAP密码保护使用了可逆的加密算法，安全程度不是太高）。

关于Radius协议中PAP认证协议的密码是如何加密的，具体实现可以查看RFC-2865， 大体是将用户密码按照16字节截断为p1、p2......，然后依次进行加密，形成密文c1+c2+....+cn
其中
	c1 = p1 XOR MD5(secret + Authenticator)
	c2 = p2 XOR MD5(secret + c1)
	....
	cn = pi XOR MD5(secret + c(n-1))

当Radius服务器收到认证请求的时候，使用和NAS配置的共享密钥Secret，加上包头中的Authenticator，就能解密出原始的明文密码

###2. CHAP 认证

CHAP(Challenge Handshake Authentication Protocol）询问握手认证协议，或者叫做冲击握手认证协议，属于三次握手协议，相对于PAP来说安全性高一些，防止了重放攻击，因为每次询问内容是不同的

1. PC <——> NAS：所说的三次握手也是这个过程，当链路建立以后，首先NAS会向PC发送一个Challenge字符串，这个字符串是随机的，PC利用用户输入的明文密码和Challenge使用单向加密算法加密后将用户名和密文密码一同发送给NAS设备；
2. NAS <——> AAA：NAS设备收到PC发送的认证请求后，将用户名和密码分别放到User-Name和CHAP-Passowrd
中，另外必须携带Challenge给Radius服务器，否则Radius服务器无法验证密码，这个Challenge有两种方式，第一个是放到属性CHAP-Challenge中，如果Challenge长是16字节，也可以直接放到Radius包头中的Authenticator中，NAS将这些内容发送给Radius服务器，Radius服务器本地获取明文密码，加上Challenge后用相同的算法得到密文，与CHAP-Password对比得到结果；
所以看到三次握手指的是PC和NAS之间，而NAS和AAA服务器之间和PAP还是一样，NAS向AAA服务器发送一次请求，里面携带了所有数据，AAA根据这些数据判断将验证结果发送给NAS

Radius对于CHAP-Password解析如下：
1. 第一个字节是CHAP Identifier，是PC发送给NAS的Response的ID值；
2. 计算：C = MD5(CHAP-Ident + 明文密码 + Challenge)计算的到密文；
3. 将C与CHAP-Password从第二字节开始的内容对比

关于Radius对于PAP和CHAP的处理，RFC-2865中有如下描述：

FreeRadius的处理一般是复杂的协议首先确定，比如首先在属性中查找是否有CHAP-Password属性，如果有的话，就默认是CHAP认证，试图使用CHAP认证方式，如果没有找到CHAP-Password，那么看是否有User-Password，如果有User-Password就认为是PAP认证方式，使用PAP模块进行认证

CHAP的特点：

1. 该认证方法依赖于只有认证者和对端共享的密钥，密钥不是通过该链路发送的。
2. 虽然该认证是单向的，但是在两个方向都进行 CHAP 协商，同一密钥可以很容易的实现相互认证。
3. 由于 CHAP 可以用在许多不同的系统认证中，因此可以用 NAME 字段作为索引，以便在一张大型密钥表中查找正确的密钥，这样也可以在一个系统中支持多个 NAME/ 密钥对，并可以在会话中随时改变密钥。
4. CHAP 要求密钥以明文形式存在，无法使用通常的不可回复加密口令数据库。
5. CHAP 在大型网络中不适用，因为每个可能的密钥由链路的两端共同维护。

###3. MS-CHAP 认证

MS-CHAP(MicroSoft Challenge Handshake Authentication Protocol)微软询问握手认证协议，也就是微软定义的CHAP协议，定义在RFC-2433/2548中

在上面CHAP介绍中了解到，Radius服务器必须有明文密码才能验证CHAP-Password的合法性，因为需要将明文密码经过相同的MD5得到密文与CHAP-Password进行比较，而这个明文密码的获取是微软所不允许的，也就是说你通过MS提供的任何API都不能获取系统明文密码（不管是本地还是AD域，MS都不允许获取明文密码，MS压根都没存明文密码），只能获取到密码MD4后的密文（MS这种考虑也是很合理的），既然获取不了明文密码，那就用不了CHAP了，而直接用PAP的话安全程度又太低，所以微软干脆提出了MS-CHAP认证协议。从RFC-2433中CHAP和MS-CHAP区别可以看到：

MS-CHAP有2个版本，V1和V2，其中为了解决上面CHAP明文密码问题，MS首先提出了V1版本，其基本思想就是发送的密码不是CHAP中MD5（明文密码 + Challenge）这种方式了，而是直接使用MS加密后的密文，这就有一个要求，就是PC发送密码的时候用的就是单向加密后的密码再加密，所以认证端也不用明文密码（或者有明文的话，使用明文进行和MS一样的加密获取密文）进行相同操作，然后进行比较。现在使用V1版本比较少了，大多数都使用V2版本，所以下面主要分析V2版本，V2版本比V1版本主要是加入了双向验证，也就是PC也要验证服务器

MS-CHAPv2工作模式主要分为两种：
1. PC <——> NAS  <——> AAA：这是所谓的MS-CHAPv2认证流程，NAS将Challenge发送给PC，PC响应后NAS再组织Radius属性（MSCHAP-Challenge + MSCHAP-Response），然后Radius服务器根据获取到的密码（可能是明文、LM-Password、NT-Password）加上MSCHAP-Challenge计算获取密文同收到的MSCHAP-Response进行比较；
2. PC <————————>AAA：这是EAP/MS-CHAPv2，是基于EAP框架的MS-CHAPv2认证，EAP框架的作用就是透明化NAS设备，NAS设备只做翻译透传工作，而具体的认证协议协商是由Radius服务器和PC完成，所以Radius服务器需要完成1中NAS部分工作，主要是Challenge的发送，然后接收PC的响应，注意这里Radius接收到的不是1中的（MSCHAP-Challenge + MSCHAP-Response），而是原始的MS-CHAPv2的数据包，需要进行解析，所以EAP/MS-CHAPv2的实现对于Radius服务器来说更为复杂，freeRadius在EAP/MS-CHAPv2使用了将MS-CHAPv2数据包转换为（MSCHAP-Challenge + MSCHAP-Response）递归调用MS-CHAP模块来实现的

###4. EAP的出现

从上述几种认证方式可以看出， NAS作为中间人得了解PC客户端和AAA服务器的不同认证方式，比如基于PPP的PAP/CHAP，它还得了解如何将这些基于PPP等网络之上的认证协议转化为RADIUS认证协议（比如PAP转为User-Password属性，CHAP转为CHAP-Password和CHAP-Challenge属性），一旦要支持新的认证方式，参与认证的三方都得进行修改和支持，这里面作为发起人的PC和作为认证结束端的RADIUS是必须支持MS-CHAP，但是作为中间人的NAS网络设备若是能够透明的转发PC——AAA之间的数据包， 则无需修改NAS， 最终出现了EAP 协议

EAP（Extensible Authentication Protocol）可扩展认证协议，RFC-3748定义，是一种普遍使用的支持多种认证方法的认证框架协议，主要用于网络接入认证， **EAP协议一般运行在数据链路层上**， 与其说EAP是一个认证协议不如说EAP是一个认证框架，EAP本身不是认证协议，它自己不支持认证功能，它是为了承载多种认证协议而生的，EAP为扩展和协商认证协议提供了一个标准

以常用的802.1X为例，它就是使用了EAP协议:

1. 在请求者和认证者之间， EAP 协议报文使用EAPOL 封装格式，直接承载于LAN 环境中
2. 在认证者和AAA服务器之间， EAP 协议报文可以使用EAPOR（EAP over RADIUS）封装格式，承载于RADIUS 协议中；也可以由认证者端终结转换，而在认证者与AAA服务器之间传送PAP协议报文或CHAP等协议报文

EAP上面承载的认证协议对NAS是透明的，NAS设备不需要了解，也不需要支持，它只需要支持EAP即可

###5. EAP认证

EAP（Extensible Authentication Protocol），可扩展认证协议，是一种普遍使用的支持多种认证方法的认证框架协议，主要用于网络接入认证， 该协议一般运行在数据链路层上，即可以直接运行于PPP或者IEEE 802之上，不必依赖于IP， EAP可应用于无线、有线网络中。

EAP的架构非常灵活，在Authenticator（认证方）和Supplicant（客户端）交互足够多的信息之后，才会决定选择一种具体的认证方法，即允许协商所希望的认证方法。认证方不用支持所有的认证方法，因为EAP架构允许使用一个后端的认证服务器（也就是AAA服务器），此时认证方将在客户端和认证服务器之间透传消息

该协议支持重传机制，但这需要依赖于底层保证报文的有序传输，EAP不支持乱序报文的接收。协议本身不支持分片与重组，当一些EAP认证方法生成大于MTU（Maximum Transmission Unit，最大传输单元）的数据时，需要认证方法自身支持分片与重组。

现在大约有40种不同的方法。IETF的RFC中定义的方法包括：EAP-MD5, EAP-OTP, EAP-GTC, EAP-TLS, EAP-SIM,和EAP-AKA, 还包括一些厂商提供的方法和新的建议, WPA和WPA2标准已经正式采纳了5类EAP作为正式的认证机制, 无线网络中常用的方法包括EAP-TLS, EAP-SIM, EAP-AKA, PEAP, LEAP,和EAP-TTLS

**wifi认证时， 可选测如下项：**

+ EAP-TLS+ EAP-TTLS/MSCHAPv2
+ PEAPv0/EAP-MSCHAPv2
+ PEAPv1/EAP-GTC
+ EAP-SIM
+ EAP-AKA
+ EAP-AKA`
+ EAP-FAST 

###6. 基于TLS的EAP认证

安全传输层协议TLS (Transport Layer Security）用于在两个通信应用程序之间提供保密性和数据完整性。该协议由两层组成： TLS 记录协议（TLS Record）和 TLS 握手协议（TLS Handshake）。较低的层为 TLS 记录协议，位于某个可靠的传输协议（例如 TCP）上面，与具体的应用无关，所以，一般把TLS协议归为传输层安全协议

TLS被认为是SSL的继承者

IEEE的802.1X使用了EAP认证框架，因为EAP提供了可扩展的认证方法，但是这些认证方法的安全性完全取决于具体的认证方法，比如EAP-MD5、EAP-LEAP、EAP-GTC等， 802.1X最开始是为有线接入设计的，后来被用于无线网的接入，有线接入在安全性方面考虑毕竟少，如果要窃取信息需要物理上连接网络，而无线网完全不同，无线网信号没有物理边界，所以要使用802.1X的话，需要对802.1X进行安全性方面的增强，也就是增强EAP认证框架的安全性，而且要进行双向认证，那么EAP使用了IETF的TLS（Transport Layer Security）来保证数据的安全性

现在有3种基于TLS的EAP认证方法：

1. EAP-TLS
2. EAP-TTLS
3. EAP-PEAP

EAP-TLS使用TLS握手协议作为认证方式，TLS有很多优点，所以EAP选用了TLS作为基本的安全认证协议，并为TTLS和PEAP建立安全隧道，TLS已经标准化，并且进过了长期应用和分析，都没有发现明显的缺点

####6.1 EAP-TLS

EAP-TLS是IETF的一个开放标准，也是无线局域网扩展认证协议的原始版本， 被认为是最安全的EAP标准之一， 而且广泛地被无线局域网硬件和软件制造厂商,包括微软所支持， **到2005年四月，EAP－TLS是唯一厂商需要支持的WPA和WPA2的EAP类型**， 

TLS认证是基于Client和Server双方互相验证数字证书的，是双向验证方法，首先Server提供自己的证书给Client，Client验证Server证书通过后提交自己的数字证书给Server，个脆弱的密码不会导致入侵基于EAP－TLS的系统，因为攻击者仍然需要客户端的证书， 客户端的证书可以放到本地、放到key中或者智能卡中等等

TLS有一个缺点就是TLS传送用户名的时候是明文的，也就是说抓包能看到EAP-Identity的明文用户名，另外TLS是基于PKI证书体系的，这是TLS的安全基础，也是TLS的缺点，PKI太庞大，太复杂了，如果企业中没有部署PKI系统，那么为了这个认证方法而部署PKI有些复杂， 并且用户端在认证时的负载也较高

####6.2 EAP-TTLS

正因为TLS需要PKI的缺点，所以设计出现了TTLS和PEAP，这两个协议不用建立PKI系统，而在TLS隧道内部直接使用原有老的认证方法，这保证了安全性，也减小了复杂度，EAP-TTLS由Funk Software与Certicom合作推出， 是典型的两段式认证，在第一阶段建立TLS安全隧道，通过Server发送证书给Client实现Client对Server的认证（这里TTLS和PEAP仍然使用证书，但是这里的证书都是服务器证书，管理起来比TLS客户端证书要简单那的多）；当安全隧道一旦建立，第二阶段就是协商认证方法，对Client进行认证

TTLS利用TLS安全隧道交换类似RADIUS的AVPs（Attribute-Value-Pairs），实际上这些AVPs的封装和RADIUS都十分相似，TTLS这种AVPs有很好的扩展性，所以它几乎支持任何认证方法，这包括了所有EAP的认证方法，以及一些老的认证方法，比如PAP、CHAP、MS-CHAP、MS-CHAPv2等，TTLS的扩展性很好，通过新属性定义新的认证方法

####6.3 EAP-PEAP

EAP-PEAP 和 EAP-TTLS很像， 由微软，Cisco和RSA合作推出， 是一个轻量级的EAP-TTLS,只需要一份服务器端的PKI证书来建立一个安全的传输层安全通道, EAP-PEAP也是典型的两段式认证，在第一阶段建立TLS安全隧道，通过Server发送证书给Client实现Client对Server的认证（这里TTLS和PEAP仍然使用证书，但是这里的证书都是服务器证书，管理起来比TLS客户端证书要简单那的多）；当安全隧道一旦建立，第二阶段就是协商认证方法，对Client进行认证

PEAP之所以叫Protected EAP，就是它在建立好的TLS隧道之上支持EAP协商，并且只能使用EAP认证方法，这里为什么要保护EAP，是因为EAP本身没有安全机制，比如EAP-Identity明文显示，EAP-Success、EAP-Failed容易仿冒等，所以EAP需要进行保护，EAP协商就在安全隧道内部来做，保证所有通信的数据安全性, **其实PEAP最大的优点就是微软支持开发**，微软在Windows系统内集成了客户端，微软和Cisco都支持PEAP，但是他们的实现有所区别

到2005年5月，已有两个PEAP的子类型被WPA和WPA2标准批准。它们是：

1. PEAPv0/EAP-MSCHAPv2
2. PEAPv1/EAP-GTC

####6.4 EAP-FAST

EAP-FAST (Flexible Authentication via Secure Tunneling) 基于安全隧道的灵活认证， 由Cisco提出的协议方案，用于替代EAP-LEAP

EAP-FAST在普通操作模式下和PEAP类似，有两个主要阶段。第一阶段是建立一个安全的加密隧道，第二阶段是通过 MS-CHAPv2线程在验证服务器上对客户端身份进行验证。由于 MS-CHAPv2是一个相当脆弱的协议，通过字典攻击很容易被破解，因此第一阶段建立的安全加密隧道为MS-CHAPv2线程提供了一个安全的环境。与PEAP不同的是，EAP-FAST采用PAC（受保护的接入证书）来建立隧道，而PEAP则是采用服务器端的数字证书建立TLS隧道（与安全的Web服务器工作方式类似）。验证服务器的EAP-FAST Master Key会为每个用户提供一个特殊的PAC文件。而配发这个PAC的过程可以被认作是“阶段0”（也就是自动提供），或者通过其他一些方式，比如通过移动存储设备传输，通过管理员建立的共享文件夹，或者有用户限制的共享文件夹

虽然与LEAP相比进步很大，但是EAP-FAST的安全性仍然不如EAP-TLS, EAP-TTLS,或PEAP。不过EAP-FAST的验证线程速度较快，因为它采用的是对称加密，而不是EAP-TLS, EAP-TTLS,或PEAP所采用的非对称加密，但是这这种速度优势并没有什么实际意义(可能只是相差几ms) 

###7. 基于移动网络的EAP认证

####7.1 EAP-SIM/AKA

EAP-SIM/AKA 使用从移动网络中获得的鉴权三元组（EAP-SIM认证）或者五元组（EAP-AKA），得到认证需要MAC值，以及加密用的密钥，定义于RFC4186

EAP-SIM用于GSM网络，只支持单向认证，即支持网络认证UE，但是不支持UE认证网络。EAP-AKA是EAP-SIM的升级版本，用于3G网络中的用户认证，支持双向认证。注意，EAP-SIM和EAP-AKA都可以运行在SIM卡上

EAP-SIM基于GSM的认证和密钥分发机制， GSM的安全性需要如下3个算法：

1. A3算法用于认证和授权
2. A5算法用于加密用户通话内容
3. A8算法用于产生A5算法中加密数据和语音的密钥

GSM的认证安全依赖于存储在SIM卡中的128位密钥Ki，Ki永远不会在网络中传输。首先，网络运营商生成128位的随机数作为一个挑战发送到用户端，用户端和运营商端同时采用Ki和128位随机数作为A3和A8算法的输入，通过A3算法，用户得到32位的signed response（SRES），运营商端藉此SRES验证用户身份的有效性；而通过A8算法，用户端和运营商端都获得64位的密钥Kc，两端采用此Kc作为加解密的密钥，实现数据的保密通信

EAP-SIM在GSM单向认证的基础上，提供了双向认证，即服务器端认证客户端，客户端也认证服务器端，只有双向认证通过之后，服务器端才发送EAP-Success消息至客户端，客户端才可以接入网络。同时，EAP-SIM认证机制还通过多次挑战响应机制，生成更强的会话密钥

EAP-SIM 认证流程

1. WLAN_STA -> WLAN_AP : EAPOL-Start						STA端发送EAPOL-Start到AP， 请求EAP认证
2. WLAN_AP -> WLAN_STA : EAP-Request / Identify				AP端发送EAP-Request到STA端， 请求用户标识符
3. WLAN_STA -> WLAN_AP : EAP-Response / Identify				STA端发送EAP-Response， 回应用户标识符
4. WLAN_AP -> WLAN_AS  : Access-Request / EAP-Response / Identify		AP端将EAP-Response报文封装在Radius Access-Request报文中， 发送到AAA服务器
5. WLAN_AS -> WLAN_AP  : Access-Challenge / EAP-Request / SIM /Start		AAA服务器验证用户身份后， 合法后， 向AP发送EAP-Request报文， 封装在Radius Access-Challenge报文中
6. WLAN_AP -> WLAN_STA : EAP-Request / SIM / Start				AP端提取出EAP-Request报文，发送到STA端
7. WLAN_STA -> WLAN_AP : EAP-Response / SIM / Start 				STA端响应 SIM Start， 发送EAP-Response报文到AP端
8. WLAN_AP -> WLAN_AS  : Access-Request / EAP-Response / SIM / Start		AP端将EAP-Response报文封装在Radius Access-Request报文中， 发送到AAA服务器
9. WLAN_AS -> WLAN_AP  : Access-Challenge / EAP-Request / SIM / Challenge	AAA服务器验证STA的响应结果， 发送通过封装到Radius Access-Challenge报文中的EAP-Request， 发送SIM Challenge
10. WLAN_AP -> WLAN_STA : EAP-Request / SIM / Challenge			AP端将EAP-Response报文发送给STA端
11. WLAN_STA -> WLAN_AP : EAP-Response / SIM / Challenge			STA端回应EAP-Response报文给AP端
12. WLAN_AP -> WLAN_AS  : Access-Request / EAP-Response / SIM / Challenge	AP端将EAP-Response报文封装在Radius Access-Request报文中， 发送到AAA服务器
13. WLAN_AS -> WLAN_AP  : Access-Accept / EAP-Success				AAA服务器验证成功后，向AP端发送 EAP-Success 报文，封装在Radius Access-Accept报文中
14. WLAN_AP -> WLAN_STA : EAP-Success					Ap端向STA端发送 EAP-Success 报文

EAP-AKA 认证流程

1. WLAN_UE -> WLAN_AN : EAPOL-Start							STA向AP发起EAP认证请求
2. WLAN_AN -> WLAN_UE : EAP-Request / Identity 					AP向STA发送EAP-Request， 请求获取用户标识
3. WLAN_UE -> WLAN_AN : EAP-Response / Identity					STA向AP发送EAP-Response， 回复自己的用户标识符
4. WLAN_AN -> WLAN_AS : Access-Request / EAP-Message / EAP-Response / Identify   		AP将EAP-Response报文封装到RADIUS Access-Request报文中发送给AAA服务器
5. WLAN_AS -> WLAN_AN : Access-Challenge / EAP-Message / EAP-Request / AKA-Challenge  	AAA服务器生成认证矢量， 放入EAP-Request报文中，然后再封装到Radius Access-Challenge报文中，发送给AP
6. WLAN_AN -> WLAN_UE : EAP-Request / AKA-Challenge  					AP从Radius Access-Challenge报文中提取出 EAP-Request 报文， 发送给STA
7. WLAN_UE -> WLAN_AN : EAP-Response / AKA-Challenge 					STA验证网络的合法性， 并且计算出响应RES， 通过EAP-Response发送给AP
8. WLAN_AN -> WLAN_AS : Access-Request / EAP-Message / EAP-Response / AKA-Challenge（RES） 	AP将EAP-Response报文封装到Radius Access-Request报文中， 转发给AAA服务器
9. WLAN_AS -> WLAN_AN : Access-Accept / EAP-Message / EAP-Success(MSK)   		AAA服务器验证RES成功后， 生成MSK，封装到EAP-Success报文中， 再通过Radius Access-Accept报文发送到 AP
10. WLAN_AN -> WLAN_UE : EAP-Success 						AP 向STA发送 EAP-Success 通知EAP认证成功， STA端通过CK和IK计算出MSK

####7.2 EAP-AKA`

EAP-AKA’是一种新的EAP认证方法，在RFC5448中定义，名为“Improved Extensible Authentication Protocol Method for 3rd Generation Authentication and Key Agreement”。意思为EAP-AKA的改进，对EAP-AKA进行了少量的修订，主要体现在以下两个方面

+ 更新了密钥导出机制，导出的密钥和接入网络名称绑定，即在导出密钥时加入了接入网的名称，这样不同的接入网采用不同的密钥，增强了完整性，同时也扩展了EAP-AKA的适用性，便于各种接入网间的互操作。另外PRF和AT_MAC算法由SHA-1（160bits）更新为SHA-256（256bits），安全性更好 密钥导出算法如下：

	MK = PRF'(IK'|CK',"EAP-AKA'"|Identity)
	K_encr = MK[0..127]
	K_aut  = MK[128..383]
	K_re   = MK[384..639]
	MSK    = MK[640..1151]
	EMSK   = MK[1152..1663]

其中，IK’和CK’由IK和CK导出[33.302]，输入参数包括了IK、CK和接入网名称[24.302]。接入网名称在EAP-AKA’过程中由Server告知Peer

+ RFC5448也对EAP-AKA进行了更新，防止bidding down attacks (回退攻击，指中间人可能操作认证过程，使双方在协商认证算法时使用安全程度较低的算法， 即使认证双方支持安全程度更高的算法)，导致EAP-AKA’回退到EAP-AKA（这里假定EAP-AKA’比EAP-AKA更安全）。实现上，在EAP-AKA中增加了AT-BIDDING的信用，可以使EAP-AKA认证时，Peer和Server可以获知对端的EAP-AKA’能力，从而发现是否出现了EAP-AKA回退

###8. 其它EAP认证方式

####8.1 EAP-MD5

EAP-MD5是另一个IETF开放标准，但提供最少的安全。MD5Hash函数容易受到字典攻击，它被使用在不支持动态WEP的EAP中

####8.2 EAP-LEAP

LEAP（LightWeight Extensible Authentication Protocol）轻量级可扩展认证协议， 是Cisco私有认证协议，2000年底，Cisco为了解决WEP协议中存在的诸多安全漏洞，开发了一套专用的EAP（可扩展身份验证协议）协议，这套协议被称作LEAP（轻量级EAP

从理论上讲，LEAP的弱点从一开始就存在了，因为它本质上就是一个增加了Dynamic Key Rotation（动态密码交替）和Mutual Authentication（互验证）功能的增强版本的EAP-MD5。由于EAP-MD5当初并不是针对一个不受信的无线网络环境设计的，而是针对一个带有一定等级的物理安全屏障的有线网络环境设计的，因此LEAP一开始就不应该被用于无线局域网身份验证任务

LEAP的基础也很薄弱，它本身无法抵御离线的字典攻击。因为它只是单纯的依靠MS-CHAPv2来保护用于无线局域网身份验证的用户证书。在安全领域，MS-CHAPv2的不足是人人皆知，因为它并没有在NT hashes中使用SALT，而是采用了2字节的DES密钥，而且用户名竟然是采用明文形式发送。正因如此，离线的字典攻击或者类似的暴力破解可以采用一个超大的字典库对该系统进行有效的破解。而人们原本以为强度很高的密码，在LEAP系统上完全失去了优势，有些密码几分钟就被破解出来了

一位叫做Joshua Wright的黑客，他开发的专门针对LEAP网络的ASLEAP可以让没有黑客技术的人也能成功破解采用LEAP协议的无线局域网。有鉴于此，Cisco不得不表示，如果企业无法或者不愿意采用高强度的密码策略，那么最好不要采用LEAP
