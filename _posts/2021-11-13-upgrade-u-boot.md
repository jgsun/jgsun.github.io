---
layout: post
title: "Upgrade u-boot from 2009 to 2017.01 on ARM"
category: boot
tags: u-boot ARM
author: jgsun
---

* content
{:toc}

这是一篇 2017 年的旧文。

因为想用 一块 ARM 板卡来做 kernel ramdump/crash-utility 的 demo，要在u-boot阶段来 dump 内存，而该板卡的 u-boot 版本是 2009，没有支持 ubifs 和 tftpput 等命令，所以产生了升级其 u-boot 到版本 2017.01 的动机。这篇 log 记录了升级过程。
![image](images/posts/boot/m853xx_block_diagram.png)
















最后使用 2017.01 版本的 u-boot 顺利完成了 kernel ramdump/crash-utility 的 demo，而且升级后的 u-boot 也被同事拿去正式使用！

# 一、创建u-boot-2017.01 baseline
1. 从 u-boot 开源网站下载 u-boot-2017.01
<ftp://ftp.denx.de/pub/u-boot/u-boot-2017.01.tar.bz2>

2. 解压到目录/repo/jiangusu/lib/u-boot-2017.01，commit所有代码建立u-boot-2017.01的baseline

    u-boot-2017.01$ hg init
    u-boot-2017.01$ hg add*
    u-boot-2017.01$ hg ci

3. 从 /repo/jiangusu/lib/u-boot-mindspeed-sdk 库拷贝如下源代码到 /repo/jiangusu/lib/u-boot-2017.01 库相应目录，并 commit，这样之后修改的代码能产生 diff 进行跟踪。

|u-boot-mindspeed-sdk|u-boot-2017.01|
|-----------------------------|----------------------|
|cpu/m853xx/|arch/arm/mach-m853xx|
|include/asm-arm/arch-m853xx/|arch/arm/include/asm/arch-m853xx/|
|board/isam_common/|board/isam_common/|
|board/mindspeed/|board/mindspeed/|
|include/configs/niata_nor16.h|include/configs/niata_nor16.h|
|include/mindspeed_version.h|include/mindspeed_version.h|
|drivers/spi/comcerto_spi.c|drivers/spi/comcerto_spi.c|
|drivers/net/comcerto_gem.c|drivers/net/comcerto_gem.c|
|drivers/net/comcerto_gem.h|drivers/net/comcerto_gem.h|
|drivers/net/comcerto_gem_AL.c|drivers/net/comcerto_gem_AL.c|
|drivers/net/comcerto5300_gem.h|drivers/net/comcerto5300_gem.h|

# 二、u-bootb配置
u-boot 从版本 2014.10-rc1 开始使用 kconfig的配置方式，详见 u-boot 源码文档：doc/README.kconfig。u-boot-mindspeed-sdk 的版本是2009，使用的是传统 C 语言预处理的板级配置方式，所以需要升级到 kcnfig 的配置方式。

## 1. kconfig配置
(1) 在arch/arm/Kconfig 添加 COMCERTO 配置选项
```
diff --git a/arch/arm/Kconfig b/arch/arm/Kconfig
--- a/arch/arm/Kconfig
+++ b/arch/arm/Kconfig
@@ -648,6 +648,11 @@ config ARCH_ZYNQ
        select DM_USB if USB
        select BLK
 
+config COMCERTO
+       bool "Mindspeed Comcerto platform"
+       select CPU_V7
+       select OF_CONTROL
+
 config ARCH_ZYNQMP
        bool "Support Xilinx ZynqMP Platform"
        select ARM64
@@ -975,6 +980,8 @@ source "arch/arm/mach-zynq/Kconfig"
 
 source "arch/arm/cpu/armv7/Kconfig"
 
+source "arch/arm/mach-m853xx/Kconfig"
+
 source "arch/arm/cpu/armv8/zynqmp/Kconfig"
 
 source "arch/arm/cpu/armv8/Kconfig"
```

(2) 创建 arch/arm/mach-m853xx/Kconfig，配置 vendor name, soc, board name 和 board cinfiguration name

|items|value|config symbol|comments|
|------|------|------|------|
|cpu|cpu_v7|CONFIG_SYS_CPU_V7|compile arch/arm/cpu/armv7|
|soc|m853xx|CONFIG_SYS_SOC|include/asm/arch/ -> include/asm/arch-m853xx/|
|vendor|mindspeed|CONFIG_SYS_VENDOR|specify CONFIG_BOARDDIR = board/mindspeed/niata|
|board name|niata|CONFIG_SYS_BOARD|compile board/mindspeed/niata/|
|board configuration name|niata_nor|CONFIG_SYS_CONFIG_NAME|include/configs/niata_nor16.h|

