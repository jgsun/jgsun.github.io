---
layout: post
title: "Linux soft lockup"
category: debug
tags: panic soft_lockup
author: jgsun
---

* content
{:toc}

## Overview
The Linux kernel can act as a watchdog to detect both soft and hard lockups.

建议首先阅读内核文档来了解学习 Linux soft lockup，上面这句话就来自这篇文档，该文档是 soft lockup 的第一手资料。记得著名程序员左耳朵耗子有个访谈，说到学习看第一手资料的重要性！

我这篇文章纯属记录下学习的过程，供以后查询。
* [Softlockup detector and hardlockup detector (aka nmi_watchdog) — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/lockup-watchdogs.html)











## init view
![image](/images/posts/oom/soft_lockup_init.png)


## runtime view
![image](/images/posts/oom/soft_lockup_runtime.png)

### stop_one_cpu_nowait
The watchdog job runs in a stop scheduling thread that updates a timestamp every time it is scheduled. If that timestamp is not updated for 2*watchdog_thresh seconds (the softlockup threshold) the ‘softlockup detector’ (coded inside the hrtimer callback function) will dump useful debug information to the system log, after which it will call panic if it was instructed to do so or resume execution of other kernel code.

stop a cpu but don't wait for completion:
```
366  bool stop_one_cpu_nowait(unsigned int cpu, cpu_stop_fn_t fn, void *arg,
367  			struct cpu_stop_work *work_buf)
368  {
369  	*work_buf = (struct cpu_stop_work){ .fn = fn, .arg = arg, };
370  	return cpu_stop_queue_work(cpu, work_buf);
371  }
```
### is_softlockup
```
315  static int is_softlockup(unsigned long touch_ts)
316  {
317  	unsigned long now = get_timestamp();
318  
319  	if ((watchdog_enabled & SOFT_WATCHDOG_ENABLED) && watchdog_thresh){
320  		/* Warn about unreasonable delays. */
321  		if (time_after(now, touch_ts + get_softlockup_thresh()))
322  			return now - touch_ts;
323  	}
324  	return 0;
325  }
```
## 案例分析： 长时间 disable local cpu interrupt 引起的 soft lockup
这个问题来自某 hypervisor 的 bug，在 SMC call 期间，其虚拟的 TZ，长时间没有返回 Linux 处理 pending 的中断，其中也包括 watchdog timer 中断，watchdog job 长时间得到调度喂狗，导致了 soft lockup。这有点类似 ‘hardlockup’.
- **hard lockup**
  
  A ‘hardlockup’ is defined as a bug that causes the CPU to loop in kernel mode for more than 10 seconds (see “Implementation” below for details), without letting other interrupts have a chance to run.

这里 interrupt 都 disable 了超过 20s，如果使能了 hard lockup，就应该报 “NMI watchdog: Watchdog detected hard LOCKUP on cpu 0” 了。

![image](/images/posts/oom/soft_lockup_smc.png)
