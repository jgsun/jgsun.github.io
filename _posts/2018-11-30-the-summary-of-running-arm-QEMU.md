---
layout: post
title:  "QEMU模拟 arm u-boot/linux"
categories: system
tags: arm QEMU linux u-boot buildroot
author: jgsun
---

* content
{:toc}

# 概述
这篇文章是用QEMU模拟运行arm u-boot和linux的一个总结，以arm  vexpres板为例，包括用QEMU单独运行u-boot或者linux；实现QEMU运行u-boot和宿主机ubuntu网络通信，u-boot用tftp下载方式引导linux；u-boot从SD卡或者flash引导linux。本文使用buildroot作为构建系统，极大简化了编译和文件系统制作方面的工作。以buildroot自带的configs/qemu_arm_vexpress_defconfig作为起点，增加u-boot，改rootfs为initramfs等，产生了本文所用的默认配置configs/qemu_arm_vexpress-fun_defconfig，保存在github网站:https://github.com/jgsun/buildroot，用于制作sd卡和flash image的脚本文件也放到了该github网站（board/qemu/scripts目录）。

关键字：windows7，virtualbox，ubuntu18.04，buildroot







# 使用buildroot编译
buildroot已经有arm vexpress的default配置，开始`/repo/buildroot$ make qemu_arm_vexpress_defconfig`，然后在此基础上在bootloader选择u-boot，在toolchain选择glibc，编译即可，得到output/build/uboot-2018.05/u-boot，output/images/rootfs.ext2，output/images/zImage和output/images/vexpress-v2p-ca9.dtb用于QEMU启动u-boot和linux。
![image](/images/posts/virtualize/qemu-menuconfig-bootloader.png)
![image](/images/posts/virtualize/qemu-menuconfig-toolchain.png)

