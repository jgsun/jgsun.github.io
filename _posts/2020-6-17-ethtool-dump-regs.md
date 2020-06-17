---
layout: post
title:  "ethtool dump网卡寄存器"
categories: network
tags: ethtool, dump regster, encx24j600
author: jgsun
---


* content
{:toc}

# 1.问题描述
有一块板卡，外接SPI小网卡用做调试网口，如下图所示，某宝有售。对于嵌入式系统，为节约板卡面积和降低成本，经常这样搭配，将产品用不着的调试网卡外接，虽然可能带来一些调试问题（比如连接不可靠等）。这不，问题来了，老外换了一台新电脑，发现有时候电脑连不通调试网口，于是提了个FR `LEMI sometimes not actived`(LEMI 是Local Ethernet Management Interface的简称，即调试网卡)给上海的platform来解决。很奇怪，上海板卡却没有这一问题。











![image](/images/posts/network/ethtool-dump-regs/netcard.jpg)

# 2. 问题分析
## 2.1 dump网卡寄存器
遇到这种问题，第一想到的是dump出网卡的寄存器。问题来了，怎么dump出寄存器呢？"Talk is cheap, show me the code." 
首先看看网卡驱动`/drivers/net/ethernet/microchip/encx24j600.c`是否提供这一功能，还真有：
```
static const struct ethtool_ops encx24j600_ethtool_ops = {
 .get_settings = encx24j600_get_settings,
 .set_settings = encx24j600_set_settings,
 .get_drvinfo = encx24j600_get_drvinfo,
 .get_msglevel = encx24j600_get_msglevel,
 .set_msglevel = encx24j600_set_msglevel,
 .get_regs_len = encx24j600_get_regs_len,
 .get_regs = encx24j600_get_regs,
};
```
驱动ethtool_ops的encx24j600_get_regs函数可以dump寄存器，即用户空间命令ethtool就可以dump网卡寄存器。
```
/sys/kernel/debug # ethtool -d eth-mgnt
[ 1835.500841] encx24j600: register dump
[ 1835.503916] encx24j600 PHCON1: 1200
[ 1835.508033] encx24j600 PHCON2: 0002
[ 1835.511000] encx24j600 PHANA: 05E1
[ 1835.515027] encx24j600 PHANLPA: 05E1
[ 1835.518454] encx24j600 PHANE: 0003
[ 1835.522003] encx24j600 PHSTAT1: 782D
[ 1835.525644] encx24j600 PHSTAT2: 001A
[ 1835.529095] encx24j600 PHSTAT3: 1058
[ 1835.532273] encx24j600 ECON1: 0001
[ 1835.535846] encx24j600 ECON2: CB00
[ 1835.539334] encx24j600 ERXFCON: 004B
[ 1835.542961] encx24j600 ESTAT: 5F00
[ 1835.546443] encx24j600 EIR: 0780
[ 1835.549950] encx24j600 EIDLED: CB21
[ 1835.553527] encx24j600 EIE: 884C
[ 1835.557035] encx24j600 ERXST: 0600
[ 1835.560582] encx24j600 ERXTAIL: 1CAC
[ 1835.564254] encx24j600 ERXHEAD: 1CAE
[ 1835.567686] encx24j600 ERXRDPT: 1BF2
[ 1835.571214] encx24j600 EUDAST: 6000
[ 1835.574798] encx24j600 EUDAND: 6001
[ 1835.578347] encx24j600 EDMACS: 0000
[ 1835.581835] encx24j600 MACON1: 0009
[ 1835.585409] encx24j600 MACON2: 40B3
[ 1835.588922] encx24j600 MAIPG: 0C12
[ 1835.592469] encx24j600 MACLCON: 370F
[ 1835.596034] encx24j600 MABBIPG: 0015
Offset Values
------ ------
0x0000: 00 00 00 00 00 00 00 00 00 06 00 00 ac 1c 00 00 
0x0010: ae 1c 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0020: 00 00 00 00 00 00 00 00 00 00 00 00 00 60 00 00 
0x0030: 01 60 00 00 00 5f 00 00 80 07 00 00 01 00 00 00 
0x0040: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0050: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0060: 00 00 00 00 00 00 00 00 4b 00 00 00 00 00 00 00 
0x0070: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
0x0080: 09 00 00 00 b3 40 00 00 15 00 00 00 12 0c 00 00 
0x0090: 0f 37 00 00 ee 05 00 00 00 00 00 00 00 00 00 00
```
## 2.2 寄存器对比
远程访问老外的平台，dump故障时网卡寄存器，和正常时的寄存器对比，发现PHCON2寄存器不同
name	normal	abnormal	differnece
PHCON2	2002	2000	bit 1 EDSTAT: Energy Detect Status bit
1 = Energy detect circuit has detected energy on the TPIN+/- pins within the last 256 ms
 = No energy has been detected on the TPIN+/- pins within the last 256 ms

## 2.3 root cause
查看网卡data sheet，找到原因，网卡使能了energy_detect_powerdown模式，需要等待远端设备唤醒；如果不幸远端设备也使能了这个模式，那样双方可能都陷入千年的等待。。。老外正是遇到了这种情况。
```
/isam/slot_default/run # ethtool --show-priv-flags eth-mgnt
Private flags for eth-mgnt:
use_irq : on
test_spi : off
energy_detect_powerdown: on
```
The lemi chip works in a mode called “Energy Detect Power-Down”, In this mode, the PHY remains powered down until a signal is detected on the Ethernet interface. When a signal is detected on the Ethernet medium, the EDSTAT flag (PHCON2<1>) is set. The problem is that the EDSTAT flag (PHCON2<1>) is not set when plug-in. 
We can solve this issue by disable this mode by command “ethtool --set-priv-flags eth-mgnt energy_detect_powerdown off”.
![image](/images/posts/network/ethtool-dump-regs/datasheet.png)
 


# 3. 解决方案
使用ethtool命令关闭网卡energy_detect_powerdown模式，你一个调试网卡，就不用凑energy_detect_powerdown的热闹了。
```
/isam/slot_default/run # ethtool --show-priv-flags eth-mgnt
Private flags for eth-mgnt:
use_irq : on
test_spi : off
energy_detect_powerdown: on
/isam/slot_default/run # ethtool --set-priv-flags eth-mgnt energy_detect_powerdown off
```