---
layout: post
title:  "浅析 linux kernel dynamic debug"
categories: debug
tags: Linux dyndbg pr_debug dev_dbg
author: jgsun
---

* content
{:toc}


## 概述

dynamic debug (dyndbg)是内核提供的一个调试功能，允许动态的开关内核打印输出，包含的API：pr_debug()、dev_dbg() 、print_hex_dump_debug()、rint_hex_dump_bytes()和netdev_dbg等。
dynamic debug通过设置<debugfs>/dynamic_debug/control文件来控制内核输出，有多种匹配的条件：文件名，函数名，行号，模块名和输出字符的格式；
control文件的语法格式：

> command ::= match-spec* flags-spec \
> 如：echo -n 'file svcsock.c line 1603 +p' >   <debugfs>/dynamic_debug/control












其中match-spec的关键词是：
```
match-spec ::= 'func' string |
               'file' string |
               'module' string |
               'format' string |
               'line' line-range
```
可以通过设置不同的flag输出函数名，行号，模块名和线程ID，这些信息可以帮助我们调试。
```
p enables the pr_debug() callsite.
f Include the function name in the printed message
l Include line number in the printed message
m Include module name in the printed message
t Include thread ID in messages not generated from interrupt context
_ No flags are set. (Or'd with others on input)
```
详见内核文档[Documentation/admin-guide/dynamic-debug-howto.rst](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/dynamic-debug-howto.rst)
## 内核配置
内核配置：CONFIG_DYNAMIC_DEBUG=y

![image](/images/posts/debug/dyndbg_menuconfig.png)

## 代码解读
### pr_debug(include/linux/printk.h)
pr_debug的定义在文件include/linux/printk.h，从pr_debug的源码注释建议：如果写驱动，请用dev_dbg。
```
/* If you are writing a driver, please use dev_dbg instead */
#if defined(CONFIG_DYNAMIC_DEBUG)
#include <linux/dynamic_debug.h>

/* dynamic_pr_debug() uses pr_fmt() internally so we don't need it here */
#define pr_debug(fmt, ...) \
 dynamic_pr_debug(fmt, ##__VA_ARGS__)
#elif defined(DEBUG)
#define pr_debug(fmt, ...) \
 printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
#else
#define pr_debug(fmt, ...) \
 no_printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
#endif
```

### dev_dbg(include/linux/device.h)
dev_debug的定义在文件include/linux/device.h
```
#if defined(CONFIG_DYNAMIC_DEBUG)
#define dev_dbg(dev, fmt, ...)	\
 dynamic_dev_dbg(dev, dev_fmt(fmt), ##__VA_ARGS__)
#elif defined(DEBUG)
#define dev_dbg(dev, fmt, ...)	\
 dev_printk(KERN_DEBUG, dev, dev_fmt(fmt), ##__VA_ARGS__)
#else
#define dev_dbg(dev, fmt, ...)	\
({	\
 if (0)	\
  dev_printk(KERN_DEBUG, dev, dev_fmt(fmt), ##__VA_ARGS__); \
})
#endif

```
根据CONFIG_DYNAMIC_DEBUG和DEBUG的配置，pr_debug/dev_dbg的输出分为以下情况：

| 配置                           | pr_debug/dev_dbg输出情况                                     |
| ------------------------------ | ------------------------------------------------------------ |
| CONFIG_DYNAMIC_DEBUG=y DEBUG=n | 调用dynamic_pr_debug/dynamic_dev_dbg， echo -n "file xxx.c +p" > /sys/kernel/debug/dynamic_debug/control |
| CONFIG_DYNAMIC_DEBUG=y DEBUG=y | 调用dynamic_pr_debug/dynamic_dev_dbg，增加启动参数loglevel=8之后，kernel启动阶段就能看到log |
| CONFIG_DYNAMIC_DEBUG=n DEBUG=y | 调用printk，打印等级是KERN_DEBUG=7，所以要将打印等级设置为8（echo 8 > /proc/sys/kernel/printk）才能看到输出 |
| CONFIG_DYNAMIC_DEBUG=n DEBUG=n | 不打印                                                       |