# QEMU运行u-boot
```
jgsun@jgsun-VirtualBox:/repo/buildroot$ qemu-system-arm -M vexpress-a9 -kernel output/build/uboot-2018.05/u-boot -nographic
pulseaudio: set_sink_input_volume() failed
pulseaudio: Reason: Invalid argument
pulseaudio: set_sink_input_mute() failed
pulseaudio: Reason: Invalid argument


U-Boot 2018.05 (Jul 08 2018 - 15:27:56 +0800)

DRAM: 128 MiB
WARNING: Caches not enabled
Flash: 128 MiB
```
# QEMU运行linux
```
jgsun@jgsun-VirtualBox:/repo/buildroot$ cat board/qemu/arm-vexpress/readme.txt 
Run the emulation with:

  qemu-system-arm -M vexpress-a9 -smp 1 -m 256 -kernel output/images/zImage -dtb output/images/vexpress-v2p-ca9.dtb -drive file=output/images/rootfs.ext2,if=sd,format=raw -append "console=ttyAMA0,115200 root=/dev/mmcblk0" -serial stdio -net nic,model=lan9118 -net user

The login prompt will appear in the terminal that started Qemu. The
graphical window is the framebuffer.

If you want to emulate more cores change "-smp 1" to "-smp 2" for
dual-core or even "smp -4" for a quad-core configuration.

Tested with QEMU 2.12.0
```
启动log
```
jgsun@jgsun-VirtualBox:/repo/buildroot$ qemu-system-arm -M vexpress-a9 -smp 1 -m 256 -kernel output/images/zImage -dtb output/images/vexpress-v2p-ca9.dtb -drive file=output/images/rootfs.ext2,if=sd,format=raw -append "console=ttyAMA0,115200 root=/dev/mmcblk0" -serial stdio -net nic,model=lan9118 -net user
pulseaudio: set_sink_input_volume() failed
pulseaudio: Reason: Invalid argument
pulseaudio: set_sink_input_mute() failed
pulseaudio: Reason: Invalid argument
Booting Linux on physical CPU 0x0
Linux version 4.16.7 (jgsun@jgsun-VirtualBox) (gcc version 7.3.0 (Buildroot 2018.08-git-00596-gb99dbdfac9)) #1 SMP Sun Jul 8 15:36:11 CST 2018
CPU: ARMv7 Processor [410fc090] revision 0 (ARMv7), cr=10c5387d
CPU: PIPT / VIPT nonaliasing data cache, VIPT nonaliasing instruction cache
OF: fdt: Machine model: V2P-CA9
```
## 改用initramfs之后的启动命令log
```
cd /repo/training/qemu/vexpress-a9
qemu-system-arm -M vexpress-a9 -smp 4 -m 1024 -kernel uImage -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0,115200 root=/dev/ram0" -net nic,model=lan9118 -net user -nographic
jgsun@VirtualBox:/repo/training/qemu/vexpress-a9$ qemu-system-arm -M vexpress-a9 -smp 4 -m 1024 -kernel uImage -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0,115200 root=/dev/ram0" -net nic,model=lan9118 -net user -nographic
pulseaudio: set_sink_input_volume() failed
pulseaudio: Reason: Invalid argument
pulseaudio: set_sink_input_mute() failed
pulseaudio: Reason: Invalid argument
Booting Linux on physical CPU 0x0
Linux version 4.16.7 (jgsun@VirtualBox) (gcc version 7.3.0 (Buildroot 2018.08-git-00596-gb99dbdfac9-dirty)) #7 SMP Thu Jul 12 18:56:46 CST 2018
CPU: ARMv7 Processor [410fc090] revision 0 (ARMv7), cr=10c5387d
...
udhcpc: started, v1.28.4
udhcpc: sending discover
udhcpc: sending select for 10.0.2.15
udhcpc: lease of 10.0.2.15 obtained, lease time 86400
deleting routers
adding dns 10.0.2.3
OK

Welcome to Buildroot
buildroot login: root
```
# QEMU运行u-boot引导linux
## 网络引导tftp方式启动Linux内核
### 自建网桥br0
1.创建qemu-ifup_br0、qemu-ifdown_br0脚本
#### qemu-ifup_br0
```
cat board/qemu/scripts/qemu-ifup_br0 
#!/bin/sh
run_cmd()
{
 echo \$1
 eval \$1
}
run_cmd "sudo ifconfig"
run_cmd "sudo brctl addbr br0"
run_cmd "sudo ifconfig br0 10.0.2.16 up"
run_cmd "sudo ip route"
run_cmd "sudo ip addr del 10.0.2.16/24 dev enp0s3" # del ip on enp0s3
run_cmd "sudo ip route"
run_cmd "sudo brctl addif br0 enp0s3"
run_cmd "sudo route add default gw 10.0.2.2"
run_cmd "sudo ip route"
run_cmd "sudo tunctl -u \$(id -un) -t \$1"
run_cmd "sudo ifconfig $1 0.0.0.0 promisc up"
run_cmd "sudo brctl addif br0 \$1"
run_cmd "brctl show"
```
#### qemu-ifdown_br0
```
cat board/qemu/scripts/qemu-ifdown_br0 
#!/bin/sh
run_cmd()
{
 echo \$1
 eval \$1
}
run_cmd "sudo brctl delif br0 enp0s3"
run_cmd "sudo ifconfig br0 down"
run_cmd "sudo brctl delbr br0"
run_cmd "sudo ifconfig enp0s3 10.0.2.16 up"
run_cmd "brctl show"
run_cmd "sudo route add default gw 10.0.2.2"
run_cmd "ip route"
```
qemu-ifup_br0将作为QEMU的net启动script，创建TAP类型的接口tap0和网桥br0；添加tap0和enp0s3到br0；删除enp0s3的ip，为br0配置ip 10.0.2.15；这样QEMU虚拟机u-boot和linux就能联网：既能与宿主机ubuntu进行通信，也能联通外网（如ping 服务器135.251.205.177）。

