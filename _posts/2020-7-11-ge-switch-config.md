---
layout: post
title:  "GE switch配置分析"
categories: network
tags: SMI, MDIO, GE AVB switch, mdio bus
author: jgsun
---


* content
{:toc}

[TOC]
# 1. Overview
某网络板卡使用两个switch芯片，一个是7 ports GE switch 88E6321主管控制面（下文简称小switch），另外一个switch 98DX8324主管数据面（下文简称大switch），硬件框图如下。
![image](/images/posts/network/ge-switch/hardware-block.png)












从硬件框图可以看出，OBC通过SMI(System Management Intetface)接口控制小switch芯片；OBC通过PCIE接口控制大switch芯片；CPU的RGMII接口连接小switch的P5，内部转发至P3向外提供debug网口； CPU的SGMII接口接小switch的P1，大switch SGMII接小switch的p0接口，经小siwich交换转发到P4口向往提供OAM管理接口。
大小switch从网络层面都有很复杂的配置，本文仅从平台角度出发介绍小switch的配置，包括uboot和linux下的配置。
* 7 Port AVB GE Switch: 88E6321-A0-NAZ2I000
* about AVB: Audio Video Bridging (AVB) is a common name for the set of technical standards which provide improved synchronization, low-latency, and reliability for switched Ethernet networks. 详见wiki： [Audio Video Bridging](https://en.wikipedia.org/wiki/Audio_Video_Bridging)
# 2. 小switch SMI MDIO地址
OBC使用SMI MDIO协议读写小switch的寄存器，根据MDIO协议，phy device address是5bit，所以一条MDIO bus可以访问32个phy device，小switch就根据这个范围编址，让CPU可以访问port寄存器和phy寄存器。
硬件连接选择的是single-chip address mode，其寄存器map如下图所示，如地址0x15访问port5的port register。
![image](/images/posts/network/ge-switch/device-register-map.png)


* [MDIO Background](https://www.totalphase.com/support/articles/200349206-MDIO-Background)
![image](/images/posts/network/ge-switch/c22-mdio.png)
![image](/images/posts/network/ge-switch/c45-mdio.png)



# 3. u-boot配置
uboot调试阶段，需要使用debug网口。
## 3.1 force p5 link up
源码位置：<\uboot\board\octeon\lmnta\board.c>
board_init_r -> late_board_init
```
int late_board_init(void)
{
    struct mii_dev *bus = mdio_get_current_dev();
    bus->write(bus, 0x15, -1, 0x1, 0x4013);
    bus->write(bus, 0x15, -1, 0x1, 0x4033);

    /* set alternate led mode */
    bus->write(bus, 0x13, -1, 0x16, 0xE733);
    return 0;
}
```
```
mii write 0x15 1 0x4013 #set bit 14 to 1, bit 4 to 1, bit 5 to 0, 
of register 1 of port 5, in order to make the link down and add delay. 
May be a little delay need to be added between the two commands when we involve them into code.
mii write 0x15 1 0x4033 #set the link up.
```
## 3.2 u-boot MDIO初始化流程
```
board_init_r 
    eth_initialize
        cpu_eth_init
            octeon_mdio_initialize
                mdio_register
```

# 4. Kernel driver
![image](/images/posts/network/ge-switch/kernel-mdio.png)

kernel驱动由3部分组成：octeon mdio驱动，mdio bus核心驱动和expose mdio to userspace驱动。octeon mdio驱动向mdio bus核心驱动注册octeon mdio bus设备，将mem地址作为bus->id；expose mdio to userspace驱动根据mem地址找到octeon mdio bus设备，以调用mdio bus的读写函数cavium_mdio_read/cavium_mdio_write读写小switch寄存器；expose mdio to userspace驱动向用户空间创建misc字符设备。用户空间程序switch_management_app通过此misc字符设备配置小swithc的转发规则。
用户程序switch_management_app通过ioctl/write系统调用配置小switch，

## 4.1 dirver 对应DTS
```
        smi0: mdio@1180000001800 {
            compatible = "cavium,octeon-3860-mdio";
            #address-cells = <1>;
            #size-cells = <0>;
            reg = <0x11800 0x00001800 0x0 0x40>;
        };

        mdio0: mdio@0 {
            compatible = "expose-mdio";
            mdiobus-id = "8001180000001800";
            protocol = "C22";
            phy-id = <0x10 0x11 0x12 0x13 0x14 0x15 0x16 0x1b 0x1c 0x1d>;
        };

```

## 4.2 MDIO device driver init
drivers\net\phy\mdio-octeon.c
drivers/net/phy/mdio-cavium.c
```
oct_mdio_writeq(smi_en.u64, bus->register_base + SMI_EN)
of_mdiobus_register(bus->mii_bus, pdev->dev.of_node) </drivers/of/of_mdio.c>
        mdiobus_register
            device_register(&bus->dev)
            phydev = mdiobus_scan(bus, i)
            
```
启动log：
```
[ 513.283688] (c1 swapper/0 <1>) libphy: mdio_octeon: probed
[ 513.289726] (c1 swapper/0 <1>) mdio_octeon 1180000001800.mdio: Probed
[ 513.297046] (c1 swapper/0 <1>) libphy: Fixed MDIO Bus: probed
```





