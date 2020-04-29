---
layout: post
title: "OCTEON inter Core IPC by RPMsg/virtio"
categories: IO
tags: octeon, AMP linux, MIPS, RPMsg/virtio, openAMP
author: jgsun
---

* content
{:toc}
# 1. System Overvirew
项目使用OCTEON MIPS cn7130 4核处理器，运行AMP双linux系统，core0/2/3和core1分别运行Linux OS。本文基于Linux  RPMsg framework，实现Inter-Core IPC方案。













系统框图如下所示：
![image](/images/posts/rpmsg/system.png)

* 注：图中浅蓝色背景的是open-source的linux内核代码， 深蓝色背景是project开发的代码，同时也修改了 linux内核rpmsg_core和virtio 部分代码 。

系统框图体现了 RPMsg三层协议，额外 在 Transport Layer上增加了User interface Layer，用于向user space导出misc字符设备和网络设备。
关于 RPMsg协议，可参考  [RPMsg Messaging Protocol](https://github.com/OpenAMP/open-amp/wiki/RPMsg-Messaging-Protocol)
> The whole communication implementation can be separated in three different ISO/OSI layers - Transport, Media Access Control and Physical layer. Each of them can be implemented separately and for example multiple implementations of the Transport Layer can share the same implementation of the MAC Layer (VirtIO) and the Physical Layer. Each layer is described in following sections.
![image](/images/posts/rpmsg/rpmsg-protocol.jpg)

# 2. RPMsg dirver init
## 2.1 初始化框图
下图从设备驱动模型（总线，设备，驱动）的角度，描述了驱动初始化流程：包括3 条总线（platform_bus，virtio_bus和rpmsg_bus）及对应的 3个驱动（nokia_rpmsg_driver, virtio_ipc_driver 和 rpmsg_interface_drv）和3个设备（图中 未画出 platform bus）。
* 从驱动模型角度，可以从复杂的驱动逻辑中拨云见日，迅速摸清真相。
![image](/images/posts/rpmsg/driver-probe.png)

## 2.2 driver初始化 序列图
![image](/images/posts/rpmsg/rpmsg-driver-init.png)

* 注：上图是master端的 驱动初始化，remote端在buffer的处理上稍有不同，代码中使用rpmsg_role区分。

核心完成下列任务：
（1）创建收发TX/RX vritquque，注册 vritquque的 TX/RX的 notify(nokia_rpmsg_notify)/callback(rpmsg_recv_done)
（2）创建channel并创建3个endpoints（channel本身，misc设备和网络设备）
（3）注册misc字符设备和网络设备
（4）注册接收中断的callback（octeon_rpmsg_irq_handler）
（5）建立channle接收工作队列结构（rpmsg_channel_delaywork，处理函数是rpmsg_work_handler），插入接收队列链表rpmsg_dw_info[RX]
# 3. RPMsg IO序列图

## 3.1 RX
![image](/images/posts/rpmsg/rpmsg-rx.png)


## 3.1.1  rpmsg_miscdev_read
```
rpmsg_miscdev_read
    DECLARE_WAITQUEUE(wait, current)
    add_wait_queue(&misc_ept_priv->r_wait, &wait)
    __set_current_state(TASK_INTERRUPTIBLE)
    schedule()
    copy_to_user(ubuff, misc_ept_priv->mem, count)
    wake_up_interruptible(&misc_ept_priv->w_wait)
    remove_wait_queue(&misc_ept_priv->r_wait, &wait)
    __set_current_state(TASK_RUNNING)
```
## 3.1.2  rpmsg_work_handler
一旦收到IPI中断，其中断处理函数mailbox_interrupt调用 octeon_rpmsg_irq_handler，获取 channel之后的 调度该channle的工作队列处理函数 rpmsg_work_handler：
master端 rpmsg_work_handler流程：
```
rpmsg_work_handler
    vring_interrupt
        vq->vq.callback(&vq->vq) //调用rpmsg_probe创建vq时设置的rpmsg_recv_done
            virtqueue_get_buf
            rpmsg_recv_single(vrp, dev, msg, len)
                ept->cb(ept->rpdev, msg->data, msg->len, ept->priv //调用rpmsg_misc_dev_ept_cb
                virtqueue_add_inbuf //back fill inbuf
                    virtqueue_add
               virtqueue_get_buf //get next buffer?
```
## 3.2  RPMsg TX序列图
![image](/images/posts/rpmsg/rpmsg-tx.png)

## 3.3 rpmsg_miscdev_write
master端rpmsg_miscdev_write流程：
```
rpmsg_miscdev_write
    rpmsg_sendto(misc_ept_priv->rpmsg_chnl, tx_buff, size, misc_ept_priv->ept_num)
        ept->ops->sendto(ept, data, len, dst) //调用virtio_rpmsg_sendto
            rpmsg_send_offchannel_raw(rpdev, src, dst, data, len, true)
                msg = get_a_tx_buf(vrp)
                virtqueue_add_outbuf
                    virtqueue_add
                virtqueue_kick
                    virtqueue_notify
                        vq->notify(_vq) //调用nokia_rpmsg_notify
                            rpmsg_irq_trigger //调用 rpmsg_irq_trigger   发送IPI中断
```
## 3.4 misc设备初始化， I/O框图
![image](/images/posts/rpmsg/miscdev.png)

## 3.5 网络设备初始化，I/O框图
![image](/images/posts/rpmsg/ethdev.png)

# 4. DTS及mailbox中断机制
DTS指定 share memory地址和中断ID号。
```
  rpmsg0 {
   #address-cells = <0x2>;
   #size-cells = <0>;
   compatible = "octeon,nokia-rpmsg";
   input-msgirq-num = <4>; /* bit 4 of CIU_MBOX_CLR(0..3) */
   output-msgirq-num = <3>; /* bit 3 of CIU_MBOX_SET(0..3) */
   rpmsg-role = <1>;             /* 1:MASTER, 2:REMOTE */
   channel-index = <1>;
   channel-src = <11>;
   channel-dst = <12>;
   channel-name = "nokia-fsproxy-channel";
   misc-dev-endpoints = [0d];/* 0d in hex， 13 in decimal */
   eth-dev-endpoints =  [0e];/* 0e in hex， 14 in decimal */
   shmem-rx-vring   = <0 0x9f000000>;
   shmem-tx-vring   = <0 0x9f008000>;
   data-buffer-pool = <0 0x9f100000>;
   status = "okay";
  };
```
## 4.1 share memory地址
下图红圈部分，地址范围0x9F000000-0x9FFFFFFF一共16M， 在u-boot的dts里面设置memreserve。
![image](/images/posts/rpmsg/memory-map.png)


## 4.2 octeon mailbox中断机制

### 4.2.1 mailbox中断初始化
(1) 中断控制器初始化时， 分配 mailbox 中断中断描述符
```
octeon_irq_init_ciu
    irq_alloc_descs_from(OCTEON_IRQ_MBOX0, 2, 0)
        __irq_alloc_descs
            alloc_descs
                bitmap_set(allocated_irqs, start, cnt)
```
（2）内核启动调用 octeon_prepare_cpus注册中断handler
```
kernel_init_freeable
    smp_prepare_cpus
          octeon_prepare_cpus
               cvmx_write_csr(CVMX_CIU_MBOX_CLRX(coreid), mask)
               request_irq(OCTEON_IRQ_MBOX0, mailbox_interrupt, IRQF_PERCPU
```
request_irq加了 IRQF_PERCPU标志，表示OCTEON_IRQ_MBOX0（105）这个中断线是per cpu的，即每个CPU看到的是不同的控制寄存器。

### 4.2.2 mailbox中断 发送
通过调用octeon_smp_ops的send_ipi_single来发送IPI
`ops->send_ipi_single -->> octeon_smp_ops->octeon_send_ipi_single`
### 4.2.3 mainbox中断 接收
从前面知， mainbox中断 接收函数是 mailbox_interrupt，读取Mailbox Clear Registers，发现哪个bit置位，查找octeon_message_functions数组获取对应bit的处理函数并调用。
![image](/images/posts/rpmsg/octeon-mailbox-clear-reg.png)


![image](/images/posts/rpmsg/octeon-mailbox-set-reg.png)

一共 可以支持32种mainbox消息，目前内核使用了前4个bit（ scheduler_ipi，  generic_smp_call_function_interrupt， octeon_icache_flush和网络驱动cvm_oct_enable_napi   ）。
rpmsg irq驱动 调用octeon_request_ipi_handler注册每个channle的中断接收处理函数到 octeon_message_functions数组。

```
static void (*octeon_message_functions[8])(void) = {
 scheduler_ipi,
 generic_smp_call_function_interrupt,
 octeon_icache_flush,
#ifdef CONFIG_KEXEC
 octeon_crash_dump,
#endif
};
```
# 5. RPMsg/virtio通信机制

## 5.1  RPMsg framework overview
 关于 RPMsg framework，可参考    [Linux RPMsg framework overview]( https://wiki.st.com/stm32mpu/wiki/Linux_RPMsg_framework_overview ) .
> The RPMsg framework is a virtio-based messaging bus that allows a local processor to communicate with remote processors available on the system. 
The Linux® RPMsg framework is a messaging mechanism implemented on top of the virtio framework to communicate with a remote processor. It is based on virtio vrings to send/receive messages to/from the remote CPU over shared memory.
*   [Virtio: An I/O virtualization framework for Linux]( https://developer.ibm.com/articles/l-virtio/ )
* [virtio introduction - SlideShare](https://www.slideshare.net/zenixls2/052-virtio-introduction-17191942)


> The vrings are uni-directional, one vring is dedicated to messages sent to the remote processor, and the other vring is used for messages received from the remote processor. In addition, shared buffers are created in memory space visible to both processors.

The Mailbox framework is then used to notify cores when new messages are waiting in the shared buffers.

Relying on these frameworks, The RPMsg framework implements a communication based on channels. The channels are identified by a textual name and have a local (“source”) RPMsg address, and a remote (“destination”) RPMsg address”.

On the remote processor side, a RPMSG framework must also be implemented. Several solutions exist, we recommend using [OpenAMP](https://www.multicore-association.org/workgroup/oamp.php)

Overviews of the communication mechanisms involved are available at:  [OpenAMP wiki](https://github.com/OpenAMP/open-amp/wiki), blog web site is [Open-AMP Website Content](http://openamp.github.io/)
* [RPMsg Messaging Protocol]( https://github.com/OpenAMP/open-amp/wiki/RPMsg-Messaging-Protocol )
* [RPMsg Communication Flow](https://github.com/OpenAMP/open-amp/wiki/RPMsg-Communication-Flow)
* [Linux RPMsg framework overview]( https://wiki.st.com/stm32mpu/wiki/Linux_RPMsg_framework_overview )
*  [NXP_RPMSG_protocol_standardization_kick_off](http://openamp.github.io/docs/2016.10/NXP_RPMSG_protocol_standardization_kick_off.pdf)


## 5.2 virtio  Vring 结构

关于 virtio  Vring 结构，可参考：   [RPMsg Messaging Protocol](https://github.com/OpenAMP/open-amp/wiki/RPMsg-Messaging-Protocol)和 [OpenAMP RPMsg Virtio Implementation](https://github.com/OpenAMP/open-amp/wiki/OpenAMP-RPMsg-Virtio-Implementation)
> Vring is composed of three elementary parts - buffer descriptor pool, the “available” ring buffer (or input ring buffer) and the “used” ring buffer (or free ring buffer). All three elements are physically stored in the shared memory.

![image](/images/posts/rpmsg/virtio-circular-buffering.jpg)

  
![image](/images/posts/rpmsg/buffer-descriptor.jpg)

  
![image](/images/posts/rpmsg/ring-buffer-structure.jpg)

  


> A buffer management component called “VRING,” which is a ring data structure to manage buffer descriptors located in shared memory:

![image](/images/posts/rpmsg/rpmsg-buffer-management.jpg)


## 5.3  RPMsg channel, endpoints和misc，ethernet设备
### 65.3.1 图示
![image](/images/posts/rpmsg/rpmsg-endpoint-ch1.png)

channel1：

channel2：
![image](/images/posts/rpmsg/rpmsg-endpoint-ch2.png)


### 5.3.2 misc device
`/dev/rpmsg1_13`中，1是channel number， 13是endpoints number，分别对应dts里面的channle-index和misc-dev-endpoints

```
/mnt # ls /dev/rpmsg*
/dev/rpmsg1_13  /dev/rpmsg2_23
```
### 5.3.3 ethernet device
`/dev/rpmsg1_13`中，1是channel number， 14是endpoints number，分别对应dts里面的channle-index和eth-dev-endpoints
```
rpmsg1_14 Link encap:Ethernet  HWaddr 00:00:00:00:00:01  
          BROADCAST MULTICAST  MTU:482  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
rpmsg2_24 Link encap:Ethernet  HWaddr 00:00:00:00:00:01  
          BROADCAST MULTICAST  MTU:482  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
## 5.4  RPMsg Communication Flow
下图出自：[RPMsg Communication Flow](https://github.com/OpenAMP/open-amp/wiki/RPMsg-Communication-Flow)，在此基础上标注了相应的API。
![image](/images/posts/rpmsg/rpmsg-transaction-sequence-master-to-remote.jpg)

![image](/images/posts/rpmsg/rpmsg-transaction-sequence-remote-to-master.jpg)

下图从 vring使用情况的角度 描述了通信过程：
以单一channel为例，每个channel分配两快vring，每个 vring由 vring_desc, vring_avail和vring_used  3个部分组成，每个vring分256个vring_desc， vring_desc的 地址变量指向数据buffer，vring_avail和vring_used是存放参与通信的vring_desc ID，组成FIFO队列。
![image](/images/posts/rpmsg/vring-master-rx.png)
![image](/images/posts/rpmsg/vring-master-tx.png)

## 5.5 实验log
### 5.5.1 misc设备通信
core0 发送， u-boot设置启动参数logleve=8情况下：
```
/mnt # echo 1 > /dev/rpmsg1_13 
[ 1347.962087] (c01  3621        sh) rpmsg_ifc_drv virtio0.nokia-fsproxy-channel.11.12: TX From 0xb, To 0xd, Len 2, Flags 0, Reserved 0
[ 1347.974130] (c00  3621        sh) rpmsg_virtio tx: 00 00 00 0b 00 00 00 0d 00 00 00 00 00 02 00 00  
... ...
[ 1347.984695] (c00  3621        sh) rpmsg_virtio tx: 31 0a                                            1.
[ 1347.994043] (c00  3621        sh) virtqueue_add(359)desc[3].flags=0x0
[ 1348.000514] (c00  3621        sh) vq->vring.avail@800000009f009000
[ 1348.006713] (c00  3621        sh) Added buffer head 2 to 800000010c77e000, va = 800000009f120400, pa = 0x9F120400, avail = 2,aidx = 3
[ 1348.018740] (c00  3621        sh) virtqueue_kick_prepare(658)vq->vring.used->flags@800000009f00a000 = 0x0, needs_kick=1
[ 1348.029550] (c00  3621        sh) virtqueue_kick_prepare(662)needs_kick=1
[ 1348.036349] (c00  3621        sh) virtqueue_notify(679)
[ 1348.041603] (c00  3621        sh) virtqueue_notify(683)
[ 1348.046844] (c00  3621        sh) call octeon_rpmsg_irq_trigger 
[ 1348.052880] (c00  3621        sh) rpmsg_irq_trigger-105:descpu=17, tx-irq-pos=0x3
```
core1接收log：
```
# [ 1348.060415] (c00    26 kworker/0) virtqueue interrupt with no work for 800000009cb3c000
[ 1348.068444] (c00    26 kworker/0) virtqueue callback for 800000009cb38000(ffffffffc05a74b0)
[ 1348.076832] (c00    26 kworker/0) rpmsg_recv_done: rpmsg_recv_done(690)
[ 1348.083470] (c00    26 kworker/0) virtqueue_get_available_buffer(555)vq_available_idx=2@800000009cb3806e¼vq->vring.avail->idx=3
[ 1348.095056] (c00    26 kworker/0) vq->vring.avail@800000009f009000,@avail_idx=800000009d1c52b8, @len=800000009d1c52bc
[ 1348.105688] (c00    26 kworker/0) head_idx = 2, avail_idx = 2, len = 18, paddr = 0x9F120400.
[ 1348.114152] (c00    26 kworker/0) virtio_rpmsg_bus virtio0: rpmsg remote, len = 18, idx = 2, paddr = 0x9F120400, vaddr = 800000009f120400.
[ 1348.126609] (c00    26 kworker/0) virtio_rpmsg_bus virtio0: From: 0xb, To: 0xd, Len: 2, Flags: 0, Reserved: 0
[ 1348.136544] (c00    26 kworker/0) rpmsg_virtio RX: 00 00 00 0b 00 00 00 0d 00 00 00 00 00 02 00 00  ................
[ 1348.147087] (c00    26 kworker/0) rpmsg_virtio RX: 31 0a                                            1.
[ 1348.156414] (c00    26 kworker/0) rpmsg_recv_single(639)msg->dst=0xd
[ 1348.162792] (c00    26 kworker/0) rpmsg_ifc_drv virtio0.nokia-fsproxy-channel.12.11: rpmsg_misc_dev_ept_cb len rcved=2
[ 1348.173509] (c00    26 kworker/0) rpmsg_misc_dev_ept_cb31 0a                                            1.
[ 1348.183185] (c00    26 kworker/0) rpmsg_ifc_drv virtio0.nokia-fsproxy-channel.12.11: write 2 bytes, cur_len:6 
[ 1348.193210] (c00    26 kworker/0) virtqueue_get_available_buffer(555)vq_available_idx=3@800000009cb3806e¼vq->vring.avail->idx=3
[ 1348.204794] (c00    26 kworker/0) No avaiable buffer, vq_available_idx = 3, vq->vring.avail->idx = 3.
[ 1348.214032] (c00    26 kworker/0) I am in RX path
[ 1348.218747] (c00    26 kworker/0) virtio_rpmsg_bus virtio0: Received 1 messages
```
### 5.5.2 网络设备通信
设置corel0和core1网络接口IP地址为相同网段，就能相互ping通。
关闭debug log：  `echo 4 > /proc/sys/kernel/printk`
```
 ifconfig rpmsg1_14 192.1hu68.2.2 up
 ifconfig rpmsg2_24 10.10.2.2 up
 ifconfig rpmsg1_14 192.168.2.3 up
 ifconfig rpmsg2_24 10.10.2.3 up

 ping / # ping 192.168.2.3
 PING 192.168.2.3 (192.168.2.3): 56 data bytes
 64 bytes from 192.168.2.3: seq=0 ttl=64 time=12.536 ms
 64 bytes from 192.168.2.3: seq=1 ttl=64 time=14.290 ms
 # ping 10.10.2.2
 PING 10.10.2.2 (10.10.2.2): 56 data bytes
 64 bytes from 10.10.2.2: seq=1 ttl=64 time=30.054 ms
 64 bytes from 10.10.2.2: seq=1 ttl=64 time=34.636 ms
```

### 5.5.3 查询每个channle的中断计数
```
/ # cat /sys/rpmsg1/interrupts 
rx: 8
tx: 12
/ # cat /sys/rpmsg2/interrupts 
rx: 6
tx: 5
```
# 6. 参考资料
* http://openamp.github.io/
* https://github.com/OpenAMP/open-amp/wiki
* [NXP_RPMSG_protocol_standardization_kick_off](http://openamp.github.io/docs/2016.10/NXP_RPMSG_protocol_standardization_kick_off.pdf)
* [OpenAMP RPMsg Virtio Implementation]( https://github.com/OpenAMP/open-amp/wiki/OpenAMP-RPMsg-Virtio-Implementation )
* [RPMsg Messaging Protocol]( https://github.com/OpenAMP/open-amp/wiki/RPMsg-Messaging-Protocol )
* [Linux RPMsg framework overview]( https://wiki.st.com/stm32mpu/wiki/Linux_RPMsg_framework_overview )
* [Virtio: An I/O virtualization framework for Linux]( https://developer.ibm.com/articles/l-virtio/ )
* [ linux/Documentation/rpmsg.txt ]( https://github.com/torvalds/linux/blob/master/Documentation/rpmsg.txt )
* [ linux/Documentation/remoteproc.txt ]( https://github.com/torvalds/linux/blob/master/Documentation/remoteproc.txt )

