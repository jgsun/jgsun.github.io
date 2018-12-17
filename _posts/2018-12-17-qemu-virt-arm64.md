---
layout: post
title:  "QEMU模拟arm64 virt u-boot/linux"
categories: virtualize
tags: QEMU virt u-boot linux
author: jgsun
---

* content
{:toc}

# 概述
目前，QEMU里面32bit arm平台比较多，如vexpress-a9，versatilepb等， 64bit arm64平台比较少。而Linero开发的"virt"平台可同时支持32bit 和64bit的arm（详见QEMU源码[/hw/arm/virt.c
](https://github.com/qemu/qemu/blob/master/hw/arm/virt.c)）。“virt”支持 PCI，virtio，最新的arm CPU 和 大容量RAM。如果只想在最新的arm cpu上面运行linux虚拟机，不在意特定的硬件，"virt"平台是最佳选择。
关于“virt”，可进一步参考：
* [https://wiki.qemu.org/Documentation/Platforms/ARM](https://wiki.qemu.org/Documentation/Platforms/ARM)
* [Installing Debian on QEMU’s 32-bit ARM “virt” board](https://translatedcode.wordpress.com/2016/11/03/installing-debian-on-qemus-32-bit-arm-virt-board/)
* [Installing Debian on QEMU’s 64-bit ARM “virt” board](https://translatedcode.wordpress.com/2017/07/24/installing-debian-on-qemus-64-bit-arm-virt-board/)

本文详细记录使用buildroot集成编译、使用QEMU "virt"平台来模拟运行arm64 u-boot/linux的过程，从而搭建arm64/linux的学习平台。











# QEMU 模拟运行 "virt" u-boot
官方的u-boot源码支持QEMU arm "virt"平台，详见文档[U-Boot on QEMU's 'virt' machine on ARM & AArch64](https://github.com/u-boot/u-boot/blob/master/doc/README.qemu-arm)。
## buildroot编译u-boot
buildroot设置如图：
![image](/images/posts/virtualize/qemu-buildroot-virt-uboot.png)

## qemu运行u-boot
指定machine参数为virt，bios参数为u-boot.bin，加上-nographic即可运行u-boot：
```
jgsun@VirtualBox:/work/buildroot$ qemu-system-aarch64 -machine virt -cpu cortex-a57 -bios output/images/u-boot.bin -nographic

U-Boot 2018.11 (Dec 17 2018 - 14:04:48 +0800)

DRAM: 128 MiB
In: pl011@9000000
Out: pl011@9000000
Err: pl011@9000000
Net: No ethernet found.
Hit any key to stop autoboot: 0 
=> 
```
## flash启动"virt" u-boot
"virt"平台支持flash启动u-boot，先制作flash，将u-boot.bin拷贝flash的零地址，加-drive参数即可从flash启动u-boot：
1.制作flash.bin并将u-boot.bin拷贝到0地址
```
jgsun@VirtualBox:/work/buildroot$ dd if=/dev/zero of=flash.bin bs=4096 count=16384
jgsun@VirtualBox:/work/buildroot$ dd if=output/images/u-boot.bin of=flash.bin conv=notrunc bs=4096
```
2.加-drive参数从flash启动u-boot
```
jgsun@VirtualBox:/work/buildroot$ qemu-system-aarch64 -machine virt -cpu cortex-a57 -m 1G -drive file=flash.bin,format=raw,if=pflash -nographic

U-Boot 2018.11 (Dec 17 2018 - 14:04:48 +0800)

DRAM: 1 GiB
In: pl011@9000000
Out: pl011@9000000
Err: pl011@9000000
Net: No ethernet found.
Hit any key to stop autoboot: 0 
=> 
```
## 给u-boot添加网络支持
> 注意：要使用较新的u-boot版本， 刚开始使用u-boot-2018.05版本包eth0 mac address未设置的错误，后来换用u-boot-2018.11的版本就可以了。

virbr0 是 KVM 默认创建的一个 bridge，其作用是为连接其上的虚机网卡提供 NAT 访问外网的功能。virbr0 默认分配了一个IP 192.168.122.1，并为连接其上的其他虚拟网卡提供 DHCP 服务。利用这一机制让QEMU模拟的arm64 u-boot可以访问host网络，就可以从tftp服务器引导linux了。
qemu-ifup_virbr0脚本内容：

```
jgsun@VirtualBox:/work/buildroot$ cat board/qemu/scripts/qemu-ifup_virbr0 
#!/bin/sh
run_cmd()
{
 echo $1
 eval $1
}
run_cmd "sudo ifconfig $1 0.0.0.0 promisc up"
run_cmd "sudo brctl addif virbr0 $1"
run_cmd "brctl show"
```
启动之后设置ipaddr环境变量之后，可以ping通host ip 192.168.122.1：
```
jgsun@VirtualBox:/work/buildroot$ sudo /work/repo/qemu/qemu-src/aarch64-softmmu/qemu-system-aarch64 -machine virt -cpu cortex-a57 -m 1G -drive file=flash.bin,format=raw,if=pflash -nographic -netdev type=tap,id=net0,script=board/qemu/scripts/qemu-ifup_virbr0 -device e1000,netdev=net0
sudo ifconfig tap0 0.0.0.0 promisc up
sudo brctl addif virbr0 tap0
brctl show
bridge name	bridge id	STP enabled	interfaces
virbr0	8000.5254005634be	yes	tap0
       virbr0-nic


U-Boot 2018.11 (Dec 17 2018 - 14:04:48 +0800)

DRAM: 1 GiB
In: pl011@9000000
Out: pl011@9000000
Err: pl011@9000000
Net: No ethernet found.
Hit any key to stop autoboot: 0 
```
运行bootcmd_dhcp环境变量使用dhcp client获取id地址和设置服务器地址，即virbr0的地址192.168.122.1：
```
=> run bootcmd_dhcp
starting USB...
No controllers found
e1000: 52:54:00:12:34:56
       
Warning: e1000#0 using MAC address from ROM
BOOTP broadcast 1
DHCP client bound to address 192.168.122.76 (7 ms)
Using e1000#0 device
TFTP from server 192.168.122.1; our IP address is 192.168.122.76
Filename 'boot.scr.uimg'.
Load address: 0x40200000
Loading: *
TFTP error: 'File not found' (1)
Not retrying...
BOOTP broadcast 1
DHCP client bound to address 192.168.122.76 (6 ms)
Using e1000#0 device
TFTP from server 192.168.122.1; our IP address is 192.168.122.76
Filename 'boot.scr.uimg'.
Load address: 0x40400000
Loading: *
TFTP error: 'File not found' (1)
Not retrying...
```
ping服务器地址：
```
=> ping 192.168.122.1
Using e1000#0 device
host 192.168.122.1 is alive
```

# QEMU 模拟运行 "virt" linux
buildroot已经有arm64 "virt"平台的默认配置[configs/qemu_aarch64_virt_defconfig](https://git.buildroot.net/buildroot/tree/configs/qemu_aarch64_virt_defconfig)，所以直接运行make qemu_aarch64_virt_defconfig配置之后编译之后，运行board目录readme文档[board/qemu/aarch64-virt/readme.txt](https://git.buildroot.net/buildroot/tree/board/qemu/aarch64-virt/readme.txt)的启动命令就可以运行linux了。

这里根据实验需要修改和增加了一些配置，所用配置文件见我的github仓库：[buildroot/configs/qemu_aarch64_virt-fun_defconfig](https://github.com/jgsun/buildroot/blob/master/configs/qemu_aarch64_virt-fun_defconfig)，添加了网络支持，最终运行命令是：
```
jgsun@VirtualBox:/work/buildroot$ cat board/qemu/scripts/start_qemu_aarch64.sh 
sudo /work//repo/qemu/qemu-src/aarch64-softmmu/qemu-system-aarch64 -M virt \
 -cpu cortex-a57 -nographic -smp 4 -m 512 \
 -kernel output/images/Image \
 -append "root=/dev/ram0 console=ttyAMA0 kmemleak=on loglevel=8" \
 -netdev type=tap,ifname=tap0,id=eth0,script=board/qemu/scripts/qemu-ifup_virbr0,queues=2 \
 -device virtio-net-pci,netdev=eth0,mac='00:00:00:01:00:01',vectors=6,mq=on \
```