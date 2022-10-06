---
layout: post
title: "Measure irq_disable latency"
category: process_irq
tags: tmux tabby .tmux.conf
author: jgsun
---

* content
{:toc}

# Overview
The Linux on top of hypervisor has hrtimer overrun issue, 1.33ms hrtimer may has delay for more than 10ms.
The hypervisor supplier hope us check if there are overruns in native Linux.

> 测试用 patch：[hrtimer_test module and enable irqsoff tracer](https://github.com/jgsun/jgsun.github.io/blob/master/doc/patches/0001-meta-qti-base-hrtimer_test.ko_and-enable-irqsoff-tra.patch)











- **irqsoff tracer**

Traces the areas that disable interrupts and saves the trace with the longest max latency. See tracing_max_latency. When a new max is recorded, it replaces the old trace. It is best to view this trace with the latency-format option enabled, which happens automatically when the tracer is selected.

- **tracing_max_latency**

Some of the tracers record the max latency. For example, the maximum time that interrupts are disabled. The maximum time is saved in this file. The max trace will also be stored, and displayed by “trace”. A new max trace will only be recorded if the latency is greater than the value in this file (in microseconds).
By echoing in a time into this file, no latency will be recorded unless it is greater than the time in this file.

# Test and analysis
## Measure irqsoff latency
### Preparation: enable irqsoff tracer
The kernel config will be added:
```
CONFIG_CC_VERSION_TEXT="aarch64-oe-linux-gcc (GCC) 9.3.0"
CONFIG_CC_IS_GCC=y
CONFIG_GCC_VERSION=90300
CONFIG_CLANG_VERSION=0
CONFIG_CC_HAS_ASM_INLINE=y
CONFIG_RELAY=y
CONFIG_CC_HAVE_STACKPROTECTOR_SYSREG=y
CONFIG_STACKPROTECTOR_PER_TASK=y
CONFIG_LTO_NONE=y
CONFIG_HAVE_ARCH_PREL32_RELOCATIONS=y
CONFIG_PLUGIN_HOSTCC=""
# CONFIG_FORTIFY_SOURCE is not set
CONFIG_INIT_STACK_NONE=y
CONFIG_KASAN_STACK=1
CONFIG_HAVE_FUNCTION_GRAPH_TRACER=y
CONFIG_TRACER_MAX_TRACE=y
CONFIG_RING_BUFFER_ALLOW_SWAP=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_TRACE_PREEMPT_TOGGLE=y
CONFIG_PREEMPTIRQ_EVENTS=y
CONFIG_IRQSOFF_TRACER=y
CONFIG_PREEMPT_TRACER=y
CONFIG_SCHED_TRACER=y
CONFIG_HWLAT_TRACER=y
CONFIG_TRACER_SNAPSHOT=y
CONFIG_TRACER_SNAPSHOT_PER_CPU_SWAP=y
CONFIG_STACK_TRACER=y
CONFIG_BLK_DEV_IO_TRACE=y
CONFIG_KPROBE_EVENTS_ON_NOTRACE=y
CONFIG_KPROBES_DEBUG=y
CONFIG_TRACING_MAP=y
CONFIG_HIST_TRIGGERS=y
```
Secondly, enlarge the ftrace ring-buffer size
but, for some reason, the tracing_max_latency is more than 16ms, but no "trace" there, we need to enlarge the ftrace ring-buffer size

### irqsoff latency after enable dynamic debug in qseecom driver

```
root@sa81x5:/sys/kernel/debug/dynamic_debug# echo "file qseecom.c +p" > control
root@sa81x5:~# dmesg -n 8
root@sa81x5:/sys/kernel/debug/tracing# echo 100000 > buffer_size_kb
root@sa81x5:/sys/kernel/debug/tracing# echo irqsoff > current_tracer 
root@sa81x5:/sys/kernel/debug/tracing# echo 0 > options/function-trace
root@sa81x5:/sys/kernel/debug/tracing# echo 0 > tracing_max_latency 
root@sa81x5:/sys/kernel/debug/tracing# cat trace
# tracer: irqsoff
#
# irqsoff latency trace v1.1.5 on 5.4.161-qgki-debug
# --------------------------------------------------------------------
# latency: 12116 us, #4/4, CPU#7 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:8)
#    -----------------
#    | task: qseecom-unload--207 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
#  => started at: vprintk_emit
#  => ended at:   vprintk_emit
#
#
#                    _------=> CPU#            
#                   / _-----=> irqs-off        
#                  | / _----=> need-resched    
#                  || / _---=> hardirq/softirq 
#                  ||| / _--=> preempt-depth   
#                  |||| /     delay            
#  cmd     pid     ||||| time  |   caller      
#     \   /        |||||  \    |   /         
qseecom--207       7d..1    0us*: console_trylock_spinning <-vprintk_emit
qseecom--207       7d..1 12116us : console_trylock_spinning <-vprintk_emit
qseecom--207       7d..1 12117us : trace_hardirqs_on <-vprintk_emit
qseecom--207       7d..1 12118us : <stack trace>
 => vprintk_default
 => vprintk_func
 => printk
 => __dynamic_pr_debug
 => __qseecom_unload_app_kthread_func
 => kthread
 => ret_from_fork
```
Analysis: irq disable in console_trylock_spinning for more than 12ms.


### check the irq_disable by trace point
```
root@sa81x5:~# while true;do CMD="taskset -c 6 qseecom_sample_client -v smplap64 0 1";echo; echo "## Run cmd: 0"; echo $CMD; $CMD; sleep 5; echo; done
root@sa81x5:~# taskset -c 6 insmod /data/hrtimer_test.ko
root@sa81x5:/sys/kernel/debug/tracing# echo irq:irq_handler_entry > set_event
-sh: can't create set—_event: Permission denied
root@sa81x5:/sys/kernel/debug/tracing# echo irq:irq_handler_entry > set_event
root@sa81x5:/sys/kernel/debug/tracing# echo irq:irq_handler_exit >> set_event
root@sa81x5:/sys/kernel/debug/tracing# echo preemptirq:preempt_disable >> set_event
root@sa81x5:/sys/kernel/debug/tracing# echo preemptirq:preempt_enable >> set_event
jianguos@jianguos-gv:hrtimer$ adb shell cat /sys/kernel/debug/tracing/trace_pipe | grep "\[006\]" > trace_hrtimer_irq.txt &
jianguos@jianguos-gv:hrtimer$ tail -f trace_hrtimer_irq.txt | grep overrun
           <...>-6462    [006] d.h1 14372.743619: hrtimer_cb: 8 overruns
```
Analysis: In function console_unlock, the local cpu irq is disable for about 10ms, which introduce the hrtimer overruns.
```
           <...>-6462    [006] d..1 14372.732649: irq_disable: caller=console_unlock+0xc0/0x510 parent=vprintk_emit+0x134/0x1d0
           <...>-6462    [006] d..1 14372.743615: irq_enable: caller=console_unlock+0x39c/0x510 parent=vprintk_emit+0x134/0x1d0
           <...>-6462    [006] d..1 14372.743616: irq_disable: caller=el1_irq+0xb4/0x200 parent=console_unlock+0x3a0/0x510
           <...>-6462    [006] d.h1 14372.743617: irq_handler_entry: irq=3 name=arch_timer
           <...>-6462    [006] d.h1 14372.743618: hrtimer_cb: hrtimer stat: now: 14372743560 [us] expect_expires: 14372733903 [us], delta: 9656, next_expires: 14372744570
           <...>-6462    [006] d.h1 14372.743619: hrtimer_cb: 8 overruns
           <...>-6462    [006] d.h1 14372.743625: irq_handler_exit: irq=3 ret=handled
```
如果降低打印等级 “dmesg -n 1”，则 hrtimer test module 没有 overruns 情况。 

# Conclusion
On native Linux, the irqsoff latency is come from printk, we need to be careful to remove any console log from the system to ensure no overruns from the hrtimer.

The reason why qseecom_simple_test introduce the overruns is because the qseecom driver take pe_warn in three places, now I have a patch to lower their loglevel, which remove the overruns also.

This should be not the case for hypervisor based Linux, where printing happens into buffer and not to the hardware directly.


# Reference
[ftrace - irqsoff Tracer — The Linux Kernel documentation](https://docs.kernel.org/trace/ftrace.html#irqsoff)
