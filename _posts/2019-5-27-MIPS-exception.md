---
layout: post
title: "OCTEON MIPS 中断处理"
categories: interrupt
tags: MIPS, OCTEON, exeception, interrupt 
author: jgsun
---

* content
{:toc}
# 1. 概述
在MIPS中，中断、陷阱、系统调用和任何可以中断程序正常执行流的情况都称异常。 本文讲述OCTEON CN71xx平台的异常初始化和中断异常处理流程， 所用u-boot和linux代码库分别是u-boot-octeon-sdk3.1和linux-octeon-sdk3.1。











# 2. 异常初始化
为了让中断正常工作，操作系统必须进行必要的初始化，包括exception初始化，中断子系统初始化，中断控制器初始化，使能本地cpu中断和使能中断控制器外设中断共5个步骤，如下图所示。这些初始化建立中断向量入口，创建并初始化核心数据结构：struct irq_desc, struct irq_chip, struct irq_domain等，使能硬件中断。
![image](/images/posts/interrupt/octeon-ex-init.png)
## 2.1 exception向量初始化
这个步骤包含EBase寄存器初始化和异常向量数组初始化。
### 2.1.1  Root.EBase 寄存器初始化
当Root.Status[BEV] = 0时，Root.EBase 寄存器设定异常入口地址，其 初始化在u-boot和linux两个阶段完成：
（1）u-boot阶段
在do_bootoctlinux调用octeon_setup_boot_desc_block设置boot参数 boot_info_block_array[core].exception_base：
`boot_info_block_array[core].exception_base = cur_exception_base`
cur_exception_base等于多少？从下面的代码片段得出其等于0x1000（ 4K对齐 ） 。
```
uint32_t cur_exception_base = EXCEPTION_BASE_INCR;
/* Increment size for exception base addresses (4k minimum) */
#ifndef EXCEPTION_BASE_INCR
#define EXCEPTION_BASE_INCR     (4*1024)
#endif
```
接着do_bootoctlinux/start_linux调用 start_os_common写 Root.EBase 寄存器：

