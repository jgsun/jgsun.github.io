---
layout: post
title:  "cpld irq controller Linux 驱动设计"
categories: system_io
tags: CPLD, irq controller, generic irqchip
author: jgsun
---


* content
{:toc}

# 1. Overview
CPLD是嵌入式板卡常用的一种可编程器件，其通过并行总线，SPI或者I2C和OBC(On Board Controller)相连，用于扩展GPIO（LED灯，中断，报警）或者低速IO总线（如SPI/I2C总线的PLL时钟、EEPROM和温感芯片等，UART等）。OBC通过CPLD fireware提供寄存器访问控制CPLD所接各种外设。
CPLD linux内核驱动需要自行设计，提供访问cpld寄存器和处理外设中断两大核心功能，本文介绍了一种cpld中断控制器驱动的设计方法，基于内核generic irqchip来实现。











# 2.  Linux中断控制器驱动系统框图

![image](/images/posts/cpld/cpld-irq.png)