```
diff --git a/arch/arm/mach-m853xx/Kconfig b/arch/arm/mach-m853xx/Kconfig
new file mode 100644
--- /dev/null
+++ b/arch/arm/mach-m853xx/Kconfig
@@ -0,0 +1,22 @@
+if COMCERTO
+
+config SYS_VENDOR
+       string "Vendor name"
+       default "mindspeed"
+
+config SYS_SOC
+       default "m853xx"
+
+config SYS_BOARD
+       string "Board name"
+       default "niata"
+
+config SYS_CONFIG_NAME
+       string "Board configuration name"
+       default "niata_nor16"
+       help
+         This option contains information about board configuration name.
+         Based on this option include/configs/<CONFIG_SYS_CONFIG_NAME>.h header
+         will be used for board configuration.
+
+endif
```
需要注意的是 SYS_CONFIG_NAME 设置要和 include/comfigs/niata_nor16.h 名字一致，因为编译的时候，这个将根据此配置去引用 include/comfigs/niata_nor16.h 这个头文件；此外 SYS_SOC 也非常重要，要和arch/arm/include/asm/arch-m853xx/ 路径的名称 arch-m853xx 一致，设置成 m853xx，而不是 arch-m853xx，Makefile 将根据 SYS_SOC 找到 arch/arm/include/asm/arch-m853xx/，否则如果源文件包含`#include <asm/arch/hardware.h>` 将找不到头文件。

(3) 创建 configs/niata_nor16_defconfig
运行 make menuconfig，在 ARM architecture 的 Target select 选中刚刚在 arch/arm/Kconfig 创建的平台“Mindspeed Comcerto platform”； 然后选择其他配置，如 Command line interface/Boot commands/bootm、go、run等；然后 save，u-boot 根目录将得到配置文件 .config，然后将其拷贝到 configs 目录并命名为 niata_nor16_defconfig. (这里须将配置文件命名为 niata_nor16_defconfig，因为buildroot 设置的 U-Boot board name=niata_nor16，这样在通过 buildroot 启动 u-boot 编译的时候将执行 make niata_nor16_config 进行配置)


## 2. 修改 include/configs/niata_nor16.h
niata_nor16.h 是板级配置文件，需要去掉和 configs/niata_nor16_defconfig 重复的部分，否则编译的时候报 warning。u-boot 鼓励采用kconfig 配置方式，目标是消灭 Board configuration 方式；编译最后顶层 Makefile 会调用 scripts/check-config.sh 来检查，如果发现新的 CONFIG 没有使用 Kconfig 就会报错（暂时将它去掉了，不会影响编译结果）。
```
@# Check that this build does not use CONFIG options that we do not
@# know about unless they are in Kconfig. All the existing CONFIG
@# options are whitelisted, so new ones should not be added.
# Tentatively disable u-boot.cfg check
$(srctree)/scripts/check-config.sh u-boot.cfg \
        $(srctree)/scripts/config_whitelist.txt ${srctree} 1>&2    
```
```
./scripts/check-config.sh u-boot.cfg \
                ./scripts/config_whitelist.txt . 1>&2
Error: You must add new CONFIG options using Kconfig
The following new ad-hoc CONFIG options were detected:
CONFIG_C5300
CONFIG_CMD_NET_ETHDIAG
CONFIG_CMD_RAMTEST
CONFIG_COMCERTO_GEMAC
CONFIG_COMCERTO_NIATA
CONFIG_COMCERTO_SPI
CONFIG_DDR3_1333MHZ
CONFIG_ENC28J60
CONFIG_ENC28J60_BUS
CONFIG_ENC28J60_CS
CONFIG_ENC28J60_SPEED
CONFIG_FDT_RELOCATE_ADDR
CONFIG_GEMAC1_PHY_PRESENT
CONFIG_GTH
CONFIG_ISAM
CONFIG_ISAM_CPLD
CONFIG_ISAM_CPLD_BASE
CONFIG_ISAM_DUALBOOT
CONFIG_ISAM_FLASH
CONFIG_ISAM_LED
CONFIG_ISAM_WATCHDOG
CONFIG_KERNEL_ENTRY_POINT
CONFIG_PROZONE_BASE_ADDR
CONFIG_PROZONE_SIZE
CONFIG_RAMDUMP
CONFIG_RAMDUMP_RAM
CONFIG_RAMDUMP_RAM_BASE
CONFIG_SDRAM_BASE_ADDR
CONFIG_SDRAM_SIZE

Please add these via Kconfig instead. Find a suitable Kconfig
file and add a 'config' or 'menuconfig' option.
make[1]: *** [all] Error 1
```