QEMU虚拟机u-boot引导启动的linux会自动获取和br0相同网段的IP 10.0.2.17，并添加了default route 10.0.2.2，如下所示:
```
# ip addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.17/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
# ip route
default via 10.0.2.2 dev eth0 
10.0.2.0/24 dev eth0 scope link src 10.0.2.17 
# ping 172.24.208.168
PING 172.24.208.168 (172.24.208.168): 56 data bytes
64 bytes from 172.24.208.168: seq=1 ttl=121 time=3.659 ms
64 bytes from 172.24.208.168: seq=2 ttl=121 time=2.787 ms
```
这种配置会让宿主机ubuntu不能联通外网（因为其网卡enp0s3挂载到br0了），需要在宿主机添加default route gw 10.0.2.2之后才能联通外网：
`sudo route add -net 0.0.0.0/32 gw 10.0.2.2 or sudo route add default gw 10.0.2.2` 
```
jgsun@VirtualBox:/repo/work/buildroot$ ip route
default via 10.0.2.2 dev br0 
10.0.0.0/8 dev br0 proto kernel scope link src 10.0.2.15 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 
jgsun@VirtualBox:/repo/work/buildroot$ ping 172.24.208.168
PING 172.24.208.168 (172.24.208.168) 56(84) bytes of data.
64 bytes from 172.24.208.168: icmp_seq=1 ttl=121 time=1.63 ms
64 bytes from 172.24.208.168: icmp_seq=2 ttl=121 time=1.86 ms
```
如果QEMU虚拟机只需与宿主机通信，不需要联通外网，可以不连接网卡enp0s3到网桥br0，就不影响宿主机连外网了，这时要手动将br0和QEMU虚拟机linux的网卡的ip配成一个网段如192.168.2.x。
2.带net参数启动u-boot，tap类型，名称为tap0，script为步骤2创建的qemu-ifup_br0
```
jgsun@VirtualBox:/repo/buildroot$ sudo qemu-system-arm -M vexpress-a9 -m 1024 -kernel output/build/uboot-2018.05/u-boot -nographic -net nic,model=lan9118 -net tap,ifname=tap0,script=board/qemu/arm-vexpress/qemu-ifup_br0
[sudo] password for jgsun: 
sudo tunctl -u root -t tap0
TUNSETIFF: Device or resource busy
sudo ifconfig tap0 0.0.0.0 promisc up
sudo brctl addif br0 tap0
brctl show
bridge name	bridge id	STP enabled	interfaces
br0	8000.080027f6cde5	no	enp0s3
       tap0
       vnet0
virbr0	8000.5254005634be	yes	virbr0-nic
pulseaudio: set_sink_input_volume() failed
pulseaudio: Reason: Invalid argument
pulseaudio: set_sink_input_mute() failed
pulseaudio: Reason: Invalid argument


U-Boot 2018.05 (Jul 08 2018 - 15:27:56 +0800)

DRAM: 128 MiB
WARNING: Caches not enabled
Flash: 128 MiB
MMC: MMC: 0
*** Warning - bad CRC, using default environment

In: serial
Out: serial
Err: serial
Net: smc911x-0
Hit any key to stop autoboot: 0 
=> 
```
3.设置环境变量ipaddr为10.0.2.222，serverip为10.0.2.15（ubuntu br0的ip），并ping通serverip
```
=> setenv ipaddr 10.0.2.222
=> setenv serverip 10.0.2.15
=> ping 10.0.2.15
smc911x: MAC 52:54:00:12:34:56
smc911x: detected LAN9118 controller
smc911x: phy initialized
smc911x: MAC 52:54:00:12:34:56
Using smc911x-0 device
smc911x: MAC 52:54:00:12:34:56
host 10.0.2.15 is alive
```
4.buildroot设置编译
kernel里面选择kernel/Kernel binary format为uImage；load address为0x60003000；
![image](/images/posts/virtualize/qemu-menuconfig-kernel.png)
Filesystem images里面选择initramfs：
![image](/images/posts/virtualize/qemu-menuconfig-fs.png)