```
start_os_common
    boot_info_ptr = &boot_info_block_array[core_num]
    set_except_base_addr(boot_info_ptr->exception_base)
        write_c0_ebase(addr) //将0x1000写入 Root.EBase
```
u-boot在 do_bootoctlinux函数中，先调用 octeon_setup_boot_desc_block将每个core 启动参数的 exception_base 设置为 0x1000；然后调用 start_linux/ start_os_common将 exception_base写入Root.EBase Register；所以此时 core0  Root.EBase Register 的值是 0xffffffff80001000， core2  Root.EBase Register 的值是 0xffffffff80001002，3 core  Root.EBase Register 的值是 0xffffffff80001003。（见linux阶段trap_init函数的打印）
（2） linux阶段
主核读取 EBase 寄存器确定异常入口地址，以用于将异常handler拷贝到该地址和初始化从核 EBase 寄存器。
```
#define CKSEG0	_CONST64_(0xffffffff80000000)
ebase = CKSEG0 // ebase=0xffffffff80000000
ebase += (read_c0_ebase() & 0x3ffff000) //加入u-boot写入的0x1000之后 ebase=0xffffffff80001000
```
###  2.1.2  start_kernel/ trap_init
主核core0启动阶段调用start_kernel/ trap_init，主要完成：
（1）set  ebase全局变量,等于 CKSEG0加上u-boot阶段写入的offset 0x1000，即 0xffffffff80001000
（2） per_cpu_trap_init配置Root.Status寄存器(clear BEV)和计算 系统timer中断号码cp0_compare_irq等于7
（3）set_except_vector(0, handle_int)等，初始化异常向量数组exception_handlers[32]，中断是0，handler是 handle_int
（4） set_handler(0x180, &except_vec3_generic, 0x80)，拷贝异常入口函数 except_vec3_generic到 ebase寄存器设定的入口地址
```
#define CKSEG0			_CONST64_(0xffffffff80000000)
ebase = CKSEG0 // ebase=0xffffffff80000000
ebase += (read_c0_ebase() & 0x3ffff000) //加入u-boot写入的0x1000之后 ebase=0xffffffff80001000
per_cpu_trap_init(true)
set_except_vector(23, handle_watch)
set_except_vector(0, using_rollback_handler() ? rollback_handle_int: handle_int)
    old_handler = xchg(&exception_handlers[n], handler)
set_except_vector(1, handle_tlbm)
set_handler(0x180, &except_vec3_generic, 0x80)
    memcpy((void *)(ebase + offset), addr, size)
```
trap_init的打印输出：
```
[    0.000000] trap_init(1902)ebase=0xffffffff80000000, c0_ebase=0xffffffff80001000
[    0.000000] trap_init(1908)ebase=0xffffffff80001000, c0_ebase=0xffffffff80001000
[    0.000000] trap_init(1911)ebase=0xffffffff80001000, c0_ebase=0xffffffff80001000
[    0.000000] 0-per_cpu_trap_init(1787)ebase=0xffffffff80001000, c0_ebase=0xffffffff80001000
[    0.000000] trap_init(1913)ebase=0xffffffff80001000, c0_ebase=0xffffffff80001000
[   37.130784] SMP: Booting CPU01 (CoreId  2)...
[   37.134985] 1-per_cpu_trap_init(1787)ebase=0xffffffff80001000, c0_ebase=0xffffffff80001002
[   37.135039] CPU revision is: 000d9602 (Cavium Octeon III)
[   37.135041] FPU revision is: 00739600
[   37.150784] SMP: Booting CPU02 (CoreId  3)...
[   37.172277] 2-per_cpu_trap_init(1787)ebase=0xffffffff80001000, c0_ebase=0xffffffff80001003
[   37.172316] CPU revision is: 000d9602 (Cavium Octeon III)
[   37.172318] FPU revision is: 00739600
[   37.172430] Brought up 3 CPUs
```
Root.EBase Register reset 之后是 0xffffffff80000，trap_init读取的是 0xffffffff80001，何时改写的？答案是在u-boot启动linux引导阶段： do_bootoctlinux/start_linux/ start_os_common（见 2.1.1节 ）
## 2.2  start_kernel/init_IRQ/octeon_irq_init_ciu
主核core0调用start_kernel/init_IRQ/octeon_irq_init_ciu初始化中断控制器CIU：
（1）disable core0的各CIU外设中断，后面由具体驱动使能；
（2）注册外设中断的irq_domain；
（3）给Mailbox，wagchdog中断分配 irq_desc并初始化；
（4）给timer，pci等外设分配 irq_desc并初始化（ 这里比较特别，不用在驱动里面分配 irq_desc了，直接 request_irq即可 ）。
```
octeon_irq_init_ciu_percpu()// Disable All CIU Interrupts
octeon_irq_setup_secondary = octeon_irq_setup_secondary_ciu
octeon_irq_ip2 = octeon_irq_ip2_ciu//IP2中断handler,在plat_irq_dispatch调用
octeon_irq_ip3 = octeon_irq_ip3_ciu// IP3中断handler,在plat_irq_dispatch调用
octeon_irq_ip4 =  octeon_irq_ip4_ciu// IP4中断handler,在plat_irq_dispatch调用
chip = &octeon_irq_chip_ciu_v2//set irq_chip
octeon_irq_ciu_chip = chip
ciu_domain = irq_domain_add_tree(ciu_node, &octeon_irq_domain_ciu_ops, dd)//注册irq domain
octeon_irq_force_ciu_mapping(ciu_domain, i + OCTEON_IRQ_TIMER0, 0, i + 52)//创建4个timer中断的irq_desc，这里循环4次
   irq_alloc_desc_at(irq, 0)//分配 irq_desc
   irq_domain_associate(domain, irq, line << 6 | bit)
        domain->ops->map(domain, virq, hwirq)//调用 irq domain的map handler，建立软硬中断号的映射关系
            octeon_irq_ciu_map
                octeon_irq_set_ciu_mapping(virq, line, bit, 0, oc teon_irq_ciu_chip, handle_level_irq)
                    irq_set_chip_and_handler//设置 irq_desc的irq_chip和handle_irq
```
将给irq 121对应的irq_desc分配irq_chip  octeon_irq_ciu_chip
## 2.3 start_kernel/ early_irq_init
irq号码采用静态分配，共支持511个中断号， early_irq_init 初始化struct irq_desc[511]部分成员，如锁等。
## 2.4  start_kernel/ local_irq_enable  
start_kernel调用 local_irq_enable 使能本地cpu中断。
```
local_irq_enable  
    raw_local_irq_enable
        arch_local_irq_enable <arch\mips\include\asm\irqflags.h>
            ei //set Root.Status Register（ CP0 Register 12, Select 0 ）bit0（Interrupt enable）
```
## 2.5 request_irq
使用timer的驱动调用 request_irq设置irq_desc的action->handler指针和写CIU 中断控制器CVMX_CIU_INTX_EN1_W1S使能timer外部中断。 这里以驱动arch\mips\cavium-octeon\oct_ilm.c为例。
```
request_irq
    request_threaded_irq
        action->handler = handler
        __setup_irq
            irq_startup
                irq_enable(desc)
                    desc->irq_data.chip->irq_enable(&desc->irq_data)\\调用 octeon_irq_ciu_enable_v2

```
irq_enable将调irq_desc对应 irq_chip（ octeon_irq_ciu_chip ） 的 成员函数 irq_enable（ e octeon_irq_ciu_enable_v2 ） ， 写CIU 中断控制器CVMX_CIU_INTX_EN1_W1S使能timer外部中断 ，如timer3对应bit56，如打印所示： `octeon_irq_ciu_enable_v2(519)index=2, mask=0x80000000000000`
## 2.6 SMP从核异常初始化
SMP从核被唤醒后， 在start_secondary函数中完成从核异常初始化，具体包括：
（1）在mp_ops->init_secondary函数调用write_c0_ebase初始化从核Root.EBase Register，其值是主核core0在start_kernel/trap_init初始化的全局变量ebase=0xffffffff80001000，写入之后core2的Root.EBase Register等于0xffffffff80001002；
（2）在mp_ops->init_secondary函数调用octeon_irq_setup_secondary，初始化中断控制器；
（3）在mp_ops->smp_finish调用local_irq_enable使能本地cpu中断。
```
start_secondary
    per_cpu_trap_init(false)//clear BEV
    mp_ops->init_secondary() //调用octeon_init_secondary
        write_c0_ebase((u32)ebase) //写 Root.EBase Register
        octeon_irq_setup_secondary //调用octeon_irq_setup_secondary_ciu
            octeon_irq_init_ciu_percpu//Disable All CIU Interrupts
            octeon_irq_percpu_enable//调用irq_chip->irq_cpu_online（没有定义）
    mp_ops->smp_finish() //调用octeon_smp_finish
        local_irq_enable()
```
# 3. 中断异常处理流程
 当MIPS CPU发生 外部 中断后，CPU的状态变化总结如下：
