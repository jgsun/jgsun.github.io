---
layout: post
title:  "Octeon MIPS management at Linux startup"
categories: memory
tags: octeon, start_kernel, setup_arch, MIPS, memory
author: jgsun
---


* content
{:toc}

# 1. Overview
本文介绍 OCTEON 平台内存管理，包括 U-BOOT 内存管理和 Linux 启动期间的内存管理。












# 2. u-boot 内存管理
u-boot 阶段关于内存的分为：DDR 初始化，cvmx_bootmem 分配器初始化，内存分配和释放，并在 boot liunx 将bootmem descriptor（cvmx_bootmem_desc_t）地址给 Linux kernel。
## 2.1 DDR 初始化
在 board_init_f/init_sequence 阶段调用 octeon_init_dram 完成 ddr 的硬件初始化。如果 u-boot 本身从 ram 启动，failsafe_scan 将设置GD_FLG_RAM_RESIDENT 标志，如果同时设置了环境变量 "dram_size_mbytes"，octeon_init_dram 将从 "dram_size_mbytes" 获取系统内存大小，跳过 DDR 的硬件初始化，为此，core1 的 u-boot 在 default_environment 设置了环境变量 "dram_size_mbytes"。
## 2.2 cvmx_bootmem allocator
cavium octeon SDK 设计了内存分配器 bootmem，在 u-boot 和 Linux 启动阶段都使用了这个分配器，为了和内核启动阶段的bootmem分配器区别开来，取名 cvmx_bootmem。

如下图所示，cvmx_bootmem 维护一个内存 free list 的链表，链表头 head_addr 保存在 bootmem descriptor（cvmx_bootmem_desc_t）数据结构，指向系统中第一块 free 内存，每块 free 内存的前两个 64bit 存放 next 指针（指向下一块 free 内存）和本快内存 size。
![image](/images/posts/mem/mem-mips/mem-list.png)


u-boot\executive\octeon_mem_map.h 文件在低端地址定义 bootmem descriptor 地址 BOOTLOADER_BOOTMEM_DESC_SPACE，数据结构类型是 cvmx_bootmem_desc_t（在 u-boot\executive\cvmx-bootmem.h 文件定义）；
board_init_r 阶段初始化 cvmx_bootmem 分配器，将 BOOTLOADER_BOOTMEM_DESC_SPACE 作为入参调用 cvmx_bootmem_phy_mem_list_init，建立 free list 链表结构；

此后，使用 cvmx_bootmem 分配器 API 分配和释放内存就是修改这个链表的过程；最后启动 Linux 的时候将 bootmem descriptor 的地址传递给 kernel，kernel在启动阶段将遍历此链表获得其可管理的内存范围。


cvmx_bootmem_phy_mem_list_init 默认是系统内存起始地址是 0，而 core1 的起始地址是 0x40000000，这里修改了 cvmx_bootmem_phy_mem_list_init 函数。
cvmx_bootmem 的分配函数 cvmx_bootmem_phy_alloc 可以通过参数 address_min 和 address_max 指定内存分配范围。
cvmx_bootmem 还支持给分配的内存块命名，称为 named block。
## 2.3 u-boot内存管理图
下图描述了 u-boot 阶段的内存管理情况：从 ddr 初始化到启动 Linux 共分了 9 步（红色数字）
![image](/images/posts/mem/mem-mips/mem-uboot.png)


下表简单解释这 9 个步骤：

|step|描述|
|-----|-----|
|1|DDR 初始化|
|2|cvmx_bootmem 初始化|
|3|为 u-boot code 和 device tree 分配内存，这部分内存在 Linux 也不释放，kernel 将不能管理|
|4|如果 debug 打开，打印当前内存 free list|
|5|预留低端 1MB 内存空间，这段内存存放 exception vectors 等永久保存内容，bootmem descriptor 也保存在这段空间|
|6|为 device tree 指定的 memreserve 空间分配内存，prozone 空间在此分配，不释放，kernel 将不能管理。|
|7|mapped kernel，分配地址是 vmLinux image elf header 指定的物理地址 0x800000；反之，分配地址是任意地址，从低地址开始分配|
|8|给 Linux 启动参数分配空间：包括 argument data, boot descritor block 和 cvmx_bootinfo block，含 bootmem descriptor 的地址|
|9|free 名称含tmp的name block内存|