## 实现机制
首先看pr_debug宏展开情况：
* 原始： pr_debug("hello")
* 展开： dynamic_pr_debug("hello")
* 再展：
```
static struct _ddebug __aligned(8)	\
 __attribute__((section("__verbose"))) descriptor = {	\
  .modname = KBUILD_MODNAME,	\
  .function = __func__,	\
  .filename = __FILE__,	\
  .format = ("hello"),	\
  .lineno = __LINE__,	\
  .flags = _DPRINTK_FLAGS_DEFAULT,	\
  dd_key_init(key, init)	\
 }
 if (likely(descriptor.flags & _DPRINTK_FLAGS_PRINT))	\
  __dynamic_pr_debug(&descriptor, pr_fmt(fmt),	\
       ##__VA_ARGS__);	\
```

在调用pr_debug的源文件定义一个存放在kerenl的__verbose段的名为descriptor，类型为struct _ddebug变量；然后判断descriptor.flags & _DPRINTK_FLAGS_PRINT为true的话，就输出pr_debug内容。在定义descriptor时，将其flag成员赋值为_DPRINTK_FLAGS_DEFAULT，看这个宏的定义：
```
<dynamic_debug.h (include/linux)>
#if defined DEBUG
#define _DPRINTK_FLAGS_DEFAULT _DPRINTK_FLAGS_PRINT
#else
#define _DPRINTK_FLAGS_DEFAULT 0
#endif
```
可以看出，如果源文件定义了DEBUG宏，_DPRINTK_FLAGS_DEFAULT 等于_DPRINTK_FLAGS_PRINT，所以默认就是可以打印的；如果没有定义DEBUG宏，那么_DPRINTK_FLAGS_DEFAULT 就是0，上面的条件就不成立，就不会打印出来。通过写/sys/kernel/debug/dynamic_debug/control节点来动态修改descriptor.flags为_DPRINTK_FLAGS_PRINT来使条件成立。control属性节点在源码<lib/dynamic_debug.c>中实现。
接着看__dynamic_pr_debug函数：
```
void __dynamic_dev_dbg(struct _ddebug *descriptor,
        const struct device *dev, const char *fmt, ...)
{
 struct va_format vaf;
 va_list args;

 BUG_ON(!descriptor);
 BUG_ON(!fmt);

 va_start(args, fmt);

 vaf.fmt = fmt;
 vaf.va = &args;

 if (!dev) {
  printk(KERN_DEBUG "(NULL device *): %pV", &vaf);
 } else {
  char buf[PREFIX_SIZE];

  dev_printk_emit(LOGLEVEL_DEBUG, dev, "%s%s %s: %pV",
    dynamic_emit_prefix(descriptor, buf),
    dev_driver_string(dev), dev_name(dev),
    &vaf);
 }

 va_end(args);
}
```
可见__dynamic_dev_dbg的printk的打印等级是KERN_DEBUG=7，所以要想在启动阶段看到pr_debug的输出，还需要在启动参数里面设置loglevel=8，将打印等级设置为8。比如在qemu启动参数里面设置：
`-append "root=/dev/ram0 console=ttyAMA0 kmemleak=on loglevel=8" `
## 如何boot阶段打开动态打印？
###方法一：在需要打印的源码开头定义DEBUG宏
如在drivers/of/platform.c文件开头定义，注意一定要在#include之前定义才能生效。