编译的时候将自动生成头文件 include/config.h， 可见 include <configs/niata_nor16.h>，niata_nor16.h 也在其中。
```
/* Automatically generated - do not edit */  
#define CONFIG_BOARDDIR board/mindspeed/niata
#include <config_defaults.h>
#include <config_uncmd_spl.h>
#include <configs/niata_nor16.h>
#include <asm/config.h>
#include <config_fallbacks.h>
```

# 三、代码修改
## 1. arch/arm/cpu/armv7/start.S
增加 niata 板级启动代码：

    (1) write CA9_MC CPU reset register，let arm1 in reset
    (2) No IRAM remapped to low 256KB
    (3) Enable access to debugger
    (4) set DDR Low and High limit

## 2. board/mindspeed/niata/lowlevel_init.S
修改了判断 boot 启动位置的变量，其他几乎没有改动，未来可能会增加 dual boot 启动部分代码。

## 3. arch/arm/dts/niata.dts
移动 board/mindspeed/niata/niata.dts 到目录 arch/arm/dts/niata.dts，并在 Makefile 添加：

    dtb-$(CONFIG_COMCERTO) += niata.dtb

## 4. arch/arm/cpu/u-boot.lds
拷贝自相关 changeset 的 commit message：

    (1) The link address of bss and rel_dyn is ovelay, so the contents
        of rel_dyn can be modified by variables located in bss before relocation
        and causes data abort or undefined instruction error.Now remove this
        overlay settings.
    (2) Removed board/mindspeed/niata/u-boot.lds.
    (3) Add linker script variables:__arm1_init_start and __arm1_init_end,
        used to find the start and end address of arm1_init code.

## 5. arch/arm/mach-m853xx/cpu.c
拷贝自相关changeset的commit message：

    Remove the cpu specific interfaces: cache operations, cleanup_before_linux
    and do_reset because there are common interfaces in arch/arm/cpu/armv7 and
    arch/arm/lib already.

## 6. board/isam_common/bps.c, board/isam_common/Makefile
整理自相关 changeset 的 commit message：

    (1) update the image interfaces because the common/image.c 
        Add parameter FIT_KERNEL_PROP for fit_conf_get_kernel_node 
        Use fit_image_verify to replace fit_image_check_hashes
    (2) update the Makefile.

## 7. board/mindspeed/niata/
整理自相关 changesets 的 commit message：

    (1) Update the Makefile
    (2) Mask the cpld interrupt in board_init instead of in interrupt_init,
        because the interrupt_init is a common interface.
    (3) Init the gd->ram_size in dram_init, or else the u-boot cant get the
        ram size.
    (4) Reset the timer3 after timer initialization, or else the get_ticks 
        work abnormally.
    (5) Remove the udelay and interrupt_init because they are impelmented
        commonly
    (6) Update the interface to get nand_chip index.

## 8. common/board_init_f.c,common/board_init_r.c,common/main.c
整理自相关 changesets 的 commit message：

    (1) Add board_init, isam_early_init, initf_flash and failsafe_scan
        in board_init_r
    (2) Add isam_bootdelay in main_loop.

## 9. drivers/serial/ns16550.c

    (1) Add macro CONFIG_SYS_NS16550_MEM32 to set the register length of the
        ns16550 to be 32bits.
    (2) Change to little endian.

## 10. drivers/spi/comcerto_spi.c

    Add a dummy spi_init interface to avoid compiling error:
    ...
    LD      u-boot
    common/built-in.o: In function `jumptable_init':
    /repo/jiangusu/lib/u-boot-2017.01/common/exports.c:30: undefined reference to `spi_init'

## 11. ethernet
arch/arm/mach-m853xx/cpu.c
drivers/net/comcerto_gem.c
拷贝自changeset 0a84145ac1066c84e5153679c6b60f7ba0f4d589

    (1)Add board_eth_init for LEMI init.
    (2)Add cpu_eth_init for sgmii init.
    (3)Update the net interfaces.

## 12. add mindspeed comcerto platform
arch/arm/Makefile
arch/arm/Kconfig
mach-m853xx/Kconfig

    (1) Add config item COMCERTO in arch/arm/Kconfig
    (2) Create arch/arm/mach-m853xx/Kconfig to config vendor, SOC, board name
        and board configuration name
            vendor                                 -> mindspeed
            SOC                                     -> m853xx
            board name                        -> niata
            board configuration name -> naita_nor16
    (3) Specify the machine directory name m853xx in arch/arm/Makefile