# 3. Linux启动阶段内存管理
Linux的内存管理可分为3个阶段：

|阶段|start|end|描述|
|---|---|---|---|
|1|start_kernel|bootmem_init|从系统启动到引导内存分配器 bootmem 初始化完成|
|2|bootmem_init|mm_init|引导内存分配器 bootmem 初始化完成到 buddy 初始化完成,这一阶段使用 bootmem 分配内存|
|3|mm_init|system stop|用 slab 和 buddy 分配内存|

## 3.1 setup_arch阶段工作
下图描述 Linux 启动阶段 setup_arch 完成的内存管理方面的工作：
![image](/images/posts/mem/mem-mips/mem_setup_arch.png)


下表描述主要步骤：

|step|函数|描述|
|---|---|---|
|1|prom_init|从寄存器 a3 获取 u-boot 传递参数 bootmem descriptor 地址；解析 "mem=" 启动参数，获取地址范围|
|2|plat_mem_setup|使用 cvmx_bootmem 分配器获取内核可以管理的内存范围，添加到 boot_mem_map 的 map 数组|
|3|arch_mem_addpart|将 kernel 的 code 和 data, init 代码用内存段添加到 boot_mem_map 的 map 数组|
|4|print_memory_map|打印出 boot_mem_map 的 map 数组地址范围，即 "Determined physical RAM map:"|
|5|parse_early_param|解析 early_param 参数"mem=",我们是在 prom_init 解析了，不添加 "mem=" 到 cmdline 参数列表|
|6|bootmem_init|初始化引导内存分配器 bootmem|
|7|device_tree_init|初始化 device_tree,其内存已经在 u-boot 阶段分配到 0x4008 0000|
|8|sparse_init|系统所用内存模型 sparse 初始化|
|9|plat_swiotlb_setup|software IO TLB 初始化 https://wiki.gentoo.org/wiki/IOMMU_SWIOTLB|
|a|resource_init|调用 request_resource 为 kernel 代码和内存 request and reserve memory resource|
|b|octeon_cache_init|octeon cache 初始化|
|c|paging_init|初始化分页机制，sets up the page tables, initialises the zone memory maps and sets up the zero page|


## 3.2 setup_arch之后工作
下图描述 Linux 启动阶段 setup_arch 之后到 free_initmem 的内存管理方面的工作：
![image](/images/posts/mem/mem-mips/mem-start_kernel.png)

下表描述主要步骤：

|step|函数|描述|
|---|---|---|
|1|setup_per_cpu_areas|给每个 CPU 分配内存，为系统中的每个 CPU 的 per_cpu 变量申请空间|
|2|build_all_zonelists|建立并初始化结点和内存域的数据结构|
|3|page_alloc_init|hotcpu_notifier(page_alloc_cpu_notifier, 0)|
|4|pidhash_init|根据低端内存页数和散列度，分配 hash 空间，并赋予 pid_hash|
|5|mem_init|停用 bootmem 分配器并迁移到实际的内存管理器 buddy mem_init_print_info|
|6|kmem_cache_init|初始化内核内部用于小块内存区的分配器，slab, slub 或者 slob|
|7|percpu_init_late|使用 slab 分配替换之前用 bootmem 分配的 per_cpu 变量|
|8|pgtable_init|pgtable_cache_init|
|9|vmalloc_init|非连续内存管理的初始化？|
|a|kmem_cache_init_late|在 kmem_cache_init 之后, 完善分配器的缓存机制，系统所以 slub 分配器没有定义这个函数|
|b|kmemleak_init|Kmemleak 提供了一种可选的内核泄漏检测|
|c|setup_per_cpu_pageset|初始化 CPU 高速缓存行, 为 pagesets 的第一个数组元素分配内存？|
|d|pidmap_init|PID 的位码表初始化？|
|e|anon_vma_init|匿名映射 vma 初始化|
|f|free_initmem|释放 kernel init 代码和数据|

# 4. 参考文档
* [bootmem allocator分析](https://blog.csdn.net/kris_fei/article/details/8703433)
* [深入理解Linux内存管理-之-目录导航](https://blog.csdn.net/gatieme/article/details/52384965)
* [Linux内存源码分析 - 伙伴系统(初始化和申请页框)](https://www.cnblogs.com/tolimit/p/4610974.html)

