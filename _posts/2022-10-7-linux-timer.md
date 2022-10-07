---
layout: post
title: "Linux arch_timer"
category: process
tags: timer timer_init hrtimer sched_clock local_clock
author: jgsun
---

* content
{:toc}

## Overview
Linux arch_timer 是内核最重要的基础设施之一，内核术语中，timer 带有前缀arch，可见其地位之高！内核之 arch_timer 提供系统运行的节拍，是其他定时任务的基础，相当于人之心脏！








## init view
### arch_timer 的启动 log
```
[    0.000000] (       swapper/0-0     [00]   20) arch_timer: CPU0: Trapping CNTVCT access
[    0.000000] (       swapper/0-0     [00]   20) arch_timer: cp15 and mmio timer(s) running at 19.20MHz (virt/virt).
[    0.000000] (       swapper/0-0     [00]   20) clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x46d987e47, max_idle_ns: 440795202767 ns
[    0.000006] (       swapper/0-0     [00]   20) sched_clock: 56 bits at 19MHz, resolution 52ns, wraps every 4398046511078ns
[    0.000014] (       swapper/0-0     [00]   20) arch_counter_register: arch_timer_read_counter()/sched_clock()/arch_timer_read_counter(): 0x6596d05/0x29b5/0x6596d07
[    0.000047] (       swapper/0-0     [00]   20) clocksource: Switched to clocksource arch_sys_counter
```
从启动 log 可以看到系统 counter 的晶振是 19MHz，决定了定时精度是 52ns。

### start_kernel → time_init
![image](/images/posts/process/timer/time_init.png)

### time_init → __arch_timer_setup
![image](/images/posts/process/timer/arch_timer_setup.png)

### time_init → arch_timer_common_init
![image](/images/posts/process/timer/arch_timer_common_init.png)
In arch_counter_register, we will reinitialize arch_timer_read_counter to arch_counter_get_cntvct_stable, then initialize two global variables, which will be used to get time in kernel, we will see them later in this article.

**struct clock_data cd**
![image](/images/posts/process/timer/timer_cd.png)

**tk_core**
![image](/images/posts/process/timer/timer_tk_core.png)


## timer interrupt

### arch_timer_handler_virt → tick_handle_periodic → switch to hrtimer
![image](/images/posts/process/timer/switch_to_hrtimer.png)

### arch_timer_handler_virt → hrtimer_interrupt 
![image](/images/posts/process/timer/hrtimer_interrupt.png)

### hrtimer_interrupt → tick_sched_timer
![image](/images/posts/process/timer/tick_sched_timer.png)

### hrtimer_interrupt → watchdog_timer_fn
![image](/images/posts/process/timer/watchdog_timer_fn.png)

## How to get time in kernel
### local_clock and sched_clock
![image](/images/posts/process/timer/sched_clock.png)

### ktime_get
![image](/images/posts/process/timer/ktime_get.png)

## References
* [timers — The Linux Kernel documentation](https://docs.kernel.org/timers/index.html)
* [ktime accessors — The Linux Kernel documentation](https://docs.kernel.org/core-api/timekeeping.html)
* [Hrtimers and Beyond: Transforming the Linux Time Subsystems](https://www.kernel.org/doc/ols/2006/ols2006v1-pages-333-346.pdf)
* <http://www.cs.columbia.edu/~nahum/w6998/papers/ols2006-hrtimers-slides.pdf>