# 四、遇到的问题及解决
移植 uboot 先做一个最精简版本，很多配置选项都没有打开，只配置基本的 ddr，serial 和 clk 最小系统，这样先保证 uboot 能正常启动进入命令行，然后再去添加其他的配置。这里因为已经有老版本的 u-boot 可以运行，先让 u-boot 从内存启动，不用烧写到 flash，省去调试的步骤，而且跳过 DDR/CLK 的初始化，只用先面对串口一个问题。另外调试 u-boot 启动需要借助仿真器，经常需要用到反汇编，一些地方需要对照汇编指令，利用仿真器单步跟踪，以找到出问题的地方。
反汇编命令：

    jiangusu@ASBLX60:/repo/jiangusu/ms_isam/sw/vobs/esam/build/reborn/buildroot-isam-reborn-comcerto-niata$ arm-none-linux-gnueabi-objdump -D output/build/uboot-custom/u-boot | less

## 1. 加载地址，链接地址
执行到 init_sequence_f[] 的第一个函数 setup_mon_len 的时候产生undefine error；
原因：链接地址是 0，加载地址是 0x10000000，导致 initcall_run_list(init_sequence_f) 参数地址传递错误，传递的是链接地址；将 arch/arm/config.mk 文件编译参数 -fno-pic 改为 -fpic，即位置无关之后，可正确找到 init_sequence_f 的加载地址。

