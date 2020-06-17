---
layout: post
title:  "Octeon MIPS management at linux startup"
categories: memory
tags: octeon, start_kernel, setup_arch, MIPS, memory
author: jgsun
---


* content
{:toc}

# 1. Overview
本文介绍octeon平台内存管理，包括u-boot内存管理和linux启动期间的内存管理。












# 2. u-boot内存管理
u-boot阶段关于内存的分为：DDR初始化，cvmx_bootmem分配器初始化，内存分配和释放，并在boot liunx将bootmem descriptor（cvmx_bootmem_desc_t）地址给linux kernel。
## 2.1 DDR初始化
在board_init_f/init_sequence阶段调用octeon_init_dram完成ddr的硬件初始化。如果u-boot本身从ram启动，failsafe_scan将设置GD_FLG_RAM_RESIDENT标志，如果同时设置了环境变量"dram_size_mbytes"，octeon_init_dram将从"dram_size_mbytes"获取系统内存大小，跳过DDR的硬件初始化，为此，core1的u-boot在default_environment 设置了环境变量"dram_size_mbytes"。
## 2.2 cvmx_bootmem allocator
cavium octeon SDK设计了内存分配器bootmem，在u-boot和linux启动阶段都使用了这个分配器，为了和内核启动阶段的bootmem分配器区别开来，取名cvmx_bootmem。
如下图所示，cvmx_bootmem维护一个内存free list的链表，链表头head_addr保存在bootmem descriptor（cvmx_bootmem_desc_t）数据结构，指向系统中第一块free内存，每块free内存的前两个64bit存放next指针（指向下一块free内存）和本快内存size。u-boot\executive\octeon_mem_map.h文件在低端地址定义bootmem descriptor地址BOOTLOADER_BOOTMEM_DESC_SPACE，数据结构类型是cvmx_bootmem_desc_t（在u-boot\executive\cvmx-bootmem.h文件定义）；board_init_r阶段初始化cvmx_bootmem分配器，将BOOTLOADER_BOOTMEM_DESC_SPACE作为入参调用cvmx_bootmem_phy_mem_list_init，建立free list链表结构；此后，使用cvmx_bootmem分配器API分配和释放内存就是修改这个链表的过程；最后启动linux的时候将bootmem descriptor的地址传递给kernel，kernel在启动阶段将遍历此链表获得其可管理的内存范围。
![image](/images/posts/mem/mem-mips/mem-list.pnt)


cvmx_bootmem_phy_mem_list_init默认是系统内存起始地址是0，而core1的起始地址是0x40000000，这里修改了cvmx_bootmem_phy_mem_list_init函数。
cvmx_bootmem的分配函数cvmx_bootmem_phy_alloc可以通过参数address_min和address_max指定内存分配范围。
cvmx_bootmem还支持给分配的内存块命名，称为named block。
## 2.3 u-boot内存管理图
下图描述了u-boot阶段的内存管理情况：从ddr初始化到启动linux共分了9步（红色数字）
![image](/images/posts/mem/mem-mips/mem-uboot.png)


下表简单解释这9个步骤：
|step|描述|
|-----|-----|
|1|DDR初始化|
|2|cvmx_bootmem初始化|
|3|为u-boot code和device tree分配内存，这部分内存在linux也不释放，kernel将不能管理|
|4|如果debug打开，打印当前内存free list|
|5|预留低端1MB内存空间，这段内存存放exception vectors等永久保存内容，bootmem descriptor也保存在这段空间|
|6|为device tree指定的memreserve空间分配内存，prozone空间在此分配，不释放，kernel将不能管理。|
|7|mapped kernel，分配地址是vmlinux image elf header指定的物理地址0x800000；反之，分配地址是任意地址，从低地址开始分配|
|8|给linux启动参数分配空间：包括agrument data, boot descritor block 和cvmx_bootinfo block，含bootmem descriptor的地址|
|9|free 名称含tmp的name block内存|


