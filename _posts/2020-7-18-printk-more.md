---
layout: post
title:  "printk输出线程名，线程id，cpuid和prio的实现"
categories: debug
tags: printk, comm, cpu id, pid, prio
author: jgsun
---


* content
{:toc}

# 1. Overview
我们知道，ftrace输出含有线程名，线程id和cpuid，非常有助于分析问题，是否可以让prink输出也带有这些信息的？本文介绍了其实现方法。

![image](/images/posts/debug/printk/printk.jpg)






*  ftrace 输出格式
```
        modprobe-514 [001] .... 177.645097: cpld_irq_unmask: hwirq 2, virq 26
        modprobe-514 [001] .... 177.645103: cpld_irq_unmask: ire=0x15, irs=0x16, ina=0x0
        modprobe-514 [001] .... 177.645105: cpld_irq_unmask: hwirq 2, virq 26, idata->mask 0x4
        modprobe-514 [001] .... 177.645106: cpld_irq_unmask: cpld_ic->masked=0xff, cpld_ic->started=0x0
        modprobe-514 [001] .... 177.645141: cpld_irq_unmask: cpld_ic->masked=0xfb, cpld_ic->started=0x4
```
# 2. 代码修改
![image](/images/posts/debug/printk/kconfig-comm.png)


补丁link: [0001-kernel-printk-print-comm-cpuid-pid-and-prio.patch](https://github.com/jgsun/jgsun.github.io/blob/master/doc/0001-kernel-printk-print-comm-cpuid-pid-and-prio.patch)

# 3. printk输出格式效果
```
[ 0.000000] ( swapper-0 [00] 20) Booting Linux on physical CPU 0x0000000000 [0x411fd070]
[ 0.000000] ( swapper-0 [00] 20) Linux version 5.6.11 (jgsun@VirtualBox) (gcc version 9.2.1 20191025 (GNU Toolchain for the A-profile Architecture 9.2-2019.12 (arm-9.10))) #2 SMP Sat Jul 18 15:08:23 CST 2020
[ 0.000000] ( swapper-0 [00] 20) Machine model: linux,dummy-virt
[ 0.000000] ( swapper-0 [00] 20) efi: Getting EFI parameters from FDT:
[ 0.000000] ( swapper-0 [00] 20) efi: UEFI not found.
[ 0.000000] ( swapper-0 [00] 20) On node 0 totalpages: 131072
[ 0.000000] ( swapper-0 [00] 20) DMA zone: 2048 pages used for memmap
[ 0.000000] ( swapper-0 [00] 20) DMA zone: 0 pages reserved
[ 0.000000] ( swapper-0 [00] 20) DMA zone: 131072 pages, LIFO batch:31
[ 0.000000] ( swapper-0 [00] 20) psci: probing for conduit method from DT.
[ 0.000000] ( swapper-0 [00] 20) psci: PSCIv0.2 detected in firmware.
[ 0.000000] ( swapper-0 [00] 20) psci: Using standard PSCI v0.2 function IDs
[ 0.000000] ( swapper-0 [00] 20) psci: Trusted OS migration not required
```
# 4. CONFIG_PRINTK_CALLER: Show caller information on printks
![image](/images/posts/debug/printk/kconfig-caller.png)

kernel配置选项`Show caller information on printks`也可以打印出线程id，但是我们的解决方案打印的更加全面，不仅包括线程id，还可以包含线程名，cpuid和优先级prio，也许也可以考虑把这个patch推到社区让更多人受益。


```
[ 0.000000][ T0] ( swapper-0 [00] 20) Booting Linux on physical CPU 0x0000000000 [0x411fd070]
[ 0.000000][ T0] ( swapper-0 [00] 20) Linux version 5.6.11 (jgsun@VirtualBox) (gcc version 9.2.1 20191025 (GNU Toolchain for the A-profile Architecture 9.2-2019.12 (arm-9.10))) #4 SMP Sat Jul 18 16:57:53 CST 2020
[ 0.000000][ T0] ( swapper-0 [00] 20) Machine model: linux,dummy-virt
[ 0.000000][ T0] ( swapper-0 [00] 20) efi: Getting EFI parameters from FDT:
[ 0.000000][ T0] ( swapper-0 [00] 20) efi: UEFI not found.
[ 0.000000][ T0] ( swapper-0 [00] 20) On node 0 totalpages: 131072
[ 0.000000][ T0] ( swapper-0 [00] 20) DMA zone: 2048 pages used for memmap
[ 0.000000][ T0] ( swapper-0 [00] 20) DMA zone: 0 pages reserved
[ 0.000000][ T0] ( swapper-0 [00] 20) DMA zone: 131072 pages, LIFO batch:31
```