编译得到image：uImage和vexpress-v2p-ca9.dtb拷贝到ubuntu的tftp server目录/var/lib/tftpboot
5.在u-boot命令行修改/设置u-boot环境变量bootargs，并用tftp命令下载uImage和vexpress-v2p-ca9.dtb；然后用bootm命令启动linux
注：修改u-boot源码include/configs/vexpress_common.h，可以实现让u-boot自动启动linux
`#define CONFIG_BOOTCOMMAND "tftp 0x60003000 uImage; setenv bootargs 'root=/dev/mmcblk0 console=ttyAMA0';bootm 0x60003000"`
```
setenv bootargs 'root=/dev/ram0 console=ttyAMA0'
tftp 0x60003000 uImage
tftp 0x60900000 vexpress-v2p-ca9.dtb
bootm 0x60003000 - 60900000
```
linux启动界面：
```
=> bootm 0x60003000 - 60900000
## Booting kernel from Legacy Image at 60003000 ...
   Image Name: Linux-4.16.7
   Image Type: ARM Linux Kernel Image (uncompressed)
   Data Size: 6844432 Bytes = 6.5 MiB
   Load Address: 60003000
   Entry Point: 60003000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 60900000
   Booting using the fdt blob at 0x60900000
   Loading Kernel Image ... OK
   Loading Device Tree to 7fee7000, end 7feed7ed ... OK

Starting kernel ...

Booting Linux on physical CPU 0x0
Linux version 4.16.7 (jgsun@VirtualBox) (gcc version 7.3.0 (Buildroot 2018.08-git-00596-gb99dbdfac9-dirty)) #6 SMP Wed Jul 11 15:43:21 CST 2018
CPU: ARMv7 Processor [410fc090] revision 0 (ARMv7), cr=10c5387d
CPU: PIPT / VIPT nonaliasing data cache, VIPT nonaliasing instruction cache
OF: fdt: Machine model: V2P-CA9
Memory policy: Data cache writeback
CPU: All CPU(s) started in SVC mode.
```
### 使用qemu-kvm创建的bridge virtbr0
前面步骤1，2提到将宿主局网卡enp0s3加到网桥br0上面，宿主机ubuntu需要添加default route gw 10.0.2.2才能上外网。其实这里还有一个方法：将tap0挂载到qemu-kvm创建的网桥vitbr0上面，能实现让QEMU虚拟机与宿主机通信和联通外网，也不影响宿主机联通外网。
virbr0 是 KVM 默认创建的一个 bridge，其作用是为连接其上的虚机网卡提供 NAT 访问外网的功能。virbr0 默认分配了一个IP 192.168.122.1，并为连接其上的其他虚拟网卡提供 DHCP 服务。
QEMU启动u-boot的命令：`sudo qemu-system-arm -M vexpress-a9 -m 1024 -kernel output/build/uboot-2018.05/u-boot -nographic -net nic,model=lan9118 -net tap,ifname=tap0,script=board/qemu/arm-vexpress/qemu-ifup`
网络启动脚本是qemu-ifup：
```
jgsun@VirtualBox:/repo/work/buildroot$ cat board/qemu/arm-vexpress/qemu-ifup
#!/bin/sh
cmd="sudo tunctl -u $(id -un) -t $1"
echo ${cmd}
eval ${cmd}
cmd="sudo ifconfig $1 0.0.0.0 promisc up"
echo ${cmd}
eval ${cmd}
cmd="sudo brctl addif virbr0 $1"
echo ${cmd}
eval ${cmd}
cmd="brctl show"
echo ${cmd}
eval ${cmd}
```
QEMU虚拟机linux启动之后自动获取ip 192.168.122.76，将192.168.122.1作为default route，能ping通我司主页172.24.208.168：
```
# ip addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.76/24 brd 192.168.122.255 scope global eth0
       valid_lft forever preferred_lft forever
# ip route
default via 192.168.122.1 dev eth0 
192.168.122.0/24 dev eth0 scope link src 192.168.122.76 
# ping 172.24.208.168
PING 172.24.208.168 (172.24.208.168): 56 data bytes
64 bytes from 172.24.208.168: seq=0 ttl=120 time=7.689 ms
```
启动log看出启动udhcpc获取ip
```
udhcpc: started, v1.28.4
udhcpc: sending discover
udhcpc: sending discover
udhcpc: sending select for 192.168.122.76
udhcpc: lease of 192.168.122.76 obtained, lease time 3600
deleting routers
adding dns 192.168.122.1
OK

Welcome to Buildroot
buildroot login: root
```
宿主机ubuntu的网络设置：
```
jgsun@VirtualBox:/repo/work/buildroot$ brctl show
bridge name	bridge id	STP enabled	interfaces
virbr0	8000.3a1064dd8595	yes	tap0
jgsun@VirtualBox:~$ ip route
default via 10.0.2.2 dev enp0s3 proto dhcp metric 20100 
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.16 metric 100 
169.254.0.0/16 dev enp0s3 scope link metric 1000 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown 
```
在QEMU虚拟机linux ping服务器135.251.205.177，tcpdump抓取enp0s3网口报文保存为pcap文件用wireshark打开，可以看到源地址已经被替换成enp0s3的ip 10.0.2.15
`/repo/work/buildroot$ sudo tcpdump -i enp0s3 -w /media/sf_tftpd/enp0s3.pcap`