# 3. linux启动阶段内存管理
linux的内存管理可分为3个阶段：

|阶段|start|end|描述|
|---|---|---|---|
|1|start_kernel|bootmem_init|从系统启动到引导内存分配器bootmem初始化完成|
|2|bootmem_init|mm_init|引导内存分配器bootmem初始化完成到buddy初始化完成,这一阶段使用bootmem分配内存|
|3|mm_init|system stop|用slab和buddy分配内存|

## 3.1 setup_arch阶段工作
下图描述linux启动阶段setup_arch完成的内存管理方面的工作：
![image](/images/posts/mem/mem-mips/mem_setup_arch.png)


下表描述主要步骤：
|step|函数|描述|
|---|---|---|
|1|prom_init|从寄存器a3获取u-boot传递参数bootmem descriptor地址；解析"mem="启动参数，获取地址范围|
|2|plat_mem_setup|使用cvmx_bootmem分配器获取内核可以管理的内存范围，添加到boot_mem_map的map数组|
|3|arch_mem_addpart|将kernel的code和data,init代码用内存段添加到boot_mem_map的map数组|
|4|print_memory_map|打印出boot_mem_map的map数组地址范围，即"Determined physical RAM map:"|
|5|parse_early_param|解析early_param参数"mem=",我们是在prom_init解析了，不添加"mem="到cmdline参数列表|
|6|bootmem_init|初始化引导内存分配器bootmem|
|7|device_tree_init|初始化device_tree,其内存已经在u-boot阶段分配到0x4008 0000|
|8|sparse_init|系统所用内存模型sparse初始化|
|9|plat_swiotlb_setup|software IO TLB初始化 https://wiki.gentoo.org/wiki/IOMMU_SWIOTLB|
|a|resource_init|调用request_resource为kernel代码和内存request and reserve memory resource|
|b|octeon_cache_init|octeon cache初始化|
|c|paging_init|初始化分页机制，sets up the page tables, initialises the zone memory maps and sets up the zero page|


## 3.2 setup_arch之后工作
下图描述linux启动阶段setup_arch之后到free_initmem的内存管理方面的工作：
![image](/images/posts/mem/mem-mips/mem-start_kernel.png)

下表描述主要步骤：

|step|函数|描述|
|---|---|---|
|1|setup_per_cpu_areas|给每个CPU分配内存，为系统中的每个CPU的per_cpu变量申请空间|
|2|build_all_zonelists|建立并初始化结点和内存域的数据结构|
|3|page_alloc_init|hotcpu_notifier(page_alloc_cpu_notifier, 0)|
|4|pidhash_init|根据低端内存页数和散列度，分配hash空间，并赋予pid_hash|
|5|mem_init|停用bootmem分配器并迁移到实际的内存管理器buddy mem_init_print_info|
|6|kmem_cache_init|初始化内核内部用于小块内存区的分配器，slab,slub或者slob|
|7|percpu_init_late|使用slab分配替换之前用bootmem分配的per_cpu变量|
|8|pgtable_init|pgtable_cache_init|
|9|vmalloc_init|非连续内存管理的初始化？|
|a|kmem_cache_init_late|在kmem_cache_init之后, 完善分配器的缓存机制，系统所以slub分配器没有定义这个函数|
|b|kmemleak_init|Kmemleak提供了一种可选的内核泄漏检测|
|c|setup_per_cpu_pageset|初始化CPU高速缓存行, 为pagesets的第一个数组元素分配内存？|
|d|pidmap_init|PID的位码表初始化？|
|e|anon_vma_init|匿名映射vma初始化|
|f|free_initmem|释放kernel init代码和数据|

# 4. 参考文档
* [bootmem allocator分析](https://blog.csdn.net/kris_fei/article/details/8703433)
* [深入理解Linux内存管理-之-目录导航](https://blog.csdn.net/gatieme/article/details/52384965)
* [linux内存源码分析 - 伙伴系统(初始化和申请页框)](https://www.cnblogs.com/tolimit/p/4610974.html)

