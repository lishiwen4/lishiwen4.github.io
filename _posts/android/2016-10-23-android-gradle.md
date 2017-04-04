---
layout: post
title: "使用 gradle　构建 android project"
description:
category: android
tags: [android, app]
mathjax: 
chart:
comments: false
---

###1. Gradle

Google　的　Android Studio　基于　Gradle　来构建　project，　Gradle 是以 Groovy 语言为基础，面向Java应用为主，基于DSL语法的自动化构建工具

说到Java的自动化构建工具，　比Ant和Maven强大的多，而且使用起来更加方便简单并且兼容Maven

###2. 安装 Gradle

Android Studio，第一次运行的时候需要自动安装Gradle，这个过程很漫长，而且有可能需要翻墙才下载的了，　最好是手动来安装，　可以前往其官网（[https://gradle.org/gradle-download/](https://gradle.org/gradle-download/)）下载

以　gradle-2.10 为例, 下载　gradle-2.10-bin.zip　后，　使用如下的命令来安装　gradle

    $ sudo unzip gradle-2.10-bin.zip -d /opt/gradle
    $ sudo ln -s /opt/gradle/gradle-2.10/bin/gradle /usr/local/bin/gradle

使用 `gradle -v`命令，　如果显示了版本信息，　则说明安装成功

    $ gradle -v
    Picked up JAVA_TOOL_OPTIONS: -javaagent:/usr/share/java/jayatanaag.jar 

    ------------------------------------------------------------
    Gradle 2.10
    ------------------------------------------------------------

    Build time:   2015-12-21 21:15:04 UTC
    Build number: none
    Revision:     276bdcded730f53aa8c11b479986aafa58e124a6

    Groovy:       2.4.4
    Ant:          Apache Ant(TM) version 1.9.3 compiled on December 23 2013
    JVM:          1.7.0_91 (Oracle Corporation 24.91-b01)
    OS:           Linux 3.19.0-15-generic amd64

###3. 一个最简单的android app项目

build.gradle 文件内容如下:

    buildscript {
        repositories {
            jcenter()
        }

        dependencies {
            classpath 'com.android.tools.build:gradle:1.3.1'
        }
    }

    apply plugin: 'com.android.application'

    android {
        compileSdkVersion 23
        buildToolsVersion "23.1.0"
    }

另外还需要在 local.properties 中定义 android　sdk　的路径

    sdk.dir=/home/sven/Android/Sdk
    
在该项目的根目录中依次执行如下命令，　即可构建出该 android project

    $ gradle clean
    $ gradle build

构建完成后，　会生成 build　目录, build/outputs/apk 中有３个　apk　文件:

+ app-camtest-debug.apk : debug 版，　经 zipalign　处理
+ app-camtest-debug-unaligned.apk : debug 版，　未经 zipalign　处理
+ app-camtest-release-unsigned.apk : release 版，　未经 zipalign　处理

###4. gradle 的构建过程

解析 Gradle 的构建过程之前我们需要理解在 Gradle 中非常重要的两个对象: Project和Task

+ Project : 每个项目的编译至少有一个 Project,一个 build.gradle就代表一个project
+ Task : 每个project里面包含了多个task, task代表着Gradle构建过程中可执行的最小单元,例如当构建一个组件时，可能需要先编译、打包、然后再生成文档或者发布等，这其中的每个步骤都可以定义成一个task
+ Action : Action是一个代码块，里面包含了需要被执行的代码

在编译过程中， Gradle 会根据 build 相关文件，聚合所有的project和task，执行task 中的 action。因为 build.gradle文件中的task非常多，先执行哪个后执行那个需要一种逻辑来保证。这种逻辑就是依赖逻辑，几乎所有的Task 都需要依赖其他 task 来执行，没有被依赖的task 会首先被执行。所以到最后所有的 Task 会构成一个 有向无环图

构建过程分为三个阶段:

1. 初始化阶段：首先会创建一个Project对象，然后执行build.gradle配置这个对象。如果一个工程中有多个module,那么意味着会有多个Project,也就需要多个build.gradle
2. 配置阶段：这个阶段，配置脚本会被执行，执行的过程中，新的task会被创建并且配置给Project对象
3. 执行阶段：在这个阶段，gradle 会根据传入的参数决定如何执行这些task,真正action的执行代码就在这里

在执行gradle命令的时候，是不是经常看到有些任务后面跟着[UP-TO-DATE]? 在Gradle中，每一个task都有inputs和outputs，如果在执行一个Task时，如果它的输入和输出与前一次执行时没有发生变化，那么Gradle便会认为该Task是最新的，因此Gradle将不予执行，这就是增量构建的概念。

一个task的inputs和outputs可以是一个或多个文件，可以是文件夹，还可以是project的某个property，甚至可以是某个闭包所定义的条件。自定义task默认每次执行，但通过指定inputs和outputs，可以达到增量构建的效果

###5. gradle DSL

DSL(Domain Specific Language)，中文意思是特定领域的语言, gradle DSL就是gradle领域的语言

1. gradle script是配置脚本，当脚本被执行的时候，它配置一个特定的对象, 比如说，在android studio工程中，build.gradle被执行的时候，它会配置一个Project对象，settings.gradle被执行时，它配置一个Settings对象
   + Build 类型的　Gradle　script 对应的委托对象是　'Project'
   + Init 类型的　Gradle script 对应的委托对象是　'Gradle'
   + Settings 类型的　Gradle script 对应的委托对象 'Settings'
2. 每一个Gradle script实现了一个Script接口，这意味着Script接口中定义的方法和属性都可以在脚本中使用

###6. build script

build script 通常包含　０到多个　statement 和 script blocks　组成:

+ statement : 可以是　方法调用，　属性分配，　本地变量定义
+ script blocks : 是一个方法，它的参数可以是一个闭包。这个闭包是一个配置闭包，因为当它被执行的时候，它用来配置委托对象

例如，　以一个 android project 为例, 在　其 build.gradle　中

    apply plugin: 'com.android.application'
    
以上就是一条statements,其中apply 是一个方法，后面是它的参数。这行语句之所以比较难理解是因为它使用了缩写，写全应该是这样的:

    project.apply([plugin: 'com.android.application']) 

project调用了apply方法，传入了一个Map作为参数，这个Map的key是plugin,值是com.android.application

仍旧以以一个 android project 的　build.gradle　文件为例, 其中

    dependencies {  
        compile fileTree(dir: 'libs', include: ['*.jar'])  
    }  

是一条script block,　但是因为gradle语法中用了大量的简写，dependencies写完整应该是这样的:

    project.dependencies({  
        add('compile', 'com.android.tools.build:gradle:2.0.', {  
        // Configuration statements  
        })  
    })
    
在 android project　中，　最常用的 statement　就是

1. apply : 构建android project时，　一定需要apply android plugin, 
2. task : 在构建复杂的工程时，　有时可能会需要自己定义　task　来辅助构建过程

而在 android project 中，　有以下的顶层　build script block:

+ allproject {} :   为当前project和所有的子project， 应用 "allproject {}" 块内的配置
+ buildscript {} :  设置 build.gradle 脚本的运行环境
+ configurations {} :   定义一个配置
+ dependencies {} : 定义依赖
+ repositories {} : 设置仓库
+ subprojects {} :  为所有的子project应用 "subprojects {}" 中的配置值
+ android {} :  指定android 项目特有的配置

而在一个 android project 中， "android {}"中可以有如下的 block:

+ defaultConfig{} : 默认配置，是ProductFlavor类型。它共享给其他ProductFlavor使用
+ sourceSets{ } : 源文件目录设置，是AndroidSourceSet类型
+ buildTypes{ } : BuildType类型
+ signingConfigs{ } : 签名配置，SigningConfig类型
+ productFlavors{ } : 产品风格配置，ProductFlavor类型
+ testOptions{ } : 测试配置，TestOptions类型
+ aaptOptions{ } : aapt配置，AaptOptions类型
+ lintOptions{ } : lint配置，LintOptions类型
+ dexOptions{ } : dex配置，DexOptions类型
+ compileOptions{ } : 编译配置，CompileOptions类型
+ packagingOptions{ } : PackagingOptions类型
+ splits{ } : Splits类型

###7. android project　示例

以一个 android app project　camtest为例, 其目录结构为

    camtest
        |
        |- app/                 //android app project
        |   |- src/             
        |   |- build.gradle
        |   
        |- build.gradle         //root project
        |- local.properties
        |- settings.gradle
        |
        |- introview/           // android aar project,　提供自定义view
        |   |-src/
        |   |- build.gradle
        
root project　的　local.properties 中指定了　SDK (或者设置一个环境变量 "ANDROID_HOME"， 值为 sdk 的路径， 2者是等效的):

    sdk.dir=/home/sven/Android/Sdk
    
root project　的　settings.gradle　指定了需要包含的 project (多project构建的项目， 需要该文件来包含子项目):

    include ':app', ':introview'

root project 的　build.gradle　如下:

    buildscript {                                                   //设置build环境
        repositories {
            jcenter()                                               //设置 maven 仓库
        }
        dependencies {
            classpath 'com.android.tools.build:gradle:1.3.0'        //指定 android plugin
        }
    }

    allprojects {                                                   //为当前项目和所有的子项目指定配置
        repositories {
            jcenter()
        }
    }

    task clean(type: Delete) {                                      //定义 clean  task
        delete rootProject.buildDir
    }
    
"app" project 的　build.gradle 如下:

    apply plugin: 'com.android.application'                         //若编译 app 则需要应用 application 插件

    android {
        compileSdkVersion 23                                        // 指定 sdk 版本
        buildToolsVersion "23.0.2"                                  // 指定编译工具版本

        defaultConfig {
            applicationId "com.whut3.sven.aar"                      // 指定 package name， 会覆盖 AndroidManifest 中的设定值
            minSdkVersion 21
            targetSdkVersion 23
            versionCode 1
            versionName "1.0"
        }
        buildTypes {
            release {
                minifyEnabled false
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }

        repositories {
            flatDir {                                               // 指定本地仓库
                dirs 'libs'
            }
        }
    }

    dependencies {
        compile fileTree(dir: 'libs', include: ['*.jar'])           //定义 jar 依赖
        compile(name:'introview-debug', ext:'aar')                  //定义 aar 依赖
        compile 'com.android.support:appcompat-v7:23.1.1'           //定义 appcompat 依赖
    }　

"introview" project　的　 build.gradle　如下:

    apply plugin: 'com.android.library'

    android {
        compileSdkVersion 23
        buildToolsVersion "23.0.2"

        defaultConfig {
            minSdkVersion 21
            targetSdkVersion 23
            versionCode 1
            versionName "1.0"
        }
        buildTypes {
            release {
                minifyEnabled false
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }

    dependencies {
        compile fileTree(dir: 'libs', include: ['*.jar'])
        compile 'com.android.support:appcompat-v7:23.1.1'
    }

###8. android gradle plugin 版本

在基于 Gradle 的 android　project 中, 通常在 root project 中指定 andandroid gradle plugin 版本, 例如在使用 gradle 2.10 时， 指定 android gradle plugin 版本为 “2.0.+”

    buildscript {
        ......
        dependencies {
            classpath 'com.android.tools.build:gradle:2.0.+'
        }
    }

通常新版的 android gradle plugin 需要新版的 gradle， 它们的对应关系如下

+ android gradle plugin 1.0.0 ～ 1.1.3  : 需要 gradle 2.2.1 ～ 2.3 
+ android gradle plugin 1.2.0 ～ 1.3.1  : 需要 gradle 2.2.1 ～ 2.9
+ android gradle plugin 1.5.0  :          需要 gradle 2.2.1+
+ android gradle plugin 2.0.0 ～ 2.1.2  : 需要 gradle 2.10 ～ 2.13
+ android gradle plugin 2.1.3+  :         需要 gradle 2.14.1+

详细的版本对应关系， 可以参照 android studio 官网页面 [https://gradle.org/gradle-download/](https://gradle.org/gradle-download/)

###9. 跨project进行配置

在基于多 project 构建的 gradle 项目里面 (例如 一个 anroid app 项目由 app project 和多个 aar project组成)， 每一个 project 对应一个 build.gradle 文件， 有些设定值在多个project中是相同的， 但是需要在每个 project 的 build.gradle 中重复书写， 为了减少书写量， 可以使用 “allprojects {}” 和 "subprojects {}"

+ allprojects {} : 其中的配置值将会被应用到　当前的 project　和其所有的子　project　中去
+ subprojects {} : 其中的配置值将会被应用到　当前的 project　的其所有的子　project　中去

例如:

    allprojects {
        repositories {
            jcenter()
        }
    }
    
为项目里面的当前 project 和所有子 project 设置 repositories (library 仓库)

为了保证每个项目的独立性，尽量不要在　"allprojects　｛｝"　里面进行太多的设定

当然， 子 project 中定义的配置值是可以覆盖 父 project 中通过 “allprojects {}” 和 "subprojects {}" 传递进来的配置值的

还可以使用 "project() {}" 直接对某一指定的子project进行配置, 例如 :

    project(':myLib') {
        repositories {
		  mavenCentral()
	   }
    }

直接配置了当前的 project 的 子 project "myLib" 的library 仓库

###10. 指定library 仓库

通常， 一个项目可能由多个组件构成， 除了自己开发的自检外， 可能还会依赖其它开发者提供的组件， gradle 支持使用 libray 仓库， 从指定的位置获取依赖的 libray

gradle 中使用 "repositories {}" block 来配置library 仓库， 例如， 在一个 android app 项目中有

    repositories {
            jcenter()       //设置 maven 仓库，　用于获取项目的library / aar
    }
    
如果在 "repositories {}" 中没有正确地定义好 library 仓库， 则 后续的 "dependencies {}" 中定义的依赖就不能正确的解析
    
Gradle 目前支持三种类型的仓库：

1. Maven
2. Ivy
3. Flat directory仓库

在开发 android 项目时， 最常用的就是 Maven 仓库 (用于获取远端的library)和 Flat directory仓库 (用于获取本地的library)

在编译的执行阶段，gradle 将会从仓库中取出对应需要的依赖文件，当然，gradle 本地也会有自己的缓存，不会每次都去取这些依赖

####10.1 Maven　仓库

Apache Maven是Apache开发的一个工具，提供了用于贡献library的文件服务器，　目前只有２个标准的　android Maven　仓库

1. Maven Central : 由sonatype.org维护的Maven仓库, 它是Apache Maven、SBT和其他构建系统的默认仓库，并能很容易被Apache Ant/Ivy、Gradle和其他工具所使用。开源组织例如Apache软件基金会、Eclipse基金会、JBoss和很多个人开源项目都将构件发布到中央仓库
   + 只需要在 "repositories { }"　块中写入 "mavenCentral()"，　即可使用　mavenCentral
2. jcenter : JFrog公司的Bintray网站中的Java仓库，它是当前世界上最大的Java和Android开源软件构件仓库，　所有内容都通过内容分发网络（CDN）使用加密https连接获取，JCenter是Goovy Grape内的默认仓库，Gradle内建支持（jcenter()仓库）
   + 只需要在 "repositories { }"　块中写入 "jcenter()"，　即可使用　jcentral

虽然jcenter和Maven Central 都是标准的 android library仓库，但是它们维护在完全不同的服务器上，由不同的人提供内容，两者之间毫无关系。在jcenter上有的可能 Maven Central 上没有，反之亦然

起初，Android Studio 选择Maven Central作为默认仓库。如果你使用老版本的Android Studio创建一个新项目，mavenCentral()会自动的定义在build.gradle中。

但是Maven Central的最大问题是对开发者不够友好。上传library异常困难。上传上去的开发者都是某种程度的极客。同时还因为诸如安全方面的其他原因，Android Studio团队决定把默认的仓库替换成jcenter。正如你看到的，一旦使用最新版本的Android Studio创建一个项目，jcenter()自动被定义，而不是mavenCentral()

除了这两个标准的　Maven服务器之外，如果我们使用的library的作者是把该library放在自己的服务器上或者本地，我们还可以自己定义特有的Maven仓库服务器，　Twitter的Fabric.io 就是这种情况，它们在https://maven.fabric.io/public上维护了一个自己的Maven仓库。如果你想使用Fabric.io的library，你必须自己如下定义仓库的url :

    repositories {
        maven { url 'https://maven.fabric.io/public' }
    }
    
仓库中的libray，　通常使用如下的字符形式来表示

    GROUP_ID:ARTIFACT_ID:VERSION
    
例如 : “com.android.support:appcompat-v7:23.1.1”
    
在　"dependencies　{}"　中，　添加依赖后， Gradle　会询问Maven仓库服务器这个library是否存在，如果存在，gradle会获得请求library的路径，然后下载这些文件到我们的电脑上，与我们的项目一起编译，　例如:

    dependencies {
        compile 'com.android.support:appcompat-v7:23.1.1'
    }

####10.2　Ivy 仓库

可以通过URL地址或本地文件系统地址，在 gradle　中　使用　远程Ivy仓库或者本地

例如

    repositories {
        ivy {
            url "http://ivy.petrikainulainen.net/repo"
        }
    }

或者

    repositories {
        ivy {       
            url "../ivy-repo"
        }
    }

####10.3 Flat directory仓库

如果想要使用本地的library， 可以配置　flat directory仓库, 例如

    repositories {
        flatDir {
            dirs 'libA', 'libB'
        }
    }    

这意味着系统将在libA 和 libB 目录下搜索依赖, 可以根据需要指定多个搜索目录

    dependencies{
        compile(name:'introview-debug', ext:'aar')
    }

###11. 依赖管理

配置完 library 仓库后， 我们可以声明其中的依赖，如果我们想要声明一个新的依赖，应该使用 "dependencies {}" 块, 例如

    dependencies{
        compile(name:'introview-debug', ext:'aar')
    }
    
在 Gradle 中， 不同的 dependencies 是使用名称来标识的， dependencies 的具体内容则是 configuration 来定义的，一个 configuration 包含： 名称，属性， 并且可以扩展其它的 configuration

例如， 定义一个 dependency ：

    configurations {
        compile
    }
    
许多 gradle 插件都预实现了一些 dependency， 在 apply 插件后， 就可以直接定义这些依赖了， 以 java plugin 为例， 实现了了如下的 dependencies:

+ compile           : 在编译时依赖该依赖项， 例如依赖 jar/aar project等
+ compileOnly       : 只在编译时依赖该依赖项， 运行时不依赖
+ compileClasspath  : 编译时依赖该 classpath
+ runtime           : 在运行时依赖该依赖项
+ testCompile       : 在编译测试代码时依赖该依赖项
+ testCompileOnly   : 只在编译测试代码时依赖该依赖项， 运行时不依赖
+ testCompilePath   : 只在编译测试代码时依赖该依赖项， 运行时不依赖
+ testRuntime       : 在运行测试代码时依赖该依赖项

在android project 中还可以添加如下的依赖

+ androidTestCompile        : 在编译测试应用时依赖该依赖项
+ debugCompile              : 在编译debug版时依赖该项
+ releaseCompile            : 在编译release版时依赖该项
+ &lt;buildtype&gt;Compile  : 在编译指定的build type时依赖该项， 在添加一个buildtype时， 会自动生成该依赖

dependencies 可以按照其依赖的对象不同， 进行分类， 最常见的几种分类如下:

+ External module dependency : 外部模块依赖, 即依赖外部仓库中的 module (例如 library aar plugin 等)
+ Project dependency : project 依赖, 即在基于多project的项目中， 一个project依赖另一个project (例如， 一个 app project 依赖于一个 aar/jar project)
+ File dependency : 文件依赖, 即依赖本地文件系统上的某个文件， 最常见的就是添加本地的aar/jar包依赖了

####11.1 外部模块依赖

外部模块依赖即是指"依赖外部仓库中的 module", 一个外部依赖项可以使用如下的字串来指定

    group:name:version
    
其中:

+ group属性 : 指定依赖的分组（在Maven中，就是groupId）。
+ name属性 : 指定依赖的名称（在Maven中，就是artifactId）。
+ version属性 : 指定外部依赖的版本（在Maven中，就是version）

这些属性在Maven仓库中是必须的，如果你使用其他仓库，一些属性可能是可选的

例如， 在 android app 开发时， 通常依赖 appcompact v7， 若要使用 maven 仓库中的 appcompact v7， 则可以使用如下的方式:

    dependencies {
        compile 'com.android.support:appcompat-v7:23.1.1'
    }
    
又例如， 在编译android app对应的测试应用时，通常依赖 Junit4, 若要使用  maven 仓库中的 Junit4， 则可以使用如下的方式:

    dependencies {
        testCompile 'junit:junit:4.12'
    }    

如果要保证我们依赖的库始终处于最新的状态， 我们可以通过添加通配符“+”的方式， 比如:

   dependencies {
        compile 'com.android.support:appcompat-v7:23.1+'
    }

但是一般不要这么做，因为每次编译都要去做网络请求查看是否有新版本导致编译过慢

####11.2 project 依赖

project 依赖, 是指基于多project的项目中， 一个project依赖另一个project， 例如， 若在一个 android app项目里面， 有一部分功能被组织成一个名为 introview 的 project， 先被编译成一个 aar， 然后 app project依赖这一 aar project， 则可以使用如下的方式定义依赖

    dependencies {
        compile project(':introview')
    }
    
可见， 在指定project依赖时， 只需要指定 "group:name:version" 中的 name 属性即可
    
####11.3 本地文件依赖

件依赖, 即依赖本地文件系统上的某个文件， 最常见的就是添加本地的aar/jar包依赖了

例如， 若某个android app project在编译时， 依赖一个放置在 libaar 目录中的 introview.aar 文件， 则可使用如下的方式指定依赖

    repositories {
        flatDir {
            dirs 'libaar'
        }
    }

    dependencies {
        compile(name:'introview-debug', ext:'aar')
    }

若需要依赖某个目录下的单个jar文件， 则可以使用如下的方式来定义依赖

    dependencies {
        compile files('libs/xxxx.jar')
    }
    
若需要依赖某个目录下的所有jar文件，则可以使用如下的方式来定义依赖

    dependencies { hdhshdhshuhjsdhahjhsjahdhdhuehduhehghghhgfdsasddfghkkkk
        compile fileTree(dir: 'libs', include: ['*.jar'])
    }

###12. 应用插件

使用 `apply xxx` 来应用 gradle 插件

例如，　在 android app module 中:

    apply plugin: 'com.android.application'
    
在　android aar module 中则是:

    apply plugin: 'com.android.library'

在一个 project 里面 (即同一个 build.gradle里面) 同时应用 "com.android.application" 和 "java" 插件将导致编译 error

###13. Build Type

gradle 的 android  plugin默认会定义2种 build type: debug 和 release

这2种build type 的默认配置如下

+ debug :
   + debuggable:                true
   + jniDebugBuild:             false
   + renderscriptDebugBuild:    false
   + renderscriptOptimLevel:    3
   + applicationIdSuffix:       null
   + versionNameSuffix:         null
   + signingConfig:             android.signingConfigs.debug
   + zipAlign:                  false
   + runProguard:               fasle
   + proguardFile:              N/A
   + proguardFiles:             N/A
+ release :
   + debuggable:                false
   + jniDebugBuild:             false
   + renderscriptDebugBuild:    false
   + renderscriptOptimLevel:    3
   + applicationIdSuffix:       null
   + versionNameSuffix:         null
   + signingConfig:             null
   + zipAlign:                  true
   + runProguard:               fasle
   + proguardFile:              N/A
   + proguardFiles:             N/A
   
可以在 build.gradle 中修改这两种 build type 的参数

    android {
        ......
        buildTypes {
            debug{
                packageNameSuffix "-debug"
            }
        }
        ......
    }

如有需要， 还可以自己定义新的 build type 来辅助构建过程, 例如， 定义一个名为 “mybuild” 的 build type

    android {
        ......
        buildTypes {
            mybuild {
                ......
            }
        }
        ......
    }

如果在定义 build type 时， 需要让其继承一个已有的 build type, 例如， 继承 名为 “debug” 的 build type , 则可以使用 "initWith" :

    android {
        buildTypes {
            mybuild.initWith(buildTypes.debug)
            mybuild {
                ......
            }
        }
    }

对于每一个 build type (无论是 android plugin 预定义， 还是用户自定的), gradle 会为其生成如下的配置:

+ 创建task: 创建一个名为 assemble<build type name> 的 task, assemble task 也会添加对 这一 task 的依赖
   + 例如名为 “mybuild” 的build type， 对应的task是 "assembleMybuild",
   + assembleDebug 这一 task 正是对应了 android plugin 定义的 "Debug" 这一 build type
   + assembleRelease 这一 task 正是对应了 android plugin 定义的 "Release" 这一 build type
+ 创建新的 sourceSet : 针对每一个Build Type，一个新的对应的sourceSet会被创建，这个sourceSet使用一个默认的路径 src/<build type name>*
   + 这就意味着Build Type的名字不能是 main 或者 androidTest， 这2个名字已经被 android plugin 所使用
+ 定义一个新的依赖 : 生成一个名为 <build type name>Compile 的依赖项， 可以使用该依赖项来定义该 build type 所独有的依赖
   + 例如名为 “mybuild” 的build type， 生成名为　“mybuiildCompile” 的依赖，　在　"denpendencies {}“　中可以直接使用该依赖
   
###14. build variant

在开发中我们可能会需要在编译的时候动态根据当前的编译类型输出不同样式的apk文件， 这个时候就需要利用 build variant 了

build variant 可以分为2类

+ build config
+ product flavors

####14.1 build config

在 build.gradle 脚本中可以定义 BuildConfig 字段， 然后在 project 的源码里面可以获取这些值 (利用这一特性， 可以在不同的 build type 或者 product flavors 里面定义不同值的字段， 在代码里面根据这些字段做灵活处理)

使用 buildConfigField 关键字， 即可在 build.gradle 里面定义 BuildConfig 字段， 例如

    buildTypes {
        release {
            buildConfigField "String", "APP_ID", "\"releaseId\""    
            buildConfigField "boolean", "LOG_DEBUG", "false"
        }

        debug {
            buildConfigField "String", "APP_ID", "\"debugId\""
            buildConfigField "boolean", "LOG_DEBUG", "true"
        }
    }
    
在 project 的源码中， 可以直接使用如下的方式来引用上述的 BuildConfig 字段

    BuildConfig.APP_ID
    BuildConfig.LOG_DEBUG
    
android plugin 预先定义了如下的 BuildConfig 字段， 可以直接使用

    public final class BuildConfig {
        public static final boolean DEBUG;          // ”buildTypes {}“ 中配置的 “debuggable” 的值
        public static final String APPLICATION_ID;  // android package name
        public static final String BUILD_TYPE;      // 当前的 build type 的名称
        public static final String FLAVOR;          // 当前的 product flavor的名称
        public static final int VERSION_CODE;       // 当前的版本号数值
        public static final String VERSION_NAME;    // 当前的版本号显示值
    }

####14.2 product flavors

前面讲到的配置， 都是针对同一份源码编译不同的版本， 如果需要针对不同的源码来编译的话 (例如收费版和免费版使用不同的源码结构), 则需要使用 product flavors 

product flavors 和 build type 的差异是 :

+ product flavors 和 build type 是2个不同的配置维度
   + product flavor 和 defaultConfig 共享所有配置条目 （所以 product flavors 中可以修改 package name 等信息）
   + build type 有自己的配置条目
+ 二者都会自动生成 sourceSet
   + 自动生成的 sourceSet 参见 soureSet 章节
   + 通常是在 不同的 product flavor 中定义差异化代码， build type 中配置不同的编译选项
+ product flavors 和 build type 还可以配合使用， 针对不同的源码， 编译不同的版本
   
例如， 在自定义的 product flavors 中修改 package name:

    android {
        productFlavors {
            free {
                applicationId "com.whut3.sven.free"
            }
        
            charge {
                applicationId "com.whut3.sven.charge"
            }
        }
    }

对于添加的 product flavors， Gradle 会自动生成如下的 task

+ assemble&lt;product flavors name&gt;
+ compile&lt;product flavors name&gt;&lt;build type name&gt;AndroidTestSources
+ compile&lt;product flavors name&gt;&lt;build type name&gt;Sources
+ compile&lt;product flavors name&gt;&lt;build type name&gt;AndroidTestSources
+ compile&lt;product flavors name&gt;DebugAndroidTestSources
+ compile&lt;product flavors name&gt;DebugSources
+ compile&lt;product flavors name&gt;DebugSources
+ lint&lt;product flavors name&gt;&lt;build type name&gt;
+ test&lt;product flavors name&gt;&lt;build type name&gt;UnitTest

例如， 在一个 自定义了 build type "mybuild" 和 product flavors "free" 的 project 里面， gradle会自动生成如下的task

+ assembleFree
+ compileFreeDebugAndroidTestSources
+ compileFreeDebugSources
+ compileFreeDebugUnitTestSources
+ compileFreeReleaseAndroidTestSources
+ compileFreeReleaseSources
+ compileFreeReleaseUnitTestSources
+ compileFreeMybuildAndroidTestSources
+ compileFreeMybuildSources
+ compileFreeMybuildUnitTestSources
+ lintFreeDebug
+ lintFreeRelease
+ lintFreeMybuild
+ testFreeDebugUnitTest
+ testFreeReleaseUnitTest
+ testFreeMybuildUnitTest
   
###15.　Source Set

gradle 的　java plugin 引入了一个　”source set“　的概念，　”source set“　指一组将要被一起编译和执行此的源码文件, ”source set“ 最常见的一个用场景就是按照逻辑上的功能将源码分组(例如正式代码分为一组，　将单元测试代码分为一组)

android gradle project 中源码文件是指：

+ Manifest
+ java
+ android res 资源文件
+ assets 资源文件
+ RenderScript 文件
+ aidl
+ jni
+ jnilib
+ java resource 资源文件

在不同的 gradle plugin 插件中还可以添加不同的源码文件类型定义

一个 ”source set“ 和一个 compile classpath 以及 runtime path　相关联

####15.1 android plugin 的标准 sourceSet

java plugin 定义了２个标准的 source set :

+ main : 包含产品的源码, 编译进　jar 文件
+ test : 包含测试所用的代码，　编译后使用 JUnit　运行

而　android plugin 在此基础上，　还定义了另一个 source set

+ androidTest : 在一个 android project　中， test　中通常是单元测试代码，　而 androidTest　中则是 apk 的整体测试代码 (单元测试代码在编译的pc上执行， 而apk测试代码则是在android设备上执行)

**在　Android Studio　中生成的 android project　的 src　目录下，　有３个目录 : androidTest, main, test, 正是对应了这３个 source set**

java plugin 假设基于 gradle　的　project　的源码文件结构如下:

    src/main/java               //项目的java 源码
    src/main/resources          //项目的资源文件
    src/test/java               //测试所用的 java 代码
    src/test/resources          //测试所用的资源
    src/sourceSet/java          //Java source for the given source set
    src/sourceSet/resources 		Resources for the given source set
    
但是对于　android plaugin　来说　还存在其它类型的文件，　例如　Androidmanifest.xml, assets, aidl, jni 等，　因此， android project　的默认的目录结构如下

    src/
    ├── androidTest
    │   ├── AndroidManifest.xml     //这一文件不需要，　会自动生成
    │   ├── java
    │   ├── res
    │   ├── assets
    │   ├── aidl
    │   ├── rs 
    │   ├── jni
    │   ├── jniLibs
    │   └── resources
    │       
    ├── main
    │   ├── AndroidManifest.xml
    │   ├── java
    │   ├── res
    │   ├── assets
    │   ├── aidl
    │   ├── rs 
    │   ├── jni
    │   ├── jniLibs
    │   └── resources
    │
    └── test
        ├── java    
        └── resources
        
** 注意 jar / aar 文件的位置不由 source set来指定， 而是使用本地仓库的方式来指定**

####15.2 查看 SourceSet 配置

执行 `gradle sourceSets` 命令可以列出 gradle 项目中的 Source Set 配置

例如:

    $ gradle sourceSets
    
    androidTest
    -----------
    Compile configuration: androidTestCompile
    ......
    Java-style resources: [src/androidTest/resources]


    debug
    -----
    Compile configuration: debugCompile
    build.gradle name: android.sourceSets.debug
    Java sources: [src/debug/java]
    Manifest file: src/debug/AndroidManifest.xml
    Android resources: [src/debug/res]
    Assets: [src/debug/assets]
    AIDL sources: [src/debug/aidl]
    RenderScript sources: [src/debug/rs]
    JNI sources: [src/debug/jni]
    JNI libraries: [src/debug/jniLibs]
    Java-style resources: [src/debug/resources]


    main
    ----
    Compile configuration: compile
    build.gradle name: android.sourceSets.main
    ......
    Java-style resources: [src/main/resources]
    
    
    release
    -------
    Compile configuration: releaseCompile
    build.gradle name: android.sourceSets.release
    ......
    Java-style resources: [src/release/resources]    

####15.3 自动生成的 sourceSet

在 android gradle 项目中， 定义新的 build type 或者 product flavors (后续有章节介绍)时， 都会自动生成一个同名的 sourceSet (build type 和 product flavor 共享同一命名空间，且名称不能为 "main", "test", "androidTest")

例如， 参照上一小节 "查看 SourceSet 配置", 在 一个 android project中， 会自动为预定义的 build type "debug" 和 “release” 生成对应的SourceSet

    debug
    -----
    Compile configuration: debugCompile
    build.gradle name: android.sourceSets.debug
    ......
    Java-style resources: [src/debug/resources]
    
    release
    -------
    Compile configuration: releaseCompile
    build.gradle name: android.sourceSets.release
    ......
    Java-style resources: [src/release/resources] 

build type 和 product flavor 是2个不同的配置维度， 因此可以进行排列组合， 所以， 为 build type 或者 product flavors 自动生成的SourceSet时，也可以排列组合, 生成多个 sourceSet 它们的默认配置如下:

    <build type name>
    ---
    Compile configuration: <build type name>Compile
    build.gradle name: android.sourceSets.<build type name>
    Java sources: [src/<build type name>/java]
    Manifest file: src/<build type name>/AndroidManifest.xml
    Android resources: [src/<build type name>/res]
    Assets: [src/<build type name>/assets]
    AIDL sources: [src/<build type name>/aidl]
    RenderScript sources: [src/<build type name>/rs]
    JNI sources: [src/<build type name>/jni]
    JNI libraries: [src/<build type name>/jniLibs]
    Java-style resources: [src/<build type name>/resources]

    <product flavor name>
    ---
    Compile configuration: <product flavor name>Compile
    build.gradle name: android.sourceSets.<product flavor name>
    Java sources: [src/<product flavor name>/java]
    Manifest file: src/<product flavor name>/AndroidManifest.xml
    Android resources: [src/<product flavor name>/res]
    Assets: [src/<product flavor name>/assets]
    AIDL sources: [src/<product flavor name>/aidl]
    RenderScript sources: [src/<product flavor name>/rs]
    JNI sources: [src/<product flavor name>/jni]
    JNI libraries: [src/<product flavor name>/jniLibs]
    Java-style resources: [src/<product flavor name>/resources]
 
    <product flavor name><build type name>
    ---
    Compile configuration: <product flavor name><build type name>Compile
    build.gradle name: android.sourceSets.<product flavor name><build type name>
    Java sources: [src/<product flavor name><build type name>/java]
    Manifest file: src/<product flavor name><build type name>/AndroidManifest.xml
    Android resources: [src/<product flavor name><build type name>/res]
    Assets: [src/<product flavor name><build type name>/assets]
    AIDL sources: [src/<product flavor name><build type name>/aidl]
    RenderScript sources: [src/<product flavor name><build type name>/rs]
    JNI sources: [src/<product flavor name><build type name>/jni]
    JNI libraries: [src/<product flavor name><build type name>/jniLibs]
    Java-style resources: [src/<product flavor name><build type name>/resources]
    
例如， 若自定义了 build type "mybuildtype"， 自定义了 product flavors "myflavor", 加上 android gradle plugin 预定义的 build type “debug” 和 “release”， 该 project 中会组合生成如下的 sourceSet

+ debug
+ release
+ mybuildtype
+ myflavor
+ myflavorDebug
+ myflavorRelease
+ myflavorMybuildtype

####15.4 source merge

在使用 sourceSet 并非是非此即彼的， 而是存在一个 merge 的过程， 以project的项目的主代码为例(不讨论 test 和 androidTest)， 所有为 build tye 和 product flavor生成的 sourceSet, 都会和名为 "mian" 的 sourceSet 进行合并

若同一资源， 在不同的 Source Set 中有不同的定义， 那么这个时候涉及到一个资源的优先级, 其高低顺序为:

    Build Type  >   Product Flavor  >   Main    >   Dependencies
    
在Build Type中定义的资源优先级最大，在Library 中定义的资源优先级最低, **在多project 构建的 Gradle 项目中Manifest 文件的合并， 也遵循此优先级规则**

仍假设 一个 名为 “sstest” 的 project 中有如下的 build type:

+ debug
+ relase
+ mybuildtype

有如下的 product flavor

+ myflavor

则 执行 `gradle assemble` 后， 生成多个 apk 文件， 它们对应的 sourceSet 如下:

+ sstest-myflavor-debug-unaligned.apk ： 和 "main" 合并 时， sourceSet 优先级的次序从高到低依次为
   + sourceSet "myflavorDebug"
   + sourceSet "debug"
   + sourceSet "myflavor"
+ sstest-myflavor-mybuildtype-unaligned.apk ： 和 "main" 合并 时， sourceSet 优先级的次序从高到低依次为
   + sourceSet "myflavorMybuildtype"
   + sourceSet "mybuildtype"
   + sourceSet "myflavor"         
+ sstest-myflavor-release-unaligned.apk : 和 "main" 合并 时， sourceSet 优先级的次序从高到低依次为
   + sourceSet "myflavorRelease"
   + sourceSet "release"
   + sourceSet "myflavor"
   
**需要注意的是， 在 sourceSet merge 的过程中， 可以在多个 sourceSet 中存在相同的资源文件， 但是不能存在相同java class， 否则会报 “duplicate class”, 因此， 一个java类， 要么存在于 “main” sourceSet 中， 要么存在于某个 product flavor sourceSet 中**
   
####15.5 修改 sourceSet

在 build.gradle　中的　"android {}"　中可以根据需要使用 "sourceSets　{}"　来单独修改某一类文件的目录，　例如，　指定　sourceSet ”main“ 的结构

    android {
        sourceSets {
            main {
                manifest.srcFile 'AndroidManifest.xml'
                java.srcDirs = ['src']
                resources.srcDirs = ['src']
                aidl.srcDirs = ['src']
                renderscript.srcDirs = ['src']
                res.srcDirs = ['res']
                assets.srcDirs = ['assets']
            }
        }
    }


或者也可以直接指定　source set 的　root　目录，例如, 将 "androidTest"　这一 source set 的默认目录从　"src/androidTest/*" 更换为 ”tests/*“

    android {
        sourceSets {
            androidTest.setRoot('tests')
        }
    }

###16. 配置 manifest

android gradle plugin 可以在 build.gradle 中的 "android {}" 块中使用 "defaultConfig {}"配置 manifest 条目, 例如:

    android {
        compileSdkVersion 23
        buildToolsVersion "23.0.2"
    
        defaultConfig {
            applicationId "com.whut3.sven.aar"
            minSdkVersion 21
            targetSdkVersion 23
            versionCode 1
            versionName "1.0"
        }
        ......
    }

defaultConfig {}" 中的配置条目， 如果不指定值的话， 在 DSL 脚本 (build.gradle) 中有着默认值，  而如果在构建脚本(build.gradle)中使用自定义代码来获取这些配置值的话(以 aandroid.defaultConfig.xxx 的形式引用)， 也会有默认值的，但这二者的默认值是不同的, 所有的配置条目以及其默认值如下:

+ versionCode
   + DSL 脚本中的默认值 : -1
   + 自定义代码中访问时的默认值 : value from manifest if present
+ versionName
   + DSL 脚本中的默认值 null
   + 自定义代码中访问时的默认值 : value from manifest if present
+ minSdkVersion
   + DSL 脚本中的默认值 -1
   + 自定义代码中访问时的默认值 : value from manifest if present
+ targetSdkVersion
   + DSL 脚本中的默认值 -1
   + 自定义代码中访问时的默认值 : value from manifest if present
+ applicationId
   + DSL 脚本中的默认值 null
   + 自定义代码中访问时的默认值 : value from manifest if present
+ testApplicationId
   + DSL 脚本中的默认值 null
   + 自定义代码中访问时的默认值 : app package name + “.test”
+ testInstrumentationRunner
   + DSL 脚本中的默认值 null
   + 自定义代码中访问时的默认值 : android.test.InstrumentationTestRunner
+ signingConfig
   + DSL 脚本中的默认值 null
   + 自定义代码中访问时的默认值 : null
+ runProguard
   + DSL 脚本中的默认值 false
   + 自定义代码中访问时的默认值 : false
+ proguardFile
   + DSL 脚本中的默认值 N/A
   + 自定义代码中访问时的默认值 : N/A


###17. Sign Config

android plugin 定义的 2 个build type：

+ debug : 默认使用 debug.keystore 来签名
+ release : 默认不签名

如果需要定义签名新的签名配置， 则可以使用 "signingConfigs {}" 

    android {
        signingConfigs {        
            mySign  {
                storeFile file("release.keystore")
                storePassword "storepswd"
                keyAlias "releasekey"
                keyPassword "keypswd"
            }
            ......
        }
        ......
    }

在 不同的 build type 中， 可以根据需要来使用不同的签名配置, 例如:

    android {
        signingConfigs {
            mySign {
                ......
            }
            ......        
        }
        
        buildTypes {
            mybuild {
                signingConfig signingConfigs.mySign
                ......    
            }
        }
        ......
    }

debug 这一 build type 使用的 signingConfigs 为预定义的名称为 "debug" signingConfigs

###18. proguard混淆

一般release发布版本是需要启用混淆的，这样别人反编译之后就很难分析你的代码，而我们自己开发调试的时候是不需要混淆的，所以debug默认不启用混淆， 可以根据实际需要来配置不同的 build type 的 proguard混淆

例如， 对自定义的 build type 启用混淆， 配置如下：

    android {
        buildTypes {
            mybuild {
                minifyEnables true
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
            ......
        }
        ......
    }

示例中 proguard-android.tx 和 proguard-rules.pro  放置在 project 的根目录中， 若放置在其它的目录中，在而需要指定详细的路径

**proguard 还可以去除 app 中没有使用到的类，** 比如你用了lib，lib里有很多很多类，你可能就用到了一个类， 则开启 proguard 后将不会包含 lib 中未使用到的类

###19. Shrink Resource

在编译的时候，我们可能会有很多资源并没有用到，此时就可以通过shrinkResources来优化我们的资源文件，除去那些不必要的资源

在构建时，自动移除无用资源的功能能够大幅度减小APK文件的大小（最高可减小34%）；当前能够移除的无用资源包括图片、布局、菜单等资源文件，但不包括value资源文件


    android {
        buildTypesa {
            release {
                minifyEnabled = true
                shrinkResources = true
            }
        }
    }

resource shrink 功能的使用依赖于code shrinking， 所以minifyEnabled也必须打开

如果我们需要查看该命令帮我们减少了多少无用的资源，我们也可以通过运行shrinkReleaseResources命令来查看log


对一些特殊的文件或者文件夹，比如 国际化的资源文件、屏幕适配资源，如果我们已经确定了某种型号，而不需要重新适配，我们可以直接去掉不可能会被适配的资源。这在为厂商适配机型定制app的时候是很用的。做法如下：
比如我们可能有非常多的国际化的资源，如果我们应用场景只用到了English,Danish,Dutch的资源，我们可以直接指定我们的resConfig

    <?xml version=“1.0” encoding="utf-8"?>
    <resources xmlns:tools="http://schemas.android.com/tools"
        tools:keep="@layout/keep_me, @layout/also_used_*"/>
        
对于尺寸文件也可以这样做

    android {
        defaultConfig {
            resConfigs "hdpi", "xhdpi", "xxhdpi", "xxxhdpi"
        }
    }

###20. Multi Dex

随着项目的一天天变大，慢慢的都会遇到单个dex最多65535个方法数的瓶颈，如果是ANT构建的项目就会比较麻烦，但是Gradle已经帮我们处理好了，而添加的方法也很简单，总共就分三步 : 

1. 首先在defaultConfig节点使能多DEX功能

    android {
        defaultConfig {
            multiDexEnabled true
        }
    }

2. 然后引入multidex库文件

    dependencies {
        compile 'com.android.support:multidex:1.0.0'
    }
    
3. 最后AppApplication继承一下MultiDexApplication即可

关于dex超过 64k 的限制， 解决方案还可以参考  [Android拆分与加载Dex的多种方案对比](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207151651&idx=1&sn=9eab282711f4eb2b4daf2fbae5a5ca9a&3rd=MzA3MDU4NTYzMw==&scene=6#rd)

###21. task

Gradle中有两个基本的概念：project和task, 每个Gradle的构建由一个或者多个project构成，它代表着需要被构建的组件或者构建的整个项目。每个project由一个或者多个task组成。task代表着Gradle构建过程中可执行的最小单元。例如当构建一个组件时，可能需要先编译、打包、然后再生成文档或者发布等，这其中的每个步骤都可以定义成一个task　(当我们在Android Studio点击 build,rebuild,clean菜单的时候，执行的就是一些gradle task)

Gradle project 中的Task要么是由不同的Plugin引入的，要么是我们自己在build.gradle文件中直接创建的, 将一个plugin运用到build file中时，会自动创建一系列的构建任务（build task）去运行, 无论是Java plugin　还是　Android Plugin　都会如此, gradle约定的任务有以下４个:　
    
+ assemble : 生成 project　的　output　输出
+ check : 运行所有的检查
+ build : 同时执行　assemble　和　check
+ clean : 清理　project　的　outpust　输出

**assemble check build　这３个 task　实际上并不做任何动作，　它们只是提供给　Gradle　插件的　‘锚点’，用于为实际执行动作的　task　添加依赖**, 这样做的好处是，不管你在什么类型的项目中，你都可以调用相同的 task, 例如, apply fingbug　这一插间后，　将创建一个新的task，　并且让 ‘check’　依赖它，　这样，　无论何时执行　‘check’ , 新建的task都将被执行

可以使用如下的命令，　来查看当前的　project　中定义的 task

    $ gradle tasks
    
####21.1 java　插件创建的 task

java plugin　创建的task中，　最重要的２个task分别锚挂到如下的任务中 :

1. assemble
   + jar : 编译java源码生成输出
2. check
   + test :　运行单元测试
   
通常java项目中的任务只会用到assemble和check这两个

####21.2 android 插件创建的　task

android 插件创建的的task比通用的四大task多了“connectedCheck”和“deviceCheck”，这是想要让项目忽视设备是否连接，正常执行check任务

+ assemble : 汇集所有项目输出
+ check : 运行所有校验
+ connectedCheck : 运行所有需要链接设备或模拟器的校验
+ deviceCheck : 运行调用远程设备的校验
+ build : 同时运行assemble　和　check
+ clean : 清除所有的项目输出
    
一个安卓的项目至少有两个输出，一是debug apk，二是release apk, 这两个输出都有自己对应的锚任务，来实现它们各自的构建, 调用assemble任务时会同时调用assembleDebug 和 assembleRelease来保证有两个输出

+ assemble
   + assembleDebug
   + assembleRelease

为了能够安装卸载，android plugin 为所有的build类型（debug,release,test）都创建了install/uninstall 任务

+ installDebug
+ installRelease
+ uninstallAll
   + uninstallDebug
   + uninstallRelease
   + uninstallDebugAndroidTest

事实上，　android plugin 创建的　task　远远比上述列出的要多,　可以在一个 android project　中运行 `gradle tasks`　来查看所有定义的 task 

####21.3　自定义 task

可以在 build.gradle　中声明一个 task，其语法为:

    task myTask
    task myTask { configure closure }
    task myType << { task action }
    task myTask(type: SomeType)
    task myTask(type: SomeType) { configure closure }

例如

    task mytask１ {  
        println "my task 1"  
    }
    
也可以像这样定义一个 task

    task mytask2 << {  
        println "my task 2"  
    }
    
    等价于
    
    task mytask2 {
        doLast {
            println "my task 2"  
        }
    }
    
但是这２种方式定义的　task　有着巨大的差别:

+ mytask1 : 这种方式和在　gradle.build　中直接书写 action　语句的效果相同, 其在整个构建的过程的　"初始化阶段"　就会被执行, 即无论执行哪一 task，　mytask1　都会执行
+ mytask2 : 这种方式定义的　task　在整个构建过程的　"执行阶段" 才会被执行

默认情况下，我们所创建的Task是DefaultTask类型，该类型是一个非常通用的Task类型, 另外还有许多其它类型的　task，　可以参考　[https://docs.gradle.org/current/dsl/](https://docs.gradle.org/current/dsl/)

例如，　在 android project　的　build.gradle　中有

    task clean(type: Delete) {
        delete rootProject.buildDir
    }
    
####21.4 task　依赖

当构建一个复杂的项目时，不同task之间存在依赖是必然的。比如说，如果想运行'部署'的task，必然要先运行 编译、打包、检测服务器等task，只有当这被些被依赖的task执行完成后，才会部署

task 依赖有２种:

+ mustRunAfter : 定义task必须在指定的task后运行
+ dependsOn :　定义task依赖的task, 在执行　task　前，　会先执行其依赖的 task

定义　mustRunAfter　依赖示例:

    task mytask2 {
        ......Android Gradle 完整指南
    }

    mytask2.mustRunAfter task1

定义　mustRunAfter　依赖示例:

    task mytask2(dependsOn: mytask1) {
        ......
    }
    
    或者使用如下的等价形式
    
    mytask2.dependsOn mytask1

若 task 依赖多个　task，　则可以使用　"[]"　例如

    task mytask５(dependsOn:['xx', 'xx', 'xx']) << {
        ......
    }

###22. 高级构建选项

####22.1 compileOptions

进行 Java 的版本配置，以便使用对应版本的特性， 其中配置项如下:

+ sourceCompatibility : 指定语言级别
+ targetCompatibility : 指定生成的 java 字节码的版本

例如

    android {
        compileOptions {
            sourceCompatibility = "1.6"
            targetCompatibility = "1.6"
        }
    }
    
默认值是“1.6”。这个设置将影响所有task编译Java源代码

####22.2 aaptOptions

设置 aapt 属性， 

+ failOnMissingConfigEntry ： 当configuration中某一 entry 找不到时， 强制 aapt 返回 fail
+ ignoreAssets ： 匹配模式， 指定要被忽略的 assets
+ noCompress ： 指定最终的 apk 文件中， 不要压缩存储的文件

例如:

    android {
        aaptOptions {
            noCompress 'foo', 'bar'
            ignoreAssetsPattern "!.svn:!.git:!.ds_store:!*.scc:.*:<dir>_*:!CVS:!thumbs.db:!picasa.ini:!*~"
        }
    }

####22.3 dexOptions

dexOptins 用于配置 pre odex 参数， 可用的配置项如下：

+ incremental : 是否 enable dx 的 incremental 模式， 这一模式限制很多， 有可能不能工作， 需要谨慎使用
+ javaMaxHeapSize : 指定调用 dx 时 的 “ -JXmx*” 值， 值应该使用类似于 “1024M” 这样的样式
+ jumboMode : enable dx 的 jumbo mode （即传递 '--force-jumbo' 参数）
+ preDexLibraries : 是否 pre-dex libraries， 这可以加快增量构建， 但是会减慢 clean build

例如 
    android {
        dexOptions {
            incremental false

            preDexLibraries = false

            jumboMode = false

        }
    }
    
####22.4 lintOptions

Android Lint是SDK Tools 16 (ADT 16)之后才引入的工具，通过它对Android工程源代码进行扫描和检查，可发现潜在的问题，以便程序员及早修正这个问题

lintOptions 用于设置Lint的属性， 可选的配置项如下:

+ quiet : 是否关闭lint报告的分析进度
+ abortOnError : 错误发生后是否停止gradle构建
+ ignoreWarnings : 是否只报告error
+ absolutePaths : 是否忽略有错误的文件的全/绝对路径(默认是true) 
+ checkAllWarnings : 是否检查所有问题点，包含其他默认关闭项
+ warningsAsErrors : 是否将所有warning当做error
+ disable 'TypographyFractions','TypographyQuotes' : 关闭指定问题检查
+ enable 'RtlHardcoded','RtlCompat', 'RtlEnabled' : 打开指定问题检查
+ check 'NewApi', 'InlinedApi' : 仅检查指定问题
+ noLines : error输出文件是否不包含源码行号
+ showAll ： 是否在显示错误的所有发生位置时不截取
+ lintConfig file("default-lint.xml") ： 回退lint设置时的默认规则文件
+ textReport : 是否生成txt格式报告
+ textOutput ： 重定向输出， 可以是文件或'stdout'
+ xmlReport ： 是否生成XML格式报告
+ xmlOutput file("lint-report.xml") ： 指定xml报告文档(默认是lint-results.xml)
+ htmlReport ： 是否生成HTML格式的报告(带问题解释，源码位置，等)
+ htmlOutput file("lint-report.html") : 生成的html报告的路径(默认是lint-results.html )
+ checkReleaseBuilds : 是否为所有正式版构建执行规则生成崩溃的lint检查，如果有崩溃问题将停止构建
+ fatal 'NewApi', 'InlineApi' : 在发布版本编译时检查(即使不包含lint目标)，对指定问题的规则生成崩溃
+ error 'Wakelock', 'TextViewEdits' : 对指定问题的规则生成错误 
+ warning 'ResourceAsColor' ： 对指定问题的规则生成警告
+ ignore 'TypographyQuotes' ： 忽略指定问题的规则(同关闭检查)

例如， 用gradle build命令时，经常由于lint错误终止构建， 而这些错误又经常是第三方库中的， 可以关闭lint检查， 跳过这些错误， 继续构建

    android {
        lintOptions {
            abortOnError false
        }
    }

####22.5 splits

“splits{}” 用于设置如何拆分APK, 将apk以 abi 或者 density 进行分包， 比如你想拆分成arm版和x86版， 再也不用为了缩小包的体积而专门去只留下一个arm的jni文件夹了

“splits{}” 可选的配置项如下:

+ abi : 设置abi
+ abiFilters : abi filter 列表， 用于 multi-apk
+ density : 设置density
+ densityFilters : density filter 列表， 用于 multi-apk

例如 :

    splits {
        abi {
            enable true
            reset()
            include 'armeabi', 'x86' //, 'x86', 'armeabi-v7a', 'mips'
            universalApk false
        }
    }

###23. 设置代理

gradle build 过程中， 大量的依赖库需要通过网络从maven仓库上下载， 通常直接访问国外的地址会比较慢， 需要设置代理来加快速度

若是使用 VPN 或者设置全局代理， 当然可以应用到 gradle 上， 但是若是只需要单独为 gradle 设置代理， 以是可以的， 在 project 的 根目录添加 gradle.properties 文件:

    systemProp.http.proxyHost=xxx.xxx.xxx.xxx
    systemProp.http.proxyPort=xxxx
    systemProp.https.proxyHost=xxx.xxx.xxx.xxx
    systemProp.https.proxyPort=xxxx

根据实际情况设置 IP 和 端口号









