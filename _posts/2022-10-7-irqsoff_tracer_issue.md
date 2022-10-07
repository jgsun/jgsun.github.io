---
layout: post
title: "WARNING: CPU: 7 PID: 939 at kernel/trace/trace.c:1635 update_max_tr_single"
category: debug
tags: irqsoff_tracer update_max_tr_single buffer_size_kb
author: jgsun
---

* content
{:toc}

## Overview
The customer reflected the hrtimer has big delay, taking the irqsoff tracer to check the irq disable latency.
> About irqsoff tracer: [irqsoff Tracer — The Linux Kernel documentation](https://docs.kernel.org/trace/ftrace.html#irqsoff)

But when echoing irqsoff to current_tracer, the kernel has WARNING as below, at the same time, no call trace from the trace as well.









## Issue description
Kernel has WARNING and no call_trace when enable irqsoff tracer?
```
[   98.465607] ------------[ cut here ]------------                                                           
[   98.470376] WARNING: CPU: 7 PID: 939 at kernel/trace/trace.c:1635 update_max_tr_single+0xc0/0x130          
[   98.479489] Modules linked in: hdmi_dlkm cnss2 device_management_service_v01 wlan_firmware_service_v01 pcim
[   98.503424] CPU: 7 PID: 939 Comm: sh Tainted: G S                5.4.161-qgki-debug #1                     
[   98.511554] Hardware name: ADP-STAR (DT)                                                                   
[   98.515595] pstate: 804003c5 (Nzcv DAIF +PAN -UAO)                                                         
[   98.520530] pc : update_max_tr_single+0xc0/0x130                                                           
[   98.525288] lr : update_max_tr_single+0x90/0x130                                                           
[   98.530039] sp : ffffffc0213a3da0                                                                          
[   98.533458] x29: ffffffc0213a3da0 x28: ffffffa3af3a1b40                                                    
[   98.538928] x27: 0000000000000000 x26: 0000000000000000                                                    
[   98.544395] x25: 0000000000000446 x24: ffffffdff919a000                                                    
[   98.549868] x23: ffffffdff91fb5d0 x22: ffffffa3fe305dd8                                                    
[   98.555336] x21: ffffffa3af3a1b40 x20: 0000000000000007                                                    
[   98.560801] x19: ffffffdff91fb5d0 x18: ffffffc01e42d020                                                    
[   98.566269] x17: 0000000000000000 x16: ffffffdff83cf4b0                                                    
[   98.571737] x15: 0000000000000b20 x14: 0000000000000b20                                                    
[   98.577205] x13: ffffffa119d78a80 x12: ffffffa105f48400                                                    
[   98.582671] x11: ffffffa105f49000 x10: 00000000000000ff                                                    
[   98.588135] x9 : 0000000000000162 x8 : 00000000ffffffea                                                    
[   98.593602] x7 : 0000000000000000 x6 : ffffffa119d7eb30                                                    
[   98.599070] x5 : 0000000000000080 x4 : 0000000000000000                                                    
[   98.604537] x3 : ffffffa119d7eb1c x2 : 0000000000000007                                                    
[   98.610002] x1 : ffffffa105f40800 x0 : 00000000ffffffea                                                    
[   98.615467] Call trace:                                                                                    
[   98.617993]  update_max_tr_single+0xc0/0x130                                                               
[   98.622395]  tracer_hardirqs_on+0x220/0x2a0                                                                
[   98.626705]  trace_hardirqs_on+0xb8/0x120
```
## Check the source code of WARNING
```
jianguos@jianguos-gv:ramdump$ ./decode_stacktrace.sh vmlinux /local/mnt/workspace/lv.au.121/kernel/msm-5.4/ < call_track.log 
[  272.316484] Call trace:
[  272.319010] update_max_tr_single (/local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/kernel/trace/trace.c:1635) 
[  272.323414] tracer_hardirqs_on (/local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/kernel/trace/trace_irqsoff.c:361 /local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/kernel/trace/trace_irqsoff.c:436 /local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/kernel/trace/trace_irqsoff.c:618) 
[  272.327723] trace_hardirqs_on (/local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/include/linux/compiler.h:268 /local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/arch/arm64/include/asm/preempt.h:46 /local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/kernel/trace/trace_preemptirq.c:30) 
[  272.331860] do_notify_resume (/local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/arch/arm64/include/asm/daifflags.h:83 /local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/arch/arm64/kernel/signal.c:913) 
[  272.335899] work_pending (/local/mnt/workspace/lv.au.121/poky/build/tmp-glibc/work-shared/sa81x5/kernel-source/arch/arm64/kernel/entry.S:1003) 
```
The corrosponding source code:
```
1622      ret = ring_buffer_swap_cpu(tr->max_buffer.buffer, tr->trace_buffer.buffer, cpu);
1623  
1624      if (ret == -EBUSY) {
1625          /*
1626           * We failed to swap the buffer due to a commit taking
1627           * place on this CPU. We fail to record, but we reset
1628           * the max trace buffer (no one writes directly to it)
1629           * and flag that it failed.
1630           */
1631          trace_array_printk_buf(tr->max_buffer.buffer, _THIS_IP_,
1632              "Failed to swap buffers due to commit in progress\n");
1633      }
1634  
1635      WARN_ON_ONCE(ret && ret != -EAGAIN && ret != -EBUSY);
```
By checking, it may be related to buffer size. As expected, after enlarging the buffer size, issue got fixed.
##  Solution: enlarge ftrace buffer size
```
cd /sys/kernel/debug/tracing/
echo 100000 > buffer_size_kb 
echo irqsoff > current_tracer
echo 0 > option/function-tracer
root@sa81x5:/sys/kernel/debug/tracing# cat trace
# tracer: irqsoff
#
# irqsoff latency trace v1.1.5 on 5.4.161-qgki-debug
# --------------------------------------------------------------------
# latency: 214 us, #4/4, CPU#2 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:8)
#    -----------------
#    | task: swapper/2-0 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
#  => started at: arch_cpu_idle
#  => ended at:   arch_cpu_idle
#
#
#                    _------=> CPU#            
#                   / _-----=> irqs-off        
#                  | / _----=> need-resched    
#                  || / _---=> hardirq/softirq 
#                  ||| / _--=> preempt-depth   
#                  |||| /     delay            
#  cmd     pid     ||||| time  |   caller      
#     \   /        |||||  \    |   /         
  <idle>-0         2d..1    0us!: el1_irq <-arch_cpu_idle
  <idle>-0         2dN.1  213us : el1_irq <-arch_cpu_idle
  <idle>-0         2dN.1  216us : trace_hardirqs_on <-arch_cpu_idle
  <idle>-0         2dN.1  221us : <stack trace>
 => cpuidle_idle_call
 => do_idle
 => cpu_startup_entry
 => secondary_start_kernel
root@sa81x5:/sys/kernel/debug/tracing# cat tracing_max_latency 
214
```
