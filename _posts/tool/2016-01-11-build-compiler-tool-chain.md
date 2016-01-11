---
layout: post
title: "构建交叉编译工具链"
description:
category: tool
tags: [tool, linux]
imagefeature: cover10.jpg
mathjax: 
chart:
comments: false
---

###1. 建立编译的目录树

    ~/tool-chain
            |-- src
            |-- build
            |-- tools

    export TARGET=x86_64-linux
    export PREFIX=~/tool-chain/tools

本文以在x86_64上编译x86_64的工具链为例

###2. 编译 binutils

从 gnu 官网下载 binutils [binutils-2.25](http://ftp.gnu.org/gnu/binutils/), 本文以 binutils-2.25.tar.bz2 为例

    $ cd ~/Downloads
    $ wget http://ftp.gnu.org/gnu/binutils/binutils-2.25.tar.bz2
    $ cd ~/tool-chain/src
    $ tar jxvf ~/Downloads/binutils-2.25.tar.bz2 
    
    $ cd ~/tool-chain/build
    $ mkdir binutils
    $ cd binutils
    $ ../../src/binutils-2.25/configure --target=$TARGET --prefix=$PREFIX
    $ make -j6
    $ make install
    
###3. 编译bootstrap gcc

这一不需要编译一个简单的gcc， 用于编译glibc

从 gnu 官网下载 gcc 源码 [gcc源码](http://ftp.gnu.org/gnu/gcc/)
从 gnu 官网下载 gcc 的依赖库 gmp [gmp源码](http://ftp.gnu.org/gnu/gmp/)
从 gnu 官网下载 gcc 的依赖库 mpc [mpc源码](http://ftp.gnu.org/gnu/mpc/)
从 gnu 官网下载 gcc 的依赖库 mpfr [mpfr源码](http://ftp.gnu.org/gnu/mpfr/)

    $ cd ~/Downloads
    $ wget http://ftp.gnu.org/gnu/gcc/gcc-5.2.0/gcc-5.2.0.tar.bz2
    $ wget http://ftp.gnu.org/gnu/gmp/gmp-6.1.0.tar.bz2
    $ wget http://ftp.gnu.org/gnu/mpc/mpc-1.0.3.tar.gz
    $ wget http://ftp.gnu.org/gnu/mpfr/mpfr-3.1.2.tar.bz2
    
    $ cd ~/tool-chain/src
    $ tar jxvf ~/Downloads/gcc-5.2.0.tar.bz2
    $ tar jxvf ~/Downloads/gmp-6.1.0.tar.bz2
    $ tar zxvf ~/Downloads/mpc-1.0.3.tar.gz
    $ tar jxvf ~/Downloads/mpfr-3.1.2.tar.bz2
    
    //将gmp mpc mpfr 目录拷贝到 gcc 源码目录中并重命名
    $ mv gmp-6.1.0/ gcc-5.2.0/gmp
    $ mv mpc-1.0.3/ gcc-5.2.0/mpc
    $ mv mpfr-3.1.2/ gcc-5.2.0/mpfr
    
    //若无这一步， 编译glibc时会有 "more undefined references to `__stack_chk_guard' follow" 错误
    $ cd gcc-5.2.0
    $ sed -i '/k prot/agcc_cv_libc_provides_ssp=yes' gcc/configure 
    
    $ mkdir ~/tool-chain/build/gcc-boot
    $ cd ~/tool-chain/build/gcc-boot
    
    $ ../../src/gcc-5.2.0/configure --target=$TARGET --prefix=$PREFIX -disable-threads --disable-shared --enable-languages=c --without-headers
  
    $ make all-gcc -j6
    $ make all-target-libgcc -j6
    $ make install-gcc
    $ make install-target-libgcc
    
###4. 生成kernel头文件

从 kernel.org 下载 linux 源码 [linux kernel 源码](https://www.kernel.org/pub/linux/kernel/v4.x/)

    $ cd ~/Downloads
    $ wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.1.15.tar.xz
    
    $ cd ~/tool-chain/src
    $ tar xvf ~/Downloads/linux-4.1.15.tar.xz
    
    $ mkdir ~/tool-chain/build/kernel
    $ make defconfig O=~/tool-chain/build/kernel
    
    $ make ARCH=x86_64 CROSS_COMPILE=x86_64-linux- O=~tool-chain/build/kernel/ defconfig
    $ make ARCH=x86_64 CROSS_COMPILE=x86_64-linux- O=tool-chain/build/kernel/ headers_check
    $ make ARCH=x86_64 CROSS_COMPILE=x86_64-linux- O=tool-chain/build/kernel/ headers_install INSTALL_HDR_PATH=~/tool-chain/tools/include

执行完成后， 可以看到在 “～/tool-chain/tools/include” 目录下生成了对应的linux 头文件    
    
###5. 编译 glibc

从 gnu 官网下载 libgcc 的源码 [glibc源码](http://ftp.gnu.org/gnu/libc/)

    $ cd ~/Downloads
    $ wget http://ftp.gnu.org/gnu/libc/glibc-2.22.tar.bz2
    $ cd ~/tool-chain/src
    $ tar jxvf ~/Downloads/glibc-2.22.tar.bz2
    $ mkdir ~/tool-chain/build/glibc
    $ cd ~/tool-chain/build/glibc
    $ CC=x86_64-linux-gcc ../../src/glibc-2.22/configure  --build=$TARGET --host=$TARGET --prefix=/usr --disable-profile  --with-headers=$PREFIX/include --with-binutils=$PREFIX/bin/ --enable-add-ons

    $ make -j6
    $ make install_root=${TARGET_PREFIX} DESTDIR="" install
    
需要注意的是， config 阶段的 --prefix 指定的是 glibc在目标机器上的位置， 在host机器上的安装则要使用 install_root 来指定

安装完成后， 还要修改 “～/tool-chain/lib/libc.so”， 将其中的 GROUP 项中的  libc.so.6 libc_nonshared.a 的路径修改为刚刚编译安装的glibc的libc.so.6 和libc_nonshared.a的路径， 例如， 将GROUP ( /lib/libc.so.6 /lib/libc_nonshared.a)改为GROUP ( libc.so.6 libc_nonshared.a)， 这样连接程序 ld 就会在 libc.so 所在的目录查找它需要的库，因为你的机子的/lib目录可能已经装了一个相同名字的库，一个为编译可以在你的宿主机上运行的程序的库，而不是用于交叉编译的库

###6. 编译全功能的gcc

    $ cd ~/tool-chain/build
    $ mkdir gcc-full
    $ cd gcc-full
    $ ../../src/gcc-5.2.0/configure --target=$TARGET --prefix=$PREFIX --enable-languages=c,c++ --enable-shared --disable-nls --enable-c99 --enable-long-long  --disable-multilib
    $ make -j6
    $ make install
    
###7. 参考

+ [http://blog.chinaunix.net/uid-13075095-id-2907611.html](http://blog.chinaunix.net/uid-13075095-id-2907611.html)
+ [https://www.ibm.com/developerworks/cn/linux/l-embcmpl/](https://www.ibm.com/developerworks/cn/linux/l-embcmpl/)
+ [http://blog.csdn.net/turui/article/details/6596093](http://blog.csdn.net/turui/article/details/6596093)
+ [http://www.cnblogs.com/leaven/archive/2010/11/17/1879679.html](http://www.cnblogs.com/leaven/archive/2010/11/17/1879679.html)