### 挂载NFS文件系统
1.ubuntu按照nfs server
```
sudo apt-get install nfs-kernel-server
sudo mkdir -p /root/rootfs
sudo vi /etc/exports//最后一行添加"/home/jgsun/rootfs *(sync,rw,no_root_squash)" 
jgsun@VirtualBox:~$ sudo /etc/init.d/nfs-kernel-server restart
[ ok ] Restarting nfs-kernel-server (via systemctl): nfs-kernel-server.service.
```
2.QEMU虚拟机linux挂载nfs
`mount -t nfs -o nolock 192.168.122.1:/home/jgsun/work/rootfs /mnt/rootfs`
## SD卡启动
1.制作一个8M大小的SD卡文件，并将启动文件uImage和dtb拷贝到SD卡
```
dd if=/dev/zero of=sd.ext2 bs=1M count=8
mkfs.ext2 sd.ext2
mkdir tmpfs
sudo mount -t ext2 sd.ext2 tmpfs/ -o loop
sudo cp uImage vexpress-v2p-ca9.dtb tmpfs //将用uImage和dtb拷到此处
sudo umount tmpfs
``` 
2.QEMU带-sd参数启动u-boot
`sudo qemu-system-arm -M vexpress-a9 -m 1024 -kernel output/build/uboot-2018.05/u-boot  -nographic -sd output/image/sd.ext2`
ext2ls 命令显示SD卡内文件内容：

```
=> ext2ls mmc 0
<DIR> 1024 .
<DIR> 1024 ..
<DIR> 12288 lost+found
         6891184 uImage
           14318 vexpress-v2p-ca9.dtb
```

3.ext2load命令从SD卡加载uImage和dtb到内存，设置bootargs之后用bootm启动linux：

```
=> ext2load mmc 0 0x60003000 uImage
6891184 bytes read in 1188 ms (5.5 MiB/s)
=> ext2load mmc 0 0x60900000 vexpress-v2p-ca9.dtb
14318 bytes read in 59 ms (236.3 KiB/s)
=> setenv bootargs 'root=/dev/ram0 console=ttyAMA0'
=> bootm 0x60003000 - 60900000
## Booting kernel from Legacy Image at 60003000 ...
   Image Name: Linux-4.16.7
   Image Type: ARM Linux Kernel Image (uncompressed)
   Data Size: 6891120 Bytes = 6.6 MiB
   Load Address: 60003000
   Entry Point: 60003000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 60900000
```
## nor flash启动
1.制作一个8M大小的flash文件，烧写uImage和dtb文件到flash，注意bs须是4096
```
dd if=/dev/zero of=flash.bin bs=4096 count=2048//sector size 4K，count 2K
dd if=vexpress-v2p-ca9.dtb of=flash.bin conv=notrunc bs=4096//copy dtb at the beginning
dd if=uImage of=flash.bin conv=notrunc bs=4096 seek=256//copy uImage at 1M from the beginning
```
2.QEMU带-pflash参数启动u-boot

`sudo qemu-system-arm -M vexpress-a9 -m 1024 -kernel output/build/uboot-2018.05/u-boot -nographic -pflash output/image/flash.bin`

3.cp从flash拷贝linux和dtb到内存，设置bootargs之后用bootm启动linux：
```
=> cp.b 0x0 0x60900000 0x100000
=> cp.b 0x100000 0x60003000 0x700000
=> setenv bootargs 'root=/dev/ram0 console=ttyAMA0'
=> bootm 0x60003000 - 60900000
## Booting kernel from Legacy Image at 60003000 ...
   Image Name: Linux-4.16.7
   Image Type: ARM Linux Kernel Image (uncompressed)
   Data Size: 6891120 Bytes = 6.6 MiB
   Load Address: 60003000
   Entry Point: 60003000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 60900000
```
# 参考资料
* [用Qemu模拟vexpress-a9 --- 配置 qemu 的网络功能](http://www.cnblogs.com/pengdonglin137/p/5023340.html)
* [用Qemu模拟vexpress-a9 （五） --- u-boot引导kernel，device tree的使用](https://www.cnblogs.com/pengdonglin137/p/5023961.html)
* [ubuntu下使用qemu模拟ARM(七)-----uboot从sd卡启动内核](https://blog.csdn.net/rfidunion/article/details/55254424)
* [Booting Linux with U-Boot on QEMU ARM](https://balau82.wordpress.com/2010/04/12/booting-linux-with-u-boot-on-qemu-arm/)
