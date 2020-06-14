---
layout: post
title:  "有无SFP模块ethtool都返回Link detected: yes"
categories: network
tags: ethtool, DPAA, PHY, MDIO
author: jgsun
---


* content
{:toc}

# 1. 问题描述
一个转发板卡，上联口是10G SFP+接口，无论是否外接SFP模块，使用ethtool查询网卡参数都返回`Link detected: yes`，而且不插光模块的情况下，还出现了内存泄漏。本文将详细分析原因及解决方案。








```
~ # ethtool fm-mac9
Settings for eth-mgnt:
        Supported ports: [ ]
        Supported link modes: Not reported
        Supported pause frame use: No
        Supports auto-negotiation: No
        Supported FEC modes: Not reported
        Advertised link modes: Not reported
        Advertised pause frame use: No
        Advertised auto-negotiation: No
        Advertised FEC modes: Not reported
        Speed: 10000Mb/s
        Duplex: Full
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: off
        MDI-X: Unknown
        Link detected: yes
```
# 2. 系统配置
转发板卡系统模型如下所示：用户侧downstream接收wigi报文，通过bridge转发到10G网口，通过SFP uplink上传到网络侧。
![image](/images/posts/network/ethtool-link/sys-block.png)

系统cpu是NXP ls1043a，10G网口采用ls1043a自带DPAA模块，10G serdes直连SFP模块；SFP的present，LOS，fault，tx enable信号接cpu GPIO；SFP i2c接口接CPU i2c总线；DPAA模块的Frame Manager（FMan）有两个MDIO接口（内部和外部，内部MDIO用于MAC配置和管理，两个外部MDIO用于外接PHY芯片的配置和管理），我们没有外接PHY，但却在DTS给MAC配置了PHY节点“ethernet-phy-ieee802.3-c45”。系统软件和硬件框图如下图所示。用户空间设计了libsfp动态链接库管理SFP模块：用mmap配置和管理FMan内部MDIO；通过sysfs和i2c设备文件获取SFP状态。
![image](/images/posts/network/ethtool-link/sys-config.png)

以下引用自《QorIQ LS1043A Data Path Acceleration Architecture (DPAA) Reference Manual》，说明了FMan两种MDIO：
```
6.4.3.4 MDIO Ethernet Management Interface Registers
There are two types of MDIO Ethernet Management Interfaces (EMIs), internal and external. The internal 
MDIOs support the MDIO-n for EMACn port configuration and management. The external MDIOs upport 
PHY device configuration and management through dedicated MDIOs (MDIO1 and MDIO2). Both 
internal and external MDIOs use the same register formats with some differences noted in this section. See 
Table 5-17 for the base addresses for each internal and external MDIO.
```
# 3. etotool分析
Ethtool是Linux下用于查询及设置网卡参数的命令，其项目网站[ethtool - utility for controlling network drivers and hardware](https://mirrors.edge.kernel.org/pub/software/network/ethtool/)
ethtool命令使用ioctl系统调用，内核调用网络设备驱动提供的dev_ethtool。而第一章ethtool命令返回结果的最后一行“Link detected: yes”，ioctl系统调用类型是ETHTOOL_GLINK ，内核执行ethtool驱动的ethtool_op_get_link，读取__LINK_STATE_NOCARRIER，如果是0，返回真（yes）。__LINK_STATE_NOCARRIER初始化为0，然后由phy device驱动的phy_state_machine状态机定期根据物理phy的状态修改。如下图所示：
![image](/images/posts/network/ethtool-link/ethtool.png)

# 4. ethtool问题root cause分析
通过以上分析，可以得出root cause了。
当网口up的时候，首先设置网络设备状态的__LINK_STATE_START 标志位，然后phy_device驱动会启动phy状态机通过phy芯片驱动获取phy的状态（1s轮询），如果有link，就会清网络设备状态 的__LINK_STATE_NOCARRIER标志位，ethtool就是读取网络设备状态的这两个标志位来判断link状态（相与）。
dts给mac9配置的“ethernet-phy-ieee802.3-c45”的phy节点，对应的驱动是genphy_driver，其通过MDIO bus访问phy，我们的硬件没有接phy芯片，所以访问不到link真实状态（即使插了SFP），所以phy状态机永远处于网卡自协商状态，而初始化的时候__LINK_STATE_NOCARRIER标志位是0，所以当ifconfig up mac9之后，无论是否接SFP，ethtool返回的结果总是“Link detected: yes”。
# 5. memory leak root cause分析
前面提到这个问题还引起了内存泄漏，这里也分析一下原因：
AP的wigi接口收到报文，其驱动会分配skb buffer来接收，然后经过转发到mac9，dapp驱动写入发送buffer ring，启动DMA，传输完成之后产生发送完成中断，dapp驱动执行中断处理函数释放skb buffer。 
没有接SFP模块的时候：wigi接口收到arp报文，分配skb buffer，转发到mac9设备，而因为前面解释的phy状态机的问题，协议栈误以为mac9是有link的将报文发给mac9，dapp驱动将报文写入发送buffer ring，但是没有sfp模块，不会产生发送完成中断，dpaa驱动就不会释放skb buffer，所以最后引起内存泄漏。
# 6. 解决方案
在generic phy驱动中通过gpio驱动访问SFP的present和LOS信号，以获取SFP的真实状态，让phy_state_machine更新__LINK_STATE_NOCARRIER。如下图蓝线所示。
![image](/images/posts/network/ethtool-link/solution.png)

patch链接：[drivers: phy: let kernel can detect the link status of SFP](https://github.com/jgsun/jgsun.github.io/blob/master/doc/drivers-phy-let-kernel-can-detect-the-link-status-of-SFP.patch)

