---
layout: post
title:  "win10/virtualbox/ubuntu虚拟机套娃"
categories: system
tags: VT-x, vtx, virt-nest, virtualbox, ubuntu, KVM
author: jgsun
---


* content
{:toc}
# 1. Overview
去年想在win10/VisualBox运行的ubuntu搭建KVM实验环境，需要 VisualBox启用`嵌套VT-x/AMD-V`，无奈当初VisualBox 5.x版本这个选项是灰的；在网上查了一圈，说VirtualBox已经支持嵌套AMD-V，但是VT-x比较复杂，正在开发，于是作罢。
到了今年，VisualBox也升级到6.x，但是`嵌套VT-x/AMD-V`还是会的，于是以关键字“VisualBox 嵌套VT-x/AMD-V 灰的”上百度搜索，从这篇文章[在VirtualBox 6.1里面打开嵌套 VT-x/AMD-V 功能](https://blog.csdn.net/holderlinzhang/article/details/104260531?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)找到了答案。














# 2. 使用VBoxManage enable nested-hw-virt
按这篇文章的方法使用VBoxManage 命令行的方式来打开这个选项之后，启动VirtualBox，发现`嵌套VT-x/AMD-V`选项已经可以用了。
```
C:\Users\jiangusu>cd "c:\Program Files\Oracle\VirtualBox"

c:\Program Files\Oracle\VirtualBox>VBoxManage.exe list vms
"u18" {4737eb1b-d5ad-4178-813e-5891f31095ee}

c:\Program Files\Oracle\VirtualBox>VBoxManage.exe modifyvm "u18" --nested-hw-virt on
```
![image](/images/posts/virtualize/ubutu_setting.png)


# 3. 升级VirtualBox到最新版本
启动ubuntu虚拟机，/proc/cpuinfo还是看不到vmx flag；当前所有VirtualBox版本是6.0，再将VirtualBox升级到最新版本6.1.8之后，果然可以看见vmx flag了。可见，VirtualBox确实是在6.0之后的版本才支持VT-x。

![image](/images/posts/virtualize/vb.png)


```
jgsun@VirtualBox:~/repo/buildroot$ cat /proc/cpuinfo | grep vmx
flags : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single pti tpr_shadow flexpriority fsgsbase avx2 invpcid rdseed clflushopt md_clear flush_l1d
flags : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single pti tpr_shadow flexpriority fsgsbase avx2 invpcid rdseed clflushopt md_clear flush_l1d
```
实现了虚拟机套娃，下一步就是运行QEMU X86_64虚拟机来进行KVM实践。