---
layout: post
title: "OCTEON MIPS SMP linux多核启动"
categories: system
tags: OCTEON, SMP linux, MIPS
author: jgsun
---

* content
{:toc}

[TOC]
# 1. 概述
OCTEON MIPS C7100 SMP linux的多核启动由u-boot和linux协同完成。本文所用u-boot和linux代码库分别是u-boot-octeon-sdk3.1和linux-octeon-sdk3.1。













任何MIPS Core都是从系统的虚拟地址0xbfc00000启动的，其对应的物理地址为0x1FC00000。因为上述地址处于kseg1中，所以此时系统不需要TLB映射就能够运行（这段空间通过去掉最高的三位来获得物理地址）。

Cavium CPU根据型号不同，一个CPU里面可能集成了多个Core。这些Core里面，在上电时，只有core 0（一般称之为主核，其它核称为从核）会从reset状态跳转到物理地址0x1FC00000（我们的flash起始地址会映射成这个值），开始执行相关的一系列初始化代码，如内存、外设等等。而另外一些核则仍处于reset状态，只有core 0主动去唤醒它们时，从核才有可能开始正常运转。
 
# 2. u-boot阶段

![image](/images/posts/arch/octeon-smp-linux-uboot.png)

（1）上电后core0从跳转到物理地址 0x1fc00000，即flash 0地址开始运行 u-boot，初始化TLB，memory；

（2）core0在board_init_f阶段调用octeon_boot_bus_moveable_init将SecondaryCoreInit（ 在 u-boot-octeon-sdk3.1/ arch/mips/cpu/octeon/start.S中定义 ）代码段拷贝到bootbus模块的Local Memory Region 0（后面简称BBLMR0），并将BBLMR0的地址映射到物理地址0x1fc0 0000（写MIO_BOOT_LOC_CFG寄存器）；

（3）core0在board_init_r(DDR已经初始化，u-boot完成reolcation)阶段调用octeon_enable_all_cores函数reset从核；

（4）从核reset，跳转到物理地址0x1fc00000，也即 BBLMR0 执行 SecondaryCoreInit，读取boot vector table（第一次读取）（地址BOOT_VECTOR_BASE等于虚拟地址0x80000800，即物理地址0x800）,此时 BOOT_VECTOR_BASE还没有被初始化，从核将sleep等待NMI 唤醒 中断；

（5）core0在main_loop运行bootoctlinux命令，调用octeon_setup_boot_desc_block和octeon_setup_boot_vector 初始化各核（包括core0）的boot_desc和boot_vect；将 boot_vect的 app_start_func_addr指向 start_linux函数，code_addr指向InitTLBStart函数(注：这里 InitTLBStart被转换成物理地址，从核刚刚从reset释放到时候才能寻址，因为这个时候还没有初始化TLB，从核只能访问kseg0地址段 )。 主核（即first_core，这里是core0）的 boot_desc的flag置BOOT_FLAG_INIT_CORE，后面主核在启动linux的时候（函数start_linux）将设置寄存器a2为1，主核kernel启动时发现a2被置位将执行linux初始化，并唤醒从核。

```
/* Here we need to figure out the physical address of the
 * InitTLBStart function.
 */
boot_vect[core].code_addr =
    MAKE_XKPHYS(uboot_tlb_ptr_to_phys(&InitTLBStart));
debug("Setting core %d code_addr address(0x%p) to: 0x%llx\n",
      core, &(boot_vect[core].code_addr),
      boot_vect[core].code_addr);
```
打开debug输出，内容是`Setting core 3 code_addr address(0x80000860) to: 0x800000010f0006d0`，可见reloc code_addr address是 0x800000010f0006d0，就是core0的u-boot被从flash relocate到高端地址之后 InitTLBStart的物理地址； SecondaryCoreInit在enable 64bit寻址之后，跳转到虚拟地址 0x800000010f0006d0也就是 InitTLBStart等 物理地址0x 10f0006d0开始执行 InitTLBStart。

（6）core0给从核发送NMI中断;

（7）从核跳转到 物理地址0x1fc00000，也即 BBLMR0 执行 SecondaryCoreInit， 读取boot vector table（第二次读取），得到code_addr，跳转到 InitTLBStart，初始化TLB；

（8）从核清MNI中断，跳转到init_secondary读取 boot vector table（第三次读取），得到app_start_func_addr，跳转到 start_linux，调用asm_launch_linux_entry_point函数启动linux，跳转到 kernel_entry；

（9）core0在发生MNI中断唤醒从核之后，直接调用 app_start_func_addr也就是 start_linux启动linux，跳转到 kernel_entry。
# 3. linux阶段

![image](/images/posts/arch/octeon-smp-linux-kernel.png)

（1）主核core0 启动linux的入口是 kernel_entry（在arch/mips/kerenl/head.S中定义），跳转到kernel_entry_setup（在arch\mips\include\asm\mach-cavium-octeon\kernel-entry-init.h定义），因为BootLoader将主核core0的寄存器a2设置为1，所以跳转到octeon_main_processor，继而跳转到start_kernel继续启动linux；

（2）从核启动linux的入口也是kernel_entry，因为BootLoader将从核的寄存器a2设置为0，于是进入octeon_spin_wait_boot等待主核core0唤醒;

（3）主核core0执行start_kernel，调用boot_cpu_init，设置core0点cpu状态为online（可被 调度 ），active（可被 迁移 ）， present（ 被内核接管 ） ， possible（系统存在，但没有被内核接管）；

（4）主核 core0调用setup_arch进行mips体系结构相关初始化，与SMP相关的有：prom_init调用octeon_setup_smp注册octeon_smp_ops结构体，赋值给全局变量struct plat_smp_ops *mp_ops，是多核启动的操作函数集；plat_smp_setup调用 plat_smp_ops函数集octeon_smp_ops 的octeon_smp_setup根据coremask参数设置可以被启动点从核的各cpu状态为 present（ 被内核接管 ）和 possible（系统存在，但没有被内核接管），设置各从核cpu点logic map。

（5）主核 core0在kernel_init线程执行smp_prepare_cpus，调用octeon_smp_ops 的octeon_prepare_cpus初始化mailbox并申请mailbox中断；

（6）主核 core0执行smp_init，调用idle_threads_init初始化idle线程，然后调用cpu，最终调用octeon_smp_ops的octeon_boot_secondary，将 octeon_processor_sp和octeon_processor_gp指向idle线程的stack，将octeon_processor_sp赋值为待启动从核点逻辑core id；

（7）从核 octeon_spin_wait_boot检测到 octeon_processor_sp和自己的core id匹配了，就跳出 octeon_spin_wait_boot，执行smp_bootstrap，调用start_secondary，进而调用 octeon_smp_ops 的octeon_init_secondary和octeon_smp_finish；

（8）从核执行cpu_startup_entry进入idle loop。

# 4. 参考文档
* [SMP多核启动](https://winddoing.github.io/post/49009.html)
* 《 see mips run 》
* [BootLoader(含MIPS内存管理）](https://www.cnblogs.com/dubingsky/archive/2010/06/03/1751027.html)
* 《 Cavium多核系统地址空间布局及启动.doc 》，网络下载，文档内容不全，缺多核启动代码部分。
* Bootloader_Details_Class_Part1_R1.pptx

* Bootloader_Details_Class_Part2_R1.ppt

* Bootloader_Details_Class_Part3_R1.ppt
* Bootloader_Reference_Information_R1.pdf

