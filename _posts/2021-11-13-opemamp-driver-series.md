---
layout: post
title:  "OpenAMP： remoteproc，RPMsg/Virtio driver 系列"
categories: kernel driver
tags: OpenAMP remoteproc RPMsg/Virtio
author: jgsun
---


* content
{:toc}

# 1. Overview
项目需要实现 AMP 系统，在一个 4 核的 SoC(System on a Chip) 处理器上面运行 Linux 和 VxWorks 两个操作系统： core-0/2/3 运行 Linux OS， core-1 运行 VxWorks OS， 如下图所示：
![image](/images/posts/amp/core-deployment.png)













采用 Open Asymmetric Multi-Processing (OpenAMP) framework 的 Linux kernel remoteproc driver 启动 core1 VxWorks 系统， 使用 Linux kernel RPMsg/Virtio driver 实现两个操作系统之间通讯。分 5 个 kernel module 实现，分别是 remoteproc platform diver，Mailbox interrupt controller driver， Virtio platform driver， RPMsg user interface driver 和 mailbox interface driver.

>> about OpenAMP
OpenAMP (Open Asymmetric Multi-Processing) is a framework providing the software components needed to enable the development of software applications for AMP systems. It allows operating systems to interact within a broad range of complex homogeneous and heterogeneous architectures and allows asymmetric multiprocessing applications to leverage parallelism offered by the multicore configuration.
OpenAMP (Open Asymmetric Multi-Processing) seeks to standardize the interactions between operating environments in a heterogeneous embedded system through open source solutions for Asymmetric MultiProcessing (AMP).

* <https://www.openampproject.org/>
* <https://github.com/OpenAMP/open-amp>

# 2. Remoteproc platform diver
![image](/images/posts/amp/rproc-platform.png)

# 3. Mailbox interrupt controller driver
![image](/images/posts/amp/mailbox-irq-controller.png)

# 4. Virtio platform driver 和 RPMsg user interface driver
![image](/images/posts/amp/rpmsg-platform-interface.png)

## 4.1 RPMsg/Virtio probe
![image](/images/posts/amp/rpmsg-probe.png)

# 5. Mailbox interface driver
![image](/images/posts/amp/maibox-interface.png)