（1）EPC寄存器保存了发生中断是程序执行指令的地址
（2）CP0中STATUS寄存器EXL置位为1，表示正在异常状态，这个时候CPU不能再响应中断（不管IE是否开启），自动进入核心状态
（3）CP0中CAUSE寄存器中ExcCode为0，表示是中断异常
（4）CP0中CAUSE寄存器IP被设置为不同的中断号
当发生timer中断，CPU会跳转到异常入口地址（即初始化写入的ebase寄存器指定的入口地址 0xffffffff80001000 ）开始执行异常处理程序except_vec3_generic，如下图所示：
![image](/images/posts/interrupt/octeon-ex-handling.png)
## 3.1 except_vec3_generic
读取Root.Cause寄存器，得到exception code，其中0号即外部中断，然后跳转到异常向量数组 exception_handler[0]
## 3.2  handle_int
 handle_int函数实际上就是注册到exception_handler[0]上专门处理中断引发的异常，该函数主要做的工作就是：
（1）调用SAVE_ALL保存中断现场
SAVE_ALL在arch\mips\include\asm\stackframe.h中定义：
```
.macro  SAVE_ALL docfi=0
SAVE_SOME \docfi
SAVE_AT \docfi
SAVE_TEMP \docfi
SAVE_STATIC \docfi
.endm
```
（2）调用CLI关闭中断
（3）调用plat_irq_dispatch判断中断类型进行分别处理
（4）处理中断完成后从ret_from_irq退出中断处理流程
## 3.3  plat_irq_dispatch
plat_irq_dispatch函数读取status（中断mask位）和cause寄存器相与，确定中断类型，如果是ip2则调用 octeon_irq_ip2_ciu 函数。
## 3.4 octeon_irq_ip2_ciu
octeon_irq_ip2_ciu读取ciu中断sum寄存器， 查表获取硬件中断号码，然后调用do_IRQ进入内核通用中断处理部分。

## 3.5 do_IRQ
do_IRQ调用generic_handle_irq最终执行request_irq注册的中断处理函数。
# 4 其他
## 4.1  set_c0_status函数的宏定义
set_c0_status用来设置status寄存器相应bit，其定义在文件 arch/mips/include/asm/mipsregs.h中
（1）首先定义 __BUILD_SET_C0宏用于定义` set_c0_##name`函数 ：
```
#define __BUILD_SET_C0(name)					\
static inline unsigned int					\
set_c0_##name(unsigned int set)					\
{								\
 unsigned int res, new;					\
        \
 res = read_c0_##name();					\
 new = res | set;					\
 write_c0_##name(new);					\
        \
 return res;						\
} 
```
（2）然后使用 __BUILD_SET_C0(status) 定义 set_c0_status 函数：
```
__BUILD_SET_C0(status)
__BUILD_SET_C0(config)
...
```
## 4.2  OCTEON CN71xx interrupt features
不支持外部中断控制器和矢量粥中断？实验验证， CN71xx  ebase寄存器bit[63:30]并不可写即使set WG bit，所以可编程的异常入口地址范围是[0x0, 0x3FFFF]。

# 5 参考资料
* Cavium OCTEON III CN71XX Hardware Reference Manual
* [Mips 中断处理分析](http://blog.chinaunix.net/uid-28236237-id-3913639.html)

