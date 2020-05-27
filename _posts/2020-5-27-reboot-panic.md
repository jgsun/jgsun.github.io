---
layout: post
title:  "linux reboot/panic子系统"
categories: system
tags: reboot，panic，busybox，s6，sysrq
author: jgsun
---


* content
{:toc}
# 1. Overview
作为一名嵌入式linux软件工程师，reboot命令是经常使用的命令，比如恢复板卡故障，板卡软件升级之后等。当敲了reboot命令之后，用户空间发生了什么操作？通过reboot系统调用到内核之后，内核reboot子系统又发生什么了操作，最终引导系统重启？板卡panic也会经常遇到，这绝对是嵌入式程序员的噩梦，但是内核panic子系统是如何工作的呢？了解内核reboot和panic子系统，有助于我们解决系统问题。

















# 2. reboot流程图

![image](/images/posts/system/busybox-reboot.png)


对于嵌入式系统，reboot命令都是busybox集成，当敲入reboot命令之后，进入halt_main，向1号init进程发送SIGTERM信号；如果init-system采用busybox-init，则init进程的init_main接收到信号后给其他进程发送终止信号，最后调用C库函数reboot，reboot通过系统调用sys_reboot进入内核。
内核reboot子系统将根据系统调用参数将系统halt，poweroff或者restart；如果是restart将执行device驱动的shutdown函数，执行别的内核模块注册的reboot_notifier函数；然后执行syscore_shutdown，最后调用体系结构相关的函数machine_restart，这里以Marvell MIPS octeon为例，执行octeon_restart写cpu寄存器让系统重启。
如果init system是s6 init system，reboot命令用户空间流程如下，内核reboot流程和上图是一样的。
![image](/images/posts/system/s6-reboot.png)


# 3. panic流程图
在配置Linux Magic System Request（/drivers/tty/sysrq.c）的系统中，执行`echo c > /proc/sysrq-trigger`可以触发内核panic，可借此研究内核panic子系统。下面是marvell octeon平台的panic log：
```
~ # echo c > /proc/sysrq-trigger 
[ 8361.530756] sysrq: SysRq : Trigger a crash
[ 8361.534968] (c1 sh <518>) CPU 1 Unable to handle kernel paging request at virtual address 0000000000000000, epc == ffffffffc03f5364, ra == ffffffffc03f5354
[ 8361.549852] (c1 sh <518>) Oops[#1]:
[ 8361.554302] (c1 sh <518>) CPU: 1 PID: 518 Comm: sh Tainted: G O 4.9.79-Cavium-Octeon #113
[ 8361.564569] (c1 sh <518>) task: 800000005f0e1d00 task.stack: 800000005f168000
[ 8361.572662] $ 0 : 0000000000000000 0000000014009ce1 0000000000000001 ffffffffc19f0000
[ 8361.580731] $ 4 : ffffffffc08c8480 0000000000000001 800000005f0e2000 0000000000000000
[ 8361.588796] $ 8 : 00000000000001aa ffffffffc19fe308 0000000000004517 0000000000000001
[ 8361.596859] $12 : 0000000000000000 0000000000000007 0000000000000000 0000000000000006
[ 8361.604924] $16 : 0000000000000063 ffffffffc08c0000 ffffffffc08eea80 0000000000000007
[ 8361.612990] $20 : 0000000000000000 ffffffffc08ee760 ffffffffc08f0000 0000000000000020
[ 8361.621054] $24 : 0000000000000007 ffffffffc0400ae0                                  
[ 8361.629122] $28 : 800000005f168000 800000005f16bca0 000000001072156c ffffffffc03f5354
[ 8361.637188] (c1 sh <518>) Hi : 0000000000819446
[ 8361.642936] (c1 sh <518>) Lo : c8b4395810fc2de7
[ 8361.648694] (c1 sh <518>) epc : ffffffffc03f5364 sysrq_handle_crash+0x2c/0x40
[ 8361.656965] (c1 sh <518>) ra : ffffffffc03f5354 sysrq_handle_crash+0x1c/0x40
[ 8361.665231] Status: 14009ce3 KX SX UX KERNEL EXL IE 
[ 8361.670280] (c1 sh <518>) Cause : 0080000c (ExcCode 03)
[ 8361.676463] (c1 sh <518>) BadVA : 0000000000000000
[ 8361.682211] (c1 sh <518>) PrId : 000d9602 (Cavium Octeon III)
[ 8361.689000] Modules linked in: cpld_eps(O) uio_generic_driver(O) cpld_irq(O) cpld_spi(O) generic_access(O) generic_regmap(O) reborn_class(O)
[ 8361.701711] (c1 sh <518>) Process sh (pid: 518, threadinfo=800000005f168000, task=800000005f0e1d00, tls=0000000077eba620)
[ 8361.713625] Stack : 0000000000000000 ffffffffc03f559c 80000000027cdc68 800000006590b000
[ 8361.721689] fffffffffffffffb 00000000107242e0 0000000000000002 0000000000000004
[ 8361.729755] 0000000000000063 0000000000000030 0000000000000020 ffffffffc03f575c
[ 8361.737819] 0000000000000002 000000007fca2f90 0000000000000000 ffffffffc0230f94
[ 8361.745883] 0000000000000054 800000005f7b4c00 800000005f16be00 ffffffffc01c27e4
[ 8361.753948] 800000005f16be50 80000000027cdc00 0000000000000000 0000000000000054
[ 8361.762012] 0000000010069c28 ffffffffc0085068 0000000000030002 0000000000000000
[ 8361.770075] 0000000000000000 ffffffffc011fcc0 0000000100002301 800000005fb06000
[ 8361.778141] 8000000002763100 0000000000000002 800000005f7b4c00 ffffffffc01c2c04
[ 8361.786205] 800000005f16be00 ffffffffc01c018c 800000005f7b4c00 000000005f7b4c00
[ 8361.794271] ...
[ 8361.796743] (c1 sh <518>) Call Trace:
[ 8361.801364] (c1 sh <518>) [<ffffffffc03f5364>] sysrq_handle_crash+0x2c/0x40
[ 8361.809289] (c1 sh <518>) [<ffffffffc03f559c>] __handle_sysrq+0xc4/0x200
[ 8361.816952] (c1 sh <518>) [<ffffffffc03f575c>] write_sysrq_trigger+0x4c/0x68
[ 8361.824964] (c1 sh <518>) [<ffffffffc0230f94>] proc_reg_write+0x74/0xa8
[ 8361.832540] (c1 sh <518>) [<ffffffffc01c27e4>] __vfs_write+0x3c/0x128
[ 8361.839942] (c1 sh <518>) [<ffffffffc01c2c04>] vfs_write+0xb4/0x218
[ 8361.847170] (c1 sh <518>) [<ffffffffc01c4d28>] SyS_write+0x60/0xe0
[ 8361.854314] (c1 sh <518>) [<ffffffffc008111c>] syscall_common+0x18/0x44
[ 8361.861889] Code: 3c03c19f ac627470 0000000f <a0020000> dfbf0008 03e00008 67bd0010 00000000 03e00825 
[ 8361.871754] (c1 sh <518>) 
[ 8361.875444] (c1 sh <518>) ---[ end trace 67a7cc1a1dd2d4e5 ]---
[ 8361.882242] (c1 sh <518>) We will reset the spi-nor flash
[ 8361.888615] (c1 sh <518>) reset done!
[ 8361.893241] (c1 sh <518>) Kernel panic - not syncing: Fatal exception
[ 8361.900649] (c1 sh <518>) ---[ end Kernel panic - not syncing: Fatal exception

*** NMI Watchdog interrupt on Core 0x03 ***
```
panic流程图如下：
![image](/images/posts/system/panic.png)

