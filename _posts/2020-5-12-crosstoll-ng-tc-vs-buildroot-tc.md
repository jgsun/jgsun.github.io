---
layout: post
title:  "crosstool-ng toolchain vs buildroot toolchain"
categories: build
tags: crosstool-NG, buildroot toolchain, x86-64
author: jgsun
---


* content
{:toc}
# 1. Overview
最近使用buildroot编译qemu_x86-64，其qemu_x86_64_defconfig配置的是toolchain是buildroot internal toolchain（后面简称buildroot toolchain），buildroot库曾经编过qemu arm64 virt，虽经make clean/make distclean，但是编译qemu_x86-64仍然报gcc configure的错误，最后重新clone buildroot库后才编译成功。便想使用crosstool-ng给buildroot制作external toolchain，并探究buildroot toolchain为何会失败，比较其和crosstool-ng toolchain的区别。
本文记录了crosstool-ng制作x86-64 toolchain的log，buidlroot使用该toolchain的配置，最后对比了crosstool-ng toolchain和buildroot toolchain的差异。
























# 2. 使用crosstool-ng制作x86-64 toolchain
crosstool-ng制作toolchain非常简单，其有常见架构的初始配置，按照官方document步骤操作即可。
crosstool-ng代码在github网站，其官方网站是http://crosstool-ng.github.io/
```
Crosstool-NG is a versatile (cross) toolchain generator. 
It supports many architectures and components and has a simple 
yet powerful menuconfig-style interface. 
Please read the introduction and refer to the documentation for more information.
```
## 2.1 clone crosstool-ng并切换到stable版本1.24.0
```
jgsun@VirtualBox:~/repo$ git clone https://github.com/crosstool-ng/crosstool-ng
jgsun@VirtualBox:~/repo/crosstool-ng$ git checkout crosstool-ng-1.24.0
HEAD is now at b2151f1d Merge pull request #1182 from stilor/master
```
## 2.2 编译安装crosstool-ng
```
jgsun@VirtualBox:~/repo/crosstool-ng$ ./bootstrap 
jgsun@VirtualBox:~/repo/crosstool-ng$ sudo apt-get install texinfo
jgsun@VirtualBox:~/repo/crosstool-ng$ sudo apt-get install help2man
jgsun@VirtualBox:~/repo/crosstool-ng$ sudo apt-get install gawk
jgsun@VirtualBox:~/repo/crosstool-ng$ sudo apt install libtool-bin libtool-doc
jgsun@VirtualBox:~/repo/crosstool-ng$ ./configure
jgsun@VirtualBox:~/repo/crosstool-ng$ sudo make install
```
## 2.3 配置x86_64 toolchain
```
jgsun@VirtualBox:~/repo/crosstool-ng$ ct-ng list-samples | grep x86_64   
[L..X] x86_64-w64-mingw32,arm-cortexa9_neon-linux-gnueabihf
[L..X] x86_64-multilib-linux-uclibc,moxie-unknown-moxiebox
[L...] x86_64-multilib-linux-uclibc,powerpc-unknown-elf
[L...] x86_64-centos6-linux-gnu
[L...] x86_64-centos7-linux-gnu
[L...] x86_64-multilib-linux-gnu
[L..X] x86_64-multilib-linux-musl
[L...] x86_64-multilib-linux-uclibc
[L..X] x86_64-w64-mingw32,x86_64-pc-linux-gnu
[L...] x86_64-ubuntu12.04-linux-gnu
[L...] x86_64-ubuntu14.04-linux-gnu
[L...] x86_64-ubuntu16.04-linux-gnu
[L...] x86_64-unknown-linux-gnu
[L...] x86_64-unknown-linux-uclibc
[L..X] x86_64-w64-mingw32
jgsun@VirtualBox:~/repo/crosstool-ng$ ct-ng show-x86_64-unknown-linux-gnu
[L...] x86_64-unknown-linux-gnu
    Languages : C,C++
    OS : linux-4.20.8
    Binutils : binutils-2.32
    Compiler : gcc-8.3.0
    C library : glibc-2.29
    Debug tools : gdb-8.2.1
    Companion libs : expat-2.2.6 gettext-0.19.8.1 gmp-6.1.2 isl-0.20 libiconv-1.15 mpc-1.1.0 mpfr-4.0.2 ncurses-6.1 zlib-1.2.11
    Companion tools :
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ mkdir x86_64-cross
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ cd x86_64-cross
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ ct-ng x86_64-unknown-linux-gnu     
  CONF x86_64-unknown-linux-gnu
#
# configuration written to .config
# 
***********************************************************

Initially reported by: Thomas JOURDAN
URL: 

***********************************************************

Now configured for "x86_64-unknown-linux-gnu"

```
## 2.4 编译x86_64 toolchain
```
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ sudo apt install python3
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ sudo apt-get install python3.6-dev
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ ct-ng build
```
需要安装python3.6-dev，否则报错：（先仅安装python3，还是报同样的错误）
```
[EXTRA] Building cross gdb
[ERROR] configure: error: no usable python found at /usr/bin/python3.6
[ERROR] make[2]: *** [configure-gdb] Error 1
[ERROR] make[1]: *** [all] Error 2
```
出现如下log表示编译成功
```
[INFO ] Finalizing the toolchain's directory
[INFO ] Stripping all toolchain executables
[EXTRA] Installing the populate helper
[EXTRA] Installing a cross-ldd helper
[EXTRA] Creating toolchain aliases
[EXTRA] Removing installed documentation
[EXTRA] Collect license information from: /home/jgsun/repo/crosstool-ng/x86_64-cross/.build/x86_64-unknown-linux-gnu/src
[EXTRA] Put the license information to: /home/jgsun/x-tools/x86_64-unknown-linux-gnu/share/licenses
[INFO ] Finalizing the toolchain's directory: done in 7.38s (at 62:43)
[INFO ] Build completed at 20200511.225239
[INFO ] (elapsed: 62:42.08)
[INFO ] Finishing installation (may take a few seconds)...
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ 
```
生成的toolchain在目录 /home/jgsun/x-tools/x86_64-unknown-linux-gnu/bin/：
```
jgsun@VirtualBox:~/repo/crosstool-ng/x86_64-cross$ ls ~/x-tools/x86_64-unknown-linux-gnu/bin/
x86_64-unknown-linux-gnu-addr2line x86_64-unknown-linux-gnu-ct-ng.config x86_64-unknown-linux-gnu-gcc-nm x86_64-unknown-linux-gnu-gprof x86_64-unknown-linux-gnu-objdump
x86_64-unknown-linux-gnu-ar x86_64-unknown-linux-gnu-dwp x86_64-unknown-linux-gnu-gcc-ranlib x86_64-unknown-linux-gnu-ld x86_64-unknown-linux-gnu-populate
```
# 3. buildroot使用external toolchain
首先按下图配置toolchain：
![image](/images/posts/images/posts/build/buildroot-ex-toolchain.png)


