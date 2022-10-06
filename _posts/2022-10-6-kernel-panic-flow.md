---
layout: post
title: "Kernel panic flow"
category: debug
tags: panic smp_send_stop
author: jgsun
---

* content
{:toc}

# Overview
作为一个 Linux 平台软件工程师，时常面对 panic 的世界。每当早晨打开电脑，看到 teams 人影闪动："Jianguo，刚刚发现系统 crash 了，请有空看下现场吗？" 这时候我的内心充满了 panic， 真似 “衣带渐宽终不悔，为伊消得人憔悴”， 还似 “天不老，情难绝。心似双丝网，中有千千结” 啊！是时候再次了解下内核 panic 的流程了！








# panic flow
![image](/images/posts/oom/panic_flow.png)


# smp_send_stop
smp_send_stop is very useful, it will send the IPI_CPU_STOP to other cores online and other cores will dump the stack trace.
![image](/images/posts/oom/smp_send_stop.png)


# panic_print

|bit0|print all tasks info|
|---|---|
|bit1|print system memory info|
|bit2|print timer info|
|bit3|print locks info if CONFIG_LOCKDEP is on|
|bit4|print ftrace buffer|
|bit5|print all printk messages in buffer|
|bit6|print all CPUs backtrace (if available in the arch)|

So for example to print tasks and memory info on panic, user can:

echo 3 > /proc/sys/kernel/panic_print


# References
[Documentation for /proc/sys/kernel/ — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/sysctl/kernel.html?highlight=panic_print#panic-print)
[The kernel’s command-line parameters — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/kernel-parameters.html?highlight=panic_print)