sysrq的sysrq_handle_crash调用panic函数就触发了内核panic。
内核panic子系统将dump_stack，调用注册的notifier链表的回调函数，按配置打印系统信息（默认不打印，/proc/sys/kernel/panic_print）最后根据panic_timeout的值选择以那种方式触发系统重启。如果panic_timeout小于0，直接调用emergency_restart重启cpu；如果panic_timeout大于等于0，都将写watchdog导致watchdog超时重启，不同的是panic_timeout大于0，将delay一段时间重启。

## 3.1 panic 配置选项
新版内核提供/proc/sys/kernel/panic_print配置参数，可以让内核在panic的同时打印出系统信息：
```
#define PANIC_PRINT_TASK_INFO	0x00000001 //task信息
#define PANIC_PRINT_MEM_INFO	0x00000002 //内存信息
#define PANIC_PRINT_TIMER_INFO	0x00000004 //系统timer
#define PANIC_PRINT_LOCK_INFO	0x00000008 //lock信息
#define PANIC_PRINT_FTRACE_INFO	0x00000010 //打印ftrace
#define PANIC_PRINT_ALL_PRINTK_MSG	0x00000020//打印内核log_buf
```
比如设置panic_print为0x3F将打印出上述全部系统信息：
```
root@/proc/sys/kernel# echo 0x3F > panic_print
root@/proc/sys/kernel# cat panic_on_oops
1
```
# 4. reboot和panic的notifier机制
从上面reboot和panic的流程图可以看出，都将执行内核其他模块注册的notifier函数，可以利用这一机制，通过向注册reboot和panic子系统注册notifier函数，实现在rboot和panic过程中执行特定的任务：比如reboot的时候dump外设寄存器，重启外接设备，保护现场，置内存为自刷新模式，设置是warm还是poweron重启等；panic的时候记录系统panic次数，dump系统信息等。

# 5. 结束语
reboot看似简单，但是其实现却很复杂。内核遍历执行各个设备的shutdown方法；遍历执行notifier，systern core shutdown回调函数；最后调用arch相关的mechine_restart复位cpu。我们更多的是关注板卡的启动过程，很少关注reboot，只有出了reboot相关的问题后，才会去了解。理解整个reboot的流程，可以有助于我们定位问题；也可以帮助我们面对新feature，提出新的解决方案。以前遇到一个板卡reboot卡死的问题，经过调查发现是qspi flash没有复位造成的，最后通过在flash设备驱动中的shutdown函数中和panic函数中复位flash解决了这个问题。
