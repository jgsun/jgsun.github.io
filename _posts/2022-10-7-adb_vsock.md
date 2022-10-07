---
layout: post
title: "adb communication based on vsock"
category: I/O
tags: timer timer_init hrtimer sched_clock local_clock
author: jgsun
---

* content
{:toc}

## Overview
某项目 adb 通讯架构如下，两个系统 Linux 和 Android 运行在某 type-1 hypervisor 上，user 通过 adb 同两个系统通讯。
- 和 Linux 系统的 adbd 连接简单点，直接和 Linux 系统的 adbd 通讯。
- 和 Android 系统 adb 连接要复杂一些：
  - adbd-relay 作为 vsock network 和 usb 之间的桥梁；
  - Android 系统 和 Linux 系统之间采用 vsock 通讯；
  - virtio-vsock backend device 与 vhost-vsock 和 mmio_help 内核模块通过 eventfd 和 irqfd 控制 vsock 报文的收发。













![image](/images/posts/network/vsock/adb_arch.png)
## vsock high-level arch
![image](/images/posts/network/vsock/vsock_arch.png)

图片来自 <https://static.sched.com/hosted_files/kvmforum2019/50/KVMForum_2019_virtio_vsock_Andra_Paraschiv_Stefano_Garzarella_v1.3.pdf>


## Linux OS vhost_vsock driver init
![image](/images/posts/network/vsock/linux_vhost_vsock_init.png)

## Android OS virtio_vsock driver init 
![image](/images/posts/network/vsock/android_virtio_vsock_init.png)


## vsock virtio packet buff size 带来的的连接断开问题
研究这个 adb 连接的契机也是来自一个问题：长时间测试后，和 Android 系统的 adb 总是容易断开，重启 Android 系统之后又能恢复。明显是 Android 这边的 virtio vsock 出现了问题，通过 enable vsock trace event 也得出了这个结论。

最后查明原因是： 默认 vsock virtio packet buff size 是64KB，长时间运行之后 Android 系统产生了过多的内存碎片导致不能分配长达 64KB 的 buffer 而失败，解决方案是将 buffer size 降低到 4KB。

Google 2021 年发现了这个问题，有一个 patch 将这个 buffer size 改为 configurable：[	[PATCH 1/1] virtio/vsock: Make vsock virtio packet buff size configurable](https://lkml.org/lkml/2021/7/21/561)
### Enable vsock trace event on both Linux and Android
```
cd /sys/kernel/debug/tracing
echo 'vsock:*' >> set_event
echo 1 > tracing_on
echo 0 > trace
echo irq_handler_entry >> set_event
cd /sys/kernel/debug/tracing/events/irq/irq_handler_entry
echo "name!='arch_timer'&&name!='arch_mem_timer'&&name!='dwc3'&&name!='ufshcd'&&name!='apps_rsc'&&name!='ttyS0'" > filter    
cd /sys/kernel/debug/tracing
```
### Invock adb connection to Android
`jianguos@jianguos-gv:html$ adb -t 2 shell`
### Check the vsock trace event in Linux
```
root@opsy-sa81x5:/sys/kernel/debug/tracing# cat trace
# tracer: nop
#
# entries-in-buffer/entries-written: 1/1   #P:3
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
 ->vsock:1000:55-971     [000] .... 18409.529434: virtio_transport_alloc_pkt: 2:2844408420 -> 1000:5555 len=50 type=STREAM op=RW flags=0x0
```
### Check the vsock trace event in Android
```
console:/sys/kernel/debug/tracing # cat trace                                  
# tracer: nop
#
# entries-in-buffer/entries-written: 4/4   #P:2
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |  
          <idle>-0       [000] d.h1 66016.708821: irq_handler_entry: irq=12 name=virtio6
     kworker/0:0-18178   [000] .... 66016.708919: virtio_transport_recv_pkt: 2:2844408420 -> 1000:5555 len=58 type=STREAM op=RW flags=0x0 buf_alloc=262144 fwd_cnt=7336588
            adbd-3803    [001] .... 66016.717478: virtio_transport_alloc_pkt: 1000:5555 -> 2:2844408420 len=24 type=STREAM op=RW flags=0x0
            adbd-3803    [001] .... 66016.729808: virtio_transport_alloc_pkt: 1000:5555 -> 2:2844408420 len=24 type=STREAM op=RW flags=0x0
            adbd-3803    [001] .... 66016.729812: virtio_transport_alloc_pkt: 1000:5555 -> 2:2844408420 len=25 type=STREAM op=RW flags=0x0
```
### Conclusion
- Linux 发送了 vsock 报文，但是没有收到任何 vsock 报文，也没有收到报文的中断。
- Android 收到了 vsock 报文，也 virtio_transport_alloc_pkt 准备发送报文，但是最后没有发送成功。


## References
* <https://vmsplice.net/~stefan/stefanha-kvm-forum-2015.pdf>
* <https://static.sched.com/hosted_files/devconfcz2020a/b1/DevConf.CZ_2020_vsock_v1.1.pdf>
* <https://stefano-garzarella.github.io/posts/2019-11-08-kvmforum-2019-vsock/>
* <https://stefano-garzarella.github.io/posts/2021-01-22-socat-vsock/>
* <https://stefano-garzarella.github.io/posts/2020-02-20-vsock-nested-vms-loopback>