### 方法二：在启动参数添加dyndbg="QUERY"，如qemu启动参数：
`-append "root=/dev/ram0 console=ttyAMA0 kmemleak=on loglevel=8 dyndbg=\"file drivers/of/irq.c +p\"" \`
上面两种方法都需要添加boot参数`loglevel=8`，将打印等级调整到8。
# 使用实例
## 使用pr_debug
举个例子，PXA1956内核设置CONFIG_DYNAMIC_DEBUG=y，所以pr_dubug使用的是动态打印方式。drivers/marvell/marvell-telephony/shmem/msocket.c里面的使用pr_err会不停打印出log，原先注释掉了，现在改用pr_debug使用动态方式按需打印，需要打印的时候打开动态打印开关，默认不打印。
```
--- a/drivers/marvell/marvell-telephony/shmem/msocket.c
+++ b/drivers/marvell/marvell-telephony/shmem/msocket.c
@@ -235,10 +235,8 @@ int msend(int sock, const void *buf, int len, int flags)

        if (!portq_is_synced(portq) || portq_xmit(portq, skb, block) < 0) {
                kfree_skb(skb);
-#if 0
-              pr_err("MSOCK: %s: port %d xmit error.\n",
+              pr_debug("MSOCK: %s: port %d xmit error.\n",
                      __func__, portq->port);
-#endif
                return -1;
        }
shell@pxa1956dkb:/ # echo 8 > /proc/sys/kernel/printk
shell@pxa1956dkb:/ # su 
shell@pxa1956dkb:/ # echo -n 'file msocket.c +p' > /sys/> /sys/kernel/debug/dynamic_debug/control
shell@pxa1956dkb:/ # [ 1416.261763] c5 2680 (ciImsDataInitTa) MSOCK: msend: port 9 xmit error.
[ 1418.521615] c5 2678 (ciCsdDataInitTa) MSOCK: msend: port 6 xmit error.
[ 1419.271904] c6 2680 (ciImsDataInitTa) MSOCK: msend: port 9 xmit error.
[ 1421.531695] c5 2678 (ciCsdDataInitTa) MSOCK: msend: port 6 xmit error.
[ 1422.281927] c7 2680 (ciImsDataInitTa) MSOCK: msend: port 9 xmit error.
```
### 仅定义DEBUG宏
以dirvers/of/驱动为例，先定义DEBUG选项，然后在menuconfig选择这个选项。
还要通过printk的procfs接口调整打印级别:
`echo 8 > /proc/sys/kernel/printk`
```
diff --git a/drivers/of/Kconfig b/drivers/of/Kconfig
index c6973f1..85fed7c 100644
--- a/drivers/of/Kconfig
+++ b/drivers/of/Kconfig
@@ -75,4 +75,11 @@ config OF_MTD
depends on MTD
def_bool y

+config OF_DEBUG
+ bool "device tree debugging"
+ depends on OF != n
+ help
+ This is an option for use by developers; most people should
+ say N here. This enables device tree and driver debugging.
+
endmenu # OF
diff --git a/drivers/of/Makefile b/drivers/of/Makefile
index efd0510..fd9b2a9 100644
--- a/drivers/of/Makefile
+++ b/drivers/of/Makefile
@@ -1,3 +1,4 @@
+ccflags-$(CONFIG_OF_DEBUG) := -DDEBUG
obj-y = base.o device.o platform.o
obj-$(CONFIG_OF_FLATTREE) += fdt.o
obj-$(CONFIG_OF_PROMTREE) += pdt.o
```
## 参考文档
* [Documentation/admin-guide/dynamic-debug-howto.rst](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/dynamic-debug-howto.rst)
* [pr_debug、dev_dbg等动态调试一](http://www.cnblogs.com/pengdonglin137/p/4621576.html)
* [pr_debug、dev_dbg等动态调试二](http://www.cnblogs.com/pengdonglin137/p/4622238.html)
* [pr_debug、dev_dbg等动态调试三](http://www.cnblogs.com/pengdonglin137/p/4622460.html)
* [默認打開pr_debug和dev_dbg](http://www.cnblogs.com/pengdonglin137/p/5808373.html)
* [如何打开pr_debug调试信息 ](http://blog.csdn.net/helloanthea/article/details/25330809)