```
arch/arm/config.mk
ifneq ($(CONFIG_SPL_BUILD),y)
# Check that only R_ARM_RELATIVE relocations are generated.
ALL-y += checkarmreloc
# The movt / movw can hardcode 16 bit parts of the addresses in the
# instruction. Relocation is not supported for that case, so disable
# such usage by requiring word relocations.
PLATFORM_CPPFLAGS += $(call cc-option, -mword-relocations)
PLATFORM_CPPFLAGS += $(call cc-option, -fno-pic)                     
endif
```
但是后面还会有别的问题（记得是 relocate 的时候挂住，在 copy_loop 陷入死循环），所以将链接地址和加载地址设为一样解决了这个问题，即在 inclue/configs/niata_nor16.h 中添加配置项：
```
inclue/configs/niata_nor16.h
/* Physical Memory map */
#define CONFIG_SYS_TEXT_BASE            0x10000000
#define CONFIG_SYS_UBOOT_START  CONFIG_SYS_TEXT_BASE
```
后来查资料，发现 arm u-boot 使用绝对地址，并不是完全的(pic)代码位置无关，是通过调整绝对地址符号的 label 表来实现代码的搬移来实现pic，如果不做 relocate 或者在 relocate 之前还是需要加载到连接地址的位置上。arm 平台使用 mword-relocations 来生成位置无关代码，使用链接 -pie 选项用于生成 PIE 位置无关可执行文件。对于一些绝对地址符号（例如已经初始化的全局变量），会将其以 label 的形式放在每个函数的代码实现的末端。同时，在链接的过程中，会把这些 label 的地址统一维护在 .rel.dyn 段中，当 relocation 的时候，方便对这些地址的 fix。“位置无关代码”主要是通过使用一些只会使用相对地址的指令实现，比如“b”、“bl”、“ldr”、“adr”等等。
关于 relocate 的原理，网上一篇文章[arm uboot relocation介绍](http://blog.csdn.net/ooonebook/article/details/53047992)写的很好，上面一段内容就是整理自这篇文章。

我们的 u-boot 没有启用 MMU，应该可以借助 MMU 来实现地址重映射，这方面没有继续研究。
后面在调试上电从 flash 启动的时候，将 u-boot 的链接地址设为了 flash 的地址 0xa0000000。刚上电，flash被 map 到 0 地址，u-boot 的运行地址是 0，而链接地址 0xa0000000；按正常逻辑应将链接地址设为 0，DDR 初始化之后将 u-boot 代码 relocate 到高端地址实现位置无关运行，然后将 flash remap 到 0xa0000000；但是这个 OBC 的 DDR 初始化必须在 flash remap 到 0xa0000000之后才能成功（后面[从 ram 启动正常，但是烧写到 flash 不能启动](#flash)会讲到这个问题）。可以看出，上电之初一段时间 u-boot 的运行地址和加载地址是不一样的，所以这段代码不能寻址全局变量。

## 2. UART 驱动(ns16550)：寄存器 offset 问题
不能访问 ns16550 寄存器，反汇编发现寄存器 offset 不正确，lsr 的 offset 是23这个奇怪的数字。
在 include/configs/niata_nor16.h 中增加配置项 CONFIG_SYS_NS16550_MEM32，可见 lsr 寄存器的 offset 变为正确的20（0x14）
```
 #define CONFIG_SYS_NS16550_MEM32
反汇编：
10025908 <NS16550_putc>:
10025908:       e5903014        ldr     r3, [r0, #20]
1002590c:       e6bf3f33        rev     r3, r3
10025910:       e3130020        tst     r3, #32
10025914:       0afffffb        beq     10025908 <NS16550_putc>
10025918:       e351000a        cmp     r1, #10
1002591c:       e1a03c01        lsl     r3, r1, #24
10025920:       e5803000        str     r3, [r0]
10025924:       112fff1e        bxne    lr
10025928:       eaff7a2a        b       100041d8 <hw_watchdog_reset>
```

## 3. UART驱动(ns16550)：寄存器读写大小端问题
NS16550_init 初始化读出的 lsr 的 value 是 0x60，不是期望的 0x40。
仿真器读取 UART1 寄存器，lsr 显示是 00000060：
```
msp@0>md 0xfe090000
fe090000 : 0000001c 00000000 00000007 00000000  ................
fe090010 : 00000000 00000060 000000fb 00000000  ....`...........
fe090020 : 00000000 00000000 00000000 00000000  ................
fe090030 : 00000000 00000000 00000000 00000000  ................
fe090040 : 00000000 00000000 00000000 00000000  ................
fe090050 : 00000000 00000000 00000000 00000000  ................   
```
但是跟踪发现代码 load 到寄存器的确实 61000000，正好大小端颠倒：
```
msp@0>rdall
         User     FIQ     Superv   Abort     IRQ      Undef
GPR00: fe090000 fe090000 fe090000 fe090000 fe090000 fe090000
GPR01: 00000020 00000020 00000020 00000020 00000020 00000020
GPR02: 00000120 00000120 00000120 00000120 00000120 00000120
GPR03: 61000000 61000000 61000000 61000000 61000000 61000000
GPR04: 1ffffad8 1ffffad8 1ffffad8 1ffffad8 1ffffad8 1ffffad8
```
修改 ns16550 的驱动，改为小端模式之后，串口可以工作有了打印输出。
```
diff --git a/drivers/serial/ns16550.c b/drivers/serial/ns16550.c
--- a/drivers/serial/ns16550.c
+++ b/drivers/serial/ns16550.c
@@ -29,8 +29,8 @@ DECLARE_GLOBAL_DATA_PTR;
 #define serial_out(x, y)       outb(x, (ulong)y)
 #define serial_in(y)           inb((ulong)y)
 #elif defined(CONFIG_SYS_NS16550_MEM32) && (CONFIG_SYS_NS16550_REG_SIZE > 0)
-#define serial_out(x, y)       out_be32(y, x)
-#define serial_in(y)           in_be32(y)
+#define serial_out(x, y)       out_le32(y, x)
+#define serial_in(y)           in_le32(y)
 #elif defined(CONFIG_SYS_NS16550_MEM32) && (CONFIG_SYS_NS16550_REG_SIZE < 0)
 #define serial_out(x, y)       out_le32(y, x)
 #define serial_in(y)           in_le32(y)
```

## 4. bss 段和 rel_dyn 段 overlelay
问题：代码 relocate 之后，出现 data abort 问题
```
DRAM:  2 GiB
initcall: 1000e9d0
New Stack Pointer is: 7ff38520
initcall: 1000e9b0
initcall: 1000ec44
initcall: 1000ebd8
Relocation Offset is: 6ff5c000
Relocating to 7ff5c000, new gd at 7ff39f08, sp at 7ff38520
data abort
pc : [<10000780>]          lr : [<7ff5c6e8>]
sp : 7ff38520  ip : 00000400     fp : 1003c6a4
r10: 1004fbd7  r9 : 7ff39f08     r8 : 00c5ffd4
r7 : 02008000  r6 : 00000d73     r5 : 100003b8  r4 : 6ff5c000
r3 : 1005f3c0  r2 : 100568d0     r1 : 00000017  r0 : 7ff5c036
Flags: nZCv  IRQs off  FIQs off  Mode SVC_32
Resetting CPU ...
resetting ...
wait board reset ...cpld write addr 0xa400000a value 0x4
...
```
分析：从 reset dump 出的寄存器看出 pc 是10000780（由于 ARM 的流水线机制，当前 PC 值为当前地址加 8 个字节），当前地址对应的汇编指令是：10000778:       e5901000        ldr     r1, [r0]，而 r0 存储的地址是 7ff5c036，不是 word 对齐，所以 load 的时候 data abort.

原因：u-boot 的链接脚本 arch/arm/cpu/u-boot.lds 默认设置 bss 段和  rel_dyn 段 overlay（为了省空间）。rel_dyn 段存的是全局 label的地址，用于 relocate 的时候进行修正，所有就得保证在 relocate 之前这段地址的内容不能被修改；而这里 bss 段和这段地址重合，正好有分配在 bss 段的变量在 relocate 之前进行了修改，所有导致 label 地址修改错误，所以这里需要取消 bss 段和 rel_dyn 段 overlelay 设置。

在 relocation 之前使用的 bss 段变量不多，后面在把大部分初始化工作放在 relocation 之后执行之后，就几乎没有了这样的变量了，这也是arm u-boot 默认设置 overlay 的原因，策略是尽快完成 relocate，之后全部在 DDR 运行。

如果有个别变量放在 bss 段而且在 relocation 之前需要访问，但是不想更改链接脚本 overlay 设置，可以使用使用 GCC 的扩展机制，指定该变量处于 .data 段。还有一个好处是，relocation 是不搬迁 bss 段的，放在 data 段就可以让 relocation 搬迁到新的位置，从而在 relocation 之后可以继续访问。
```
__attribute__((section(".data"))) flash_info_t flash_info[CFI_MAX_FLASH_BANKS];       /* FLASH chips info */
```

## 5.nand_init 问题
分析汇编指令和寄存器内容，nand_init 的时候在执行 NAND_CMD_RESET（0xff）的时候产生 data abort：
nand_init/nand_scan_ident/chip->cmdfunc(mtd, NAND_CMD_RESET, -1, -1)

根因：u-boot2007 根据 MTD 获取 nand_chip 指针采用了新的接口函数 mtd_to_nand，u-boot2009 采取的是在 nand_init_chip 函数中将 nand 指针赋给 mtd 的 priv 变量：mtd->priv = nand;
```
static inline struct nand_chip *mtd_to_nand(struct mtd_info *mtd)
{
	return container_of(mtd, struct nand_chip, mtd);
}
```

## 6. linux 启动解压后板子 reset
打印出 Uncompressing Linux... done, booting the kernel.之后不久板子 reset.
板子 reset原因是 kernel 在这里卡住致看门狗溢出，设置环境变量 diswdt=1，在 kernel 启动阶段禁止看门狗后，虽然板子不 reset，但是 kernel 卡住了，下面一条将分析原因和解决方法。所以在调试 kernel 启动的时候，需要禁止看门狗。

## 7. eanble CONFIG_OF_LIBFDT 之后，kennel 启动解压后 hang 的问题
NIAT-A 所用 kernel 版本较低，OBC 厂家 SDK kernel 的设备管理采用是传统的板级配置文件方式，没有使用 DTS 方式，但是我们的 isam driver（如e1，cpld）使用 DTS 方式，所以 NIATA 采用的方法是：kernel 启动参数使用 cmdline 指定，bootm 命令参数指定 dtb 地址，在bootm 命令中通过 boot_get_cmdline 从 bootargs 环境变量获取启动参数赋给 bootm_headers 的 cmdline_start 变量。
```
/*
 * Legacy and FIT format headers used by do_bootm() and do_bootm_<os>()
 * routines.
 */
typedef struct bootm_headers {	
...
#ifndef USE_HOSTCC
	image_info_t	os;		/* os image info */
	ulong		ep;		/* entry point of OS */

	ulong		rd_start, rd_end;/* ramdisk start/end */

	char		*ft_addr;	/* flat dev tree address */
	ulong		ft_len;		/* length of flat device tree */

	ulong		initrd_start;
	ulong		initrd_end;
	ulong		cmdline_start;
	ulong		cmdline_end;
	bd_t		*kbd;
#endif

```
上面说了，因为 kernel 的 isam driver 要使用 dtb，所以 u-boot 需要配置 CONFIG_OF_LIBFDT=y，以启动的时候将 dtb 地址传递给 kernel，但是使能该配置之后，启动的时候 kernel 在解压之后 hang 住了：
```
 ## Flattened Device Tree blob at 02000100
   Booting using the fdt blob at 0x2000100
   Loading Kernel Image ... OK
   Loading Device Tree to 77ffc000, end 77fff9b3 ... OK

Starting kernel ...

cpu1 ...

Uncompressing Linux... done, booting the kernel.
```
halt 住 arm1，发现 arm1 产生了 Abort 错误；从 Loading Device Tree to 77ffc000, end 77fff9b3 ... OK 看出，dtb 被 load 到了地址 0x77ffc000，位于 high_mem, kenrel 启动的时候不能访问 high_mem, 从读出的寄存器验证了这一点：
```
csp@1>rda
         User     FIQ     Superv   Abort     IRQ      Undef
GPR00: 35ffc000 35ffc000 35ffc000 35ffc000 35ffc000 35ffc000
GPR01: c0a73ad8 c0a73ad8 c0a73ad8 c0a73ad8 c0a73ad8 c0a73ad8
GPR02: c000aec0 c000aec0 c000aec0 c000aec0 c000aec0 c000aec0
GPR03: d00dfeed d00dfeed d00dfeed d00dfeed d00dfeed d00dfeed
GPR04: 35ffc000 35ffc000 35ffc000 35ffc000 35ffc000 35ffc000
GPR05: c0a6f908 c0a6f908 c0a6f908 c0a6f908 c0a6f908 c0a6f908
GPR06: c197f0c0 c197f0c0 c197f0c0 c197f0c0 c197f0c0 c197f0c0
GPR07: c000aec0 c000aec0 c000aec0 c000aec0 c000aec0 c000aec0
GPR08: 00000009 00000000 00000009 00000009 00000009 00000009
GPR09: c0a45fe4 00000000 c0a45fe4 c0a45fe4 c0a45fe4 c0a45fe4
GPR10: c0a630c0 00000000 c0a630c0 c0a630c0 c0a630c0 c0a630c0
GPR11: 00000000 00000000 00000000 00000000 00000000 00000000
GPR12: 00000000 00000000 00000000 00000000 00000000 00000000
GPR13: 00000000 00000000 c0a45f78 00000000 00000000 00000000
GPR14: 00000000 00000000 c001b9e4 ffff1004 00000000 00000000
PC   : ffff0b80
CPSR : 200001d7
SPSR :          00000042 60000013 200001d7 60000193 00004000
```
r0 等于 0x35ffc000,  r14(lr) 等于 c001b9e4，结合 kernel 的反汇编，得到当前 kernel 运行在 of_fdt_unflatten_tree 函数中，r0 是该函数的第一个入参，正是 dtb 的虚拟地址，等于 0x35ffc000，而 dtb 的加载地址是 0x77ffc000，这样线性映射之后 0x77ffc000+0xC0000000=0x37ffc000，减去 kernel 的加载地址 0x2000000，计算出的虚拟地址正好等于 0x35ffc000，所以可以看出，kernel 在启动阶段，对物理地址到虚拟地址的转换都作为线性映射来计算的。添加环境变量 fdt_high=0xffffffff 可以解决这个问题。
[U-Boot,V2,2/5] [ARM: OMAP5: Set fdt_high to enable booting with Device tree](http://patchwork.ozlabs.org/patch/230368/)
While booting with dt blob, if fdt_high is not set to
0xffffffff, the dt blob gets relocated to a high ram address,
which the kernel is not able to use without HIGHMEM.

So set it to 0xffffffff to avoid the issue.

## 8. 从 ram 启动正常，但是烧写到 flash 不能启动
首先遇到的第一个问题是 DDR init 问题：

（1）刚上电，u-boot 从 flash 启动，运行地址是0，flash 这时候也被 map 到 0 地址，理所当然的将链接地址也设为 0；

（2）SDK 是将 flash_remap 放在 ddr_init 之前，因为 flash_remap 之后运行地址将变为flash地址0xa0000000，这样运行 ddr_init 时，链接地址（0x0）和运行地址（0xa0000000）不一样，将不能寻址放置在 rodata 段的配置数据，所以改为将 flash_remap 放在 ddr_init 之后。

问题来了，在运行 DDR init 的时候，一旦写寄存器 A9IC_SCU_FILTER_START 0xFFF00040， 程序跑飞，仿真器报“DAP: overrun during debug access”；经多次尝试，恢复到将 flash_remap 放在 ddr_init 之前，并将链接地址设为 0xa0000000 才可以。这样启动顺序是：start -->> flash_remap -->> DDR/CLK init -->>board_init_f -->> relocate -->> board_init_r。

第二个问题是 board_init_f 应该尽可能精简:
如果想 DDR 初始化之后 relocate 之前的 board_init_f 做一些初始化工作，在执行到 timer_init 时，程序 hang；如果使用仿真器设置断点在 timer_init，单步执行的时候程序跑飞，仿真器报“DAP: overrun during debug access”。所以在board_init_f仅计算出relocation地址之后就进行代码搬迁，将大部分初始化工作放在 board_init_r 中。

## 9.进入 kernel 后 reboot 命令无效，u-boot reset 上电板子重启 2 次
niata的 OBC 有两个 core，用的是 AMP(Asymmetric Multi-Processing) 架构，上电启动阶段，arm0 运行 u-boot，arm1 处于 reset 状态；之后 arm1 启动 kernle，arm0 运行 DSP 的 firmware。用一个表格可以表示启动顺序。增加这段代码之后板子没有 reset 问题了。

|arm0(u-boot)|arm1(kernel)|
|------|------|
|disable interrupts set the cpu to SVC32 mode|	in reset|
|No IRAM remapped to low 256KB|	in reset|
|Enable access to debugger|	in reset|
|cpu_init_cp15|	in reset
|cpu_init_crit --> lowlevel_init(DDR/CLK init)|	in reset|
|__main|	in reset
|board_init_f/relocation|	in reset
|write 0xe59ff018 to addr 0x0(ldr pc, [pc, #24])|in reset|
|write secondary_entry to addr 0x20|	in reset|
|write 0 to cpu reset register(core1 out of reset)|	running at secondary_entry/wait 0xf4000000|
|board_init_f/relocation/board_init_r/bootm|	running at secondary_entry/wait 0xf4000000|
|do_bootm_linux|	running at secondary_entry/wait 0xf4000000|
|start_on_arm1(copy arm1_init.S to 0x04)|	running at secondary_entry/wait 0xf4000000|
|start_on_arm1(write 0x04 to 0x74000000)|	pc jump to 0x04|
|wait for starting MSP firmware|	start kernel|

须将启动 arm1 的代码放到 arch/arm/lib/crt0.S，在 board_init_r 之前启动 arm1，从 flash 启动的时候，arm1 才能启动 kernel；

之前将 arm1 启动代码在 arch/arm/cpu/armv7/start.S 中，烧写 u-boot 到 flash 之后，arm1 打印出“Uncompressing Linux... done, booting the kernel.”之后不动；但是如果 u-boot 是从ram 启动的或者通过仿真器手动操作，arm1 均可启动 kernel；分析得出如果 u-boot 从flash 启动，在 arm1 被 release 之后，在 secondary_entry 中产生了 undefined 错误；后来将启动 arm1 的代码放到 arch/arm/lib/crt0.S 就可以了。

## 10.CONFIG_EXTRA_ENV_SETTINGS 添加变量不生效问题
在 include/configs/niata_nor16.h 中 CONFIG_EXTRA_ENV_SETTINGS 添加 fdt_high 和修改 tftp 为 tftpboot 均不生效，在网上找到一篇文章解释了这个问题：

>><https://stackoverflow.com/questions/12282184/u-boot-text-area-overflow-when-adding-command-defines>
u-boot always reads the saved environment variables first. These environment variables are typically in non-volatile memory (NOR or NAND flash, or others). If the CRC of the saved environment variables are correct the saved env variables are used. If you changed your CONFIG_EXTRA_ENV_SETTINGS it won't be used!
The values in the CONFIG_EXTRA_ENV_SETTINGS will only be used when you reset the env vars to default and save them: env default -f and saveenv

## 11. saveenv issue 失败问题

    niata> saveenv
    Saving Environment to Flash...
    Unable to save the rest of sector (122880)
    Protected 1 sectors

找到出问题的代码部分在文件 `common/env_flash.c` 中，从代码和 log 分析应该是从 malloc 堆中分配了 122880 个字节，malloc 函数返回 NULL 导致报错 Unable to save the rest of sector (122880)
```
<cmd/nvedit.c>
saveenv/do_env_save/saveenv()
<common/env_flash.c>
232         up_data = end_addr + 1 - ((long)flash_addr + CONFIG_ENV_SIZE);      
233         debug("Data to save 0x%lx\n", up_data);                             
234         if (up_data) {                                                      
235                 printf("up_data=%d\n", up_data);                            
236                 saved_data = malloc(up_data);                               
237                 if (saved_data == NULL) {                                   
238                         printf("Unable to save the rest of sector (%ld)\n", 
239                                 up_data);                                   
240                         goto done;                                          
241                 }                                                                            
```
将 up_data 需要分配的内存大小改为 1024 个字节，malloc 返回正确，没有报错；所以应该是分配 122880 字节个内存，空间不够导致，查看现malloc poll 的大小设置为 128K，非常接近要分配的 122880 字节，所以将其增大到 256K，问题解决。
