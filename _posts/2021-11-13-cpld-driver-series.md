---
layout: post
title:  "CPLD Linux Kernel driver"
categories: I/O
tags: CPLD irq-controller generic-irqchip irq-domain
author: jgsun
---


* content
{:toc}

# 1. Overview
CPLD 是嵌入式板卡常用的一种可编程器件，其通过并行总线，SPI 或者 I2C 与 OBC(On Board Controller) 相连，用于扩展 GPIO（LED灯，中断，报警）或者低速 IO 总线（如 SPI/I2C 总线的 PLL 时钟、EEPROM 和温感芯片等，UART 等）。OBC 通过 CPLD fireware 提供寄存器访问控制 CPLD 所接各种外设。
下图是某板卡 cpld hw block diagram， 使用 SPI bus 连接 OBC， 其中 I2C bush 用于 cpld program 其自带 nor flash。
![image](/images/posts/cpld/cpld-hw-block.png)












CPLD linux 内核驱动需要自行设计，提供访问 cpld 寄存器和处理外设中断两大核心功能。CPLD 驱动采用模块化解耦设计， 分 5 个 kernel module 实现， 分别是 cpld spi device driver， cpld irq controller driver, uio irq driver, cpld misc driver 和 cpld hotplug driver。
# 2. cpld spi device driver
![image](/images/posts/cpld/cpld-spi-dev.png)

# 3. cpld irq controller driver
cpld中断控制器驱动基于内核generic irqchip实现。
![image](/images/posts/cpld/cpld-irq-controller.png)

# 4. uio irq driver
![image](/images/posts/cpld/uio-irq.png)

# 5. cpld misc driver
![image](/images/posts/cpld/cpld-misc.png)

# 6. cpld hotplug driver
![image](/images/posts/cpld/cpld-hotplug.png)