然后make编译，toolchain被就被安装到了output/host/bin目录，可以使用了。
```
jgsun@VirtualBox:~/repo/buildroot$ make
/usr/bin/make -j1 O=/home/jgsun/repo/buildroot/output HOSTCC="/usr/bin/gcc" HOSTCXX="/usr/bin/g++" syncconfig
make[1]: Entering directory '/home/jgsun/repo/buildroot'
make[1]: Leaving directory '/home/jgsun/repo/buildroot'
>>> toolchain-external-custom Configuring
>>> toolchain-external-custom Building
/usr/bin/gcc -O2 -I/home/jgsun/repo/buildroot/output/host/include -DBR_64 -DBR_ARCH='"nocona"' -DBR_CROSS_PATH_SUFFIX='""' -DBR_CROSS_PATH_ABS='"/home/jgsun/x-tools/x86_64-unknown-linux-gnu/bin"' -DBR_SYSROOT='"x86_64-buildroot-linux-gnu/sysroot"' -DBR_ADDITIONAL_CFLAGS='' -s -Wl,--hash-style=both toolchain/toolchain-wrapper.c -o /home/jgsun/repo/buildroot/output/build/toolchain-external-custom/toolchain-wrapper
>>> toolchain-external-custom Installing to staging directory
/usr/bin/install -D -m 0755 /home/jgsun/repo/buildroot/output/build/toolchain-external-custom/toolchain-wrapper /home/jgsun/repo/buildroot/output/host/bin/toolchain-wrapper
>>> toolchain-external-custom Copying external toolchain sysroot to staging...
>>> toolchain-external-custom Installing gdbinit
>>> toolchain-external-custom Fixing libtool files
```
#  4. crosstool-NG instead of buildroot toolchain?
[External toolchain: crosstool-NG instead of buildroot? ](http://buildroot-busybox.2317881.n4.nabble.com/External-toolchain-crosstool-NG-instead-of-buildroot-td9837.html)
我认为crosstool-ng的introduction文档[crosstool-ng](https://crosstool-ng.github.io/docs/introduction/) 对比了crosstool-NG toolchain和buildroot toolchains的区别，很好的解释了这个问题： buildroot toolchain在多个toolchains交替编译的时候使用不方便，我就遇到先编译arm64，接着编译x86，就会报gcc不能config的错误；在buildroot 执行make distclean也不行，必须重新clone 一个干净的buildroot，一切从0开始编译才行（即下文说的restart again from scratch）；如果使用crosstool-ng toolchain就没有这么麻烦。
```
There are also a number of tools that build toolchains for specific needs, which are not really scalable. Examples are:
buildroot, whose main purpose is to build root file systems, hence the name. But once you have your toolchain with buildroot, part of it is installed in the root-to-be, so if you want to build a whole new root, you either have to save the existing one as a template and restore it later, or restart again from scratch. This is not convenient.
```
buildroot 手册上也说了buildroot toolchain的优缺点：
```
Advantages of this backend:
    Well integrated with Buildroot
    Fast, only builds what’s necessary
Drawbacks of this backend:
    Rebuilding the toolchain is needed when doingmake clean, which takes time. If you’re trying to reduce your build time,consider using theExternal toolchain backend.
```
总结下，buildroot toolchail不够灵活，使用不方便；此外，crosstool-NG有社区支持，制作方便，更加专业。

# 5. 参考文档
* [crosstool-NG Documentation](http://crosstool-ng.github.io/docs/)