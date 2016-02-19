---
layout: post
title: "linux上配置git server和repo"
description:
category: tool
tags: [tool,linux]
mathjax: 
chart:
comments: false
---

###1. 配置git server

git 的远程仓库和本地仓库没有太大的区别， 例如， 可以使用 git clone 直接克隆本地的仓库

    $ git clone /home/xxx/xx.git
    
git 支持 https 和 ssh 协议来进行传输， 其中 https 使用简单， 无需配置， 但是每次执行 fetch 和 push 的时候 都要输入帐号和密码， ssh 支持 密码和密钥2种认证方式， 若配置为密钥方式， 则可以免除每次fetch和push操作都要输入密码的麻烦， 因此， 最好使用 ssh 的方式

事实上， 配置git server的主要工作就是配置ssh

####1.1 安装 openssh-server

在要搭建 git server 的机器上需要安装 openssh-server 以支持 ssh 登录

    $ apt-get instamll openssh-server
    
安装完成后， 回自动启动 sshd 以支持 ssh连接

####1.2 安装 git

在要搭建的 git server 的机器上需要安装 git 和 git-core

    $ apt-get install git git-core
    
####1.3 添加git用户

    $ sudo adduser git
    
**这一步是可选的， 这一帐号主要是用于 ssh 连接， 因此也可以使用其它的账户**， 后续将以 “git” 这一 user 为例进行讲解， 若使用其它的user， 则在执行后续步骤需要替换成实际的user
     
###1.4 切换到git用户完成后续的配置

    $ su git
    
后续在要搭建 git server 的机器上的配置都是以 git 的身份来操作

###5. 配置 ssh 所需的公私钥

ssh 的密钥认证方式需要一对公私钥， 其中公钥存放在 server 端， 私钥存放在 client 端

首先， 在作为 git client 的机器上安装 openssh-client

    $ sudo apt-get install openssh-client
    
然后执行

    ssh-keygen -C "some comment"
    
“some comment” 通常填自己的email地址

生成key apirs的过程中回提示输入密码， 可以忽略， 直接按 “enter” 键继续

生成的公私钥分别为 “.ssh/id_rsa.pub” "/home/git/.ssh/id_rsa"， 分别重命名为 “.ssh/git-client.pub” ".ssh/git-client.priv"

如果一个用户存储有多个 ssh 的私钥， 那么执行 ssh 连接时， 可能不知道使用哪一份私钥， 因此， 要在 "~/.ssh/config" 文件中进行配置

    host xxx.xxx.xxx.xxx
    user git
    hostname git server alias
    port
    identityfile ~/.ssh/git-client.priv
    
其中

+ host : git server所在的机器的 ip
+ user : git server上提供git远程仓库的用户
+ hostname : 给git server所取的别名
+ port : git server所在的机器的ssh端口
+ identityfile : ssh 连接的私钥

在 "~/.ssh/config" 中可以存在多个这样的配置块

将 作为 git client的机器上的 “～/.ssh/git-client.pub” 中的内容追加到 git server 的 “/home/git/.ssh/authorized_key” 中(若该文件不存在则新建一个， 若该文件中有多个公钥， 以空行隔开)

###1.6 测试git server

在作为git server的机器上执行如下的命令建立一个git project用于测试

    $ cd ~/
    $ git init --bare test.git

在作为git server的机器上执行如下的命令进行fetch/push测试

    $ git clone ssh://xxx.xxx.xxx.xxx/home/git/test.git
    
    $ cd test
    $ echo "hello" > text.txt
    $ git add text.txt
    $ git commit -m "add test.txt"
    $ git push origin master
    
如果都执行成功， 则说明 git server 可用

###2. 配置repo仓库

android 使用git作为代码管理工具， 还开发了repo命令行工具，对Git部分命令封装，对几百个git库有效的进行组织和管理

repo 是一个python开发的工具， 对git命令进行了封装， 另外repo需要配置一个名为 manifest 的git project， 并且编辑一个 default.xml 文件， 列出相应的git project的信息， 因此repo可以看作是利用一个git project来管理若干多个git project

虽然repo是为了android 而开发的， 但是也可以用于管理其它的git project

**假设要在 git server 的 /home/git/test/ 目录下建立一个 repo 仓库， 其管理3个 git project : apple, banana, orange**

####2.1 建立manifest.git

    $ mkdir /home/git/test
    $ cd /home/git/test
    $ git init --bare manifest.git
    
这一步的"--bare" 参数指定建立一个git裸仓库， 裸仓库没有工作区(看不到其中的内容)， 因此， 用户无法登录到git server上直接修改, 需要由 git client 克隆该 project， 修改并且提交后， 再push到 git server 上
    
####2.2 编辑default.xml

因为 manifest.git 为裸仓库， 无法直接在git server上修改， 因此， 需要clone到git client上修改

在git client上执行

    $ git clone ssh://xxx.xxx.xxx.xxx/home/git/test/manifest.git
    $ cd manifest
    $ gedit default.xml
    
输入如下的内容

    <?xml version="1.0" encoding="UTF-8"?>
    <manifest>
        <remote name="repo-server" fetch="/home/git/test/" review="review.source.Android.com" />
        <default revision="master" remote="repo-server" />
        <project name="apple" path="apple"/>
        <project name="banana" path="banana"/>
    </manifest>

也可以添加多个 “&lt;remote&gt;”，指定不同的fetch路径， 在repo init时， 通过选择不同的 remote， 实现clone不同位置的 git project的目的

保存后执行

    $ git add default.xml
    $ git commit -m "[add] add default.xml"
    $ git push origin master

###2.3 添加repo管理的git project

依次建立repo 仓库所管理的3个 git project

    $ cd /home/git/test
    $ git init --bare apple.git
    $ git init --bare banana.git
    $ git init --bare orange.git

default.xml 的 “&lt;remote&gt;” 项中的 “fetch” 和 “&lt;project&gt;” 项中的 “path”组合后， 正好是这些project的路径

###2.4 在repo client上安装repo

如果希望从repo仓库中获取git project， 还需要在机器上先安装repo工具

    $ git clone git://aosp.tuna.tsinghua.edu.cn/android/git-repo.git/ 
    $ sudo cp git-repo/repo /usr/local/bin/
    $ sudo chmod a+x /usr/local/bin/repo
    
事实上， 我们使用的 repo 工具只是一个bootstrap， 在执行 repo init时， 会先clone repo的source project， 因为国内网络的原因，从google获取repo 时可能回失败， 因此， 先修改repo的git project 

    $ sudo gedit /usr/local/bin/repo
    
修改第5行

    - REPO_URL = 'https://gerrit.googlesource.com/git-repo'
    + REPO_URL = 'git://aosp.tuna.tsinghua.edu.cn/android/git-repo.git/'

这是一个可用的repo的git project

**当然， 也可以将repo 的git projec放置在自己的git server上， 然后修改repo脚本中的“REPO_URL”项**

####2.5 测试repo server

在repo client机器上执行

    $ mkdir temp
    $ cd temp
    $ repo init -u ssh://xxx.xxx.xxx.xxx/home/git/test/manifest.git
    $ repo sync
    
若成功同步了 apple， banana， orange 这3个project， 则说明repo 仓库配置成功
