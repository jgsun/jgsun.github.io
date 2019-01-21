---
layout: post
title:  "图解linux tcpdump"
categories: network
tags: linux tcpdump
author: jgsun
---

* content
{:toc}

[TOC]
# 概述
&#8194;PF_PACKET协议是专门用于抓包的，往系统网络层注册一个协议，分为两种方式：第一种方法是通过套接字，打开指定的网卡，然后使用recvmsg读取，实际过程需要需要将报文从内核区拷贝到用户区。第二种方法是使用packet_mmap，使用共享内存方式，在内核空间中分配一块内核缓冲区，然后用户空间程序调用mmap映射到用户空间。将接收到的skb拷贝到那块内核缓冲区中，这样用户空间的程序就可以直接读到捕获的数据包了。PACKET_MMAP减少了系统调用，不用recvmsg就可以读取到捕获的报文，相比原始套接字+recvfrom的方式，减少了一次拷贝和一次系统调用。libpcap就是采用第二种方式。
往外发的包和进来的包都会调到[net/packet/af_packet.c](https://github.com/torvalds/linux/blob/master/net/packet/af_packet.c)这个文件里面的packet_rcv函数（PACKET_MMAP调用的是tpacket_rcv()函数），其中outgoing方向（出去的包）会在dev_queue_xmit_nit里面遍历ptype_all链表进行所有网络协议处理的时候调用到packet_rcv；incoming方向（从外面其他机器进来的包）会在__netif_receive_skb_core函数里面同样办法遍历ptype_all进行处理的时候调用到packet_rcv。









用两张图分别描述tcpdump两种实现方式：原始套接字+recvfrom的方式和pcap_mmap共享内存方式，包含用户空间和内核空间。
## 原始套接字+recvfrom的方式
![image](/images/posts/network/tcpdump/pcap_recvfrom.png)
## pcap_mmap共享内存方式
![image](/images/posts/network/tcpdump/pcap_mmap.png)
PACKET_MMAP减少了系统调用，不用recvfrom就可以读取到捕获的报文，相比原始套接字+recvfrom的方式，减少了一次拷贝和一次系统调用。libpcap默认采用PACKET_MMAP方式，只有在内核不支持PF_PACKET协议的情况下，才采用recvfrom方式（极少情况了）。下面详细讲述PACKET_MMAP的实现方式。
# 用户空间应用程序
> about tcpdump
* [www.tcpdump.org](https://www.tcpdump.org/)

## main函数
```
main
    pd = open_interface(device, ndo, ebuf)
    pcap_setfilter(pd, &fcode) //fcode类型struct bpf_program，用于设置过滤器
    pcap_loop(pd, cnt, callback, pcap_userdata)
```
## open_interface打开socket和创建mmap映射
```
pc = pcap_create(device, ebuf)                                                                
    p = pcap_create_interface(device_str, errbuf) //pcap_t *p
        handle = pcap_create_common(ebuf, sizeof (struct pcap_linux)) //pcap_t *handle 
        handle->activate_op = pcap_activate_linux;
        handle->can_set_rfmon_op = pcap_can_set_rfmon_linux
pcap_set_promisc(pc, !pflag) 
pcap_set_timeout(pc, 1000)
pcap_activate(pc)
    pcap_activate_linux
        handle->setfilter_op = pcap_setfilter_linux
        handle->read_op = pcap_read_linux
        activate_new(handle) //打开PF_PACKET套接字
            socket(PF_PACKET, SOCK_RAW, protocol)
        activate_mmap(handle, &status)
            prepare_tpacket_socket(handle)//set memory-mapped header version 
                getsockopt(handle->fd, SOL_PACKET, PACKET_HDRLEN, &val, &len)
                setsockopt(handle->fd, SOL_PACKET, PACKET_VERSION, &val,
                setsockopt(handle->fd, SOL_PACKET, PACKET_RESERVE, &val,
            create_ring(handle, status)//set up memory-mapped access
                setsockopt(handle->fd, SOL_PACKET, PACKET_RX_RING,
                handlep->mmapbuflen = req.tp_block_nr * req.tp_block_size
                handlep->mmapbuf = mmap(0, handlep->mmapbuflen,
            handle->read_op = pcap_read_linux_mmap_v3
        activate_old(handle) //当打开PF_PACKET套接字失败后
            handle->fd = socket(PF_INET, SOCK_PACKET, htons(ETH_P_ALL))
```
1. socket系统调用第三个参数protocol是ETH_P_ALL
2. setsockopt系统调用设置PACKET_HDRLEN，PACKET_VERSION等参数
3. setsockopt系统调用分配内核环形缓冲区
4. mmap系统调用将内核环形缓冲区映射到用户空间地址
5. 如果打开PF_PACKET套接字失败，打开PF_INET协议簇type是SOCK_PACKET的套接字

## pcap_setfilter设置BPF过滤器
```
p->setfilter_op(p, fp)//调用pcap_setfilter_linux
    set_kernel_filter(handle, &fcode)
        setsockopt(handle->fd, SOL_SOCKET, SO_ATTACH_FILTER,
```
## pcap_loop接收打印报文
```
pcap_loop(pd, cnt, callback, pcap_userdata) //callback = print_packet
    n = p->read_op(p, cnt, callback, user) //将调用pcap_read_linux_mmap_v3
        pcap_wait_for_frames_mmap
            poll(&pollinfo, 1, handlep->poll_timeout)
        h.raw = RING_GET_CURRENT_FRAME(handle) //读取缓冲区
        pcap_handle_packet_mmap
           callback(user, &pcaphdr, bp) //调用print_packet打印
```
# PF_PACKET协议的内核实现
先讲PF_PACKET协议的内核初始化初始化，然后从系统调用的角度讲述如何使用PACKET_MMAP。
## PF_PACKET协议初始化
```
module_init(packet_init)
    proto_register(&packet_proto, 0) //注册struct proto packet_proto传输层网络实现函数集
    sock_register(&packet_family_ops) //注册PF_PACKET协议簇
    register_pernet_subsys(&packet_net_ops)
        proc_create_net("packet", 0, //创建/proc/net/packet文件
    register_netdevice_notifier(&packet_netdev_notifier)//注册netdevice通知链
        .notifier_call =	packet_notifier
```
1. proto_register(&packet_proto, 0)注册的传输层网络实现函数集struct proto packet_proto非常简单，只有一个有效成员变量obj_size = sizeof(struct packet_sock)用于在socket系统调用创建socket的时候指定packet_sock的大小；和PF_NETLINK协议一样，PF_PACKET协议的实现也全部由套接字层操作函数集struct proto_ops packet_ops实现（在socket系统调用时由创建socket的函数packet_create赋值：`sock->ops = &packet_ops`）。
2. sock_register(&packet_family_ops)注册PF_PACKET协议簇，提供该协议簇创建套接字的函数packet_create。
3. proc_create_net("packet"创建proc文件packet。
4. register_netdevice_notifier(&packet_netdev_notifier)注册netdevice通知链，在网络设备发生改变（up，down等）时被通知执行。
## 从tcpdump的strace输出分析实现PACKET_MMAP的系统调用
```
root@~# strace tcpdump -i eth0
......
socket(AF_PACKET, SOCK_RAW, 768) = 3
ioctl(3, SIOCGIFINDEX, {ifr_name="lo", }) = 0
ioctl(3, SIOCGIFHWADDR, {ifr_name="eth0", ifr_hwaddr={sa_family=ARPHRD_ETHER, sa_data=00:00:00:01:00:01}}) = 0
newfstatat(AT_FDCWD, "/sys/class/net/eth0/wireless", 0x7fee51d7d0, 0) = -1 ENOENT (No such file or directory)
ioctl(3, SIOCBONDINFOQUERY, 0x7fee51d728) = -1 EOPNOTSUPP (Operation not supported)
ioctl(3, SIOCGIWNAME, 0x7fee51d780) = -1 EINVAL (Invalid argument)
ioctl(3, SIOCGIFINDEX, {ifr_name="eth0", }) = 0
bind(3, {sa_family=AF_PACKET, sll_protocol=htons(ETH_P_ALL), sll_ifindex=if_nametoindex("eth0"), sll_hatype=ARPHRD_NETROM, sll_pkttype=PACKET_HOST, sll_halen=0}, 20) = 0
getsockopt(3, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
setsockopt(3, SOL_PACKET, PACKET_ADD_MEMBERSHIP, {mr_ifindex=if_nametoindex("eth0"), mr_type=PACKET_MR_PROMISC, mr_alen=0, mr_address=}, 16) = 0
[ 442.300518] device eth0 entered promiscuous mode
setsockopt(3, SOL_PACKET, PACKET_AUXDATA, [1], 4) = 0
getsockopt(3, SOL_SOCKET, SO_BPF_EXTENSIONS, [64], [4]) = 0
mmap(NULL, 266240, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fbd3cb000
getsockopt(3, SOL_PACKET, PACKET_HDRLEN, [32], [4]) = 0
setsockopt(3, SOL_PACKET, PACKET_VERSION, [1], 4) = 0
setsockopt(3, SOL_PACKET, PACKET_RESERVE, [4], 4) = 0
ioctl(3, SIOCETHTOOL, 0x7fee51d958) = 0
getsockopt(3, SOL_SOCKET, SO_TYPE, [3], [4]) = 0
getsockopt(3, SOL_PACKET, PACKET_RESERVE, [4], [4]) = 0
setsockopt(3, SOL_PACKET, PACKET_RX_RING, 0x7fee51da78, 28) = 0
mmap(NULL, 3670016, PROT_READ|PROT_WRITE, MAP_SHARED, 3, 0) = 0x7fbd04b000
......
setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, {len=1, filter=0x7fbd59a3c8}, 16) = 0
fcntl(3, F_GETFL) = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK) = 0
recvfrom(3, 0x7fee51db08, 1, MSG_TRUNC, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
fcntl(3, F_SETFL, O_RDWR) = 0
setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, {len=1, filter=0x632f10}, 16) = 0
rt_sigaction(SIGUSR1, {sa_handler=0x40fec0, sa_mask=[], sa_flags=0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
write(2, "tcpdump: verbose output suppress"..., 75tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
) = 75
write(2, "listening on eth0, link-type EN1"..., 74listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
) = 74
fstat(1, {st_mode=S_IFCHR|0600, st_rdev=makedev(204, 64), ...}) = 0
ioctl(1, TCGETS, {B38400 opost isig icanon echo ...}) = 0
write(1, "00:07:22.118231 STP 802.1d, Conf"..., 9900:07:22.118231 STP 802.1d, Config, Flags [none], bridge-id 8000.36:d6:2e:e8:78:5f.8002, length 35
) = 99
ppoll([{fd=3, events=POLLIN}], 1, {tv_sec=1, tv_nsec=0}, NULL, 0) = 1 ([{fd=3, revents=POLLIN}], left {tv_sec=0, tv_nsec=534236288})
write(1, "00:07:24.072724 STP 802.1d, Conf"..., 9900:07:24.072724 STP 802.1d, Config, Flags [none], bridge-id 8000.36:d6:2e:e8:78:5f.8002, length 35
) = 99
ppoll([{fd=3, events=POLLIN}], 1, {tv_sec=1, tv_nsec=0}, NULL, 0) = 0 (Timeout)
ppoll([{fd=3, events=POLLIN}], 1, {tv_sec=1, tv_nsec=0}, NULL, 0) = 1 ([{fd=3, revents=POLLIN}], left {tv_sec=0, tv_nsec=70633344})
```
核心系统调用：

|syscall name|strace输出|解释|
|-----|-----|-----|
|socket|socket(AF_PACKET, SOCK_RAW, 768) = 3|创建socket，第3个参数768正是ETH_P_ALL（0x0003）的10进制数值|
|setsockopt|setsockopt(3, SOL_PACKET, PACKET_RX_RING, 0x7fee51da78, 28) = 0|PACKET_RX_RING分配环形缓冲区|
|mmap|mmap(NULL, 3670016, PROT_READ\|PROT_WRITE, MAP_SHARED, 3, 0) = 0x7fbd04b000|将PACKET_RX_RING缓冲区映射到用户空间|
|setsockopt|setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, {len=1, filter=0x632f10}, 16)|SO_ATTACH_FILTER设置filter|
|ppoll|ppoll([{fd=3, events=POLLIN}],|等待packet|

## socket系统调用
```
socket(PF_PACKET, SOCK_RAW, protocol) //protocol为ETH_P_ALL
    sys_socket/sock_create
        sock = sock_alloc()
        pf->create(net, sock, protocol, kern)
            packet_create
                sk = sk_alloc(net, PF_PACKET, GFP_KERNEL, &packet_proto, kern) //分配struct packet_sock *po
                sock->ops = &packet_ops //将struct proto_ops packet_ops赋值给struct socket *sock的成员ops
                sock_init_data(sock, sk)
                po->prot_hook.func = packet_rcv //将packet_rcv赋值给struct packet_type的成员变量func
                __register_prot_hook(sk)
                    dev_add_pack(&po->prot_hook) //将packet_type添加到链表ptype_all
```
1. sock_alloc分配struct socket *sock
2. 调用PF_PACKET协议的socket创建函数packet_create
3. sk_alloc分配struct packet_sock *po
4. sock->ops = &packet_ops将套接字层操作函数集struct proto_ops packet_ops赋值给struct socket *sock的成员ops，包含packet_setsockopt，packet_recvmsg等函数。
5. 将packet_rcv赋值给struct packet_type的成员变量func
6. 调用dev_add_pack将接收处理链路层报文的处理结构struct packet_type添加到链表ptype_all，当链路收到报文时轮询链表ptype_all，执行packet_rcv往用户空间提交报文
## setsockopt系统调用：参数PACKET_RX_RING分配rx_ring
setsockopt(3, SOL_PACKET, PACKET_RX_RING
```
__sys_setsockopt
    sock->ops->setsockopt
        packet_setsockopt
            packet_set_ring
                order = get_order(req->tp_block_size)
                pg_vec = alloc_pg_vec(req, order)
                    kcalloc
                po->prot_hook.func = (po->rx_ring.pg_vec) ?
                        tpacket_rcv : packet_rcv;
```
1. socket层调用PF_PACKET套接字层参数设置函数packet_setsockopt
2. 根据用户指定的环形缓冲区参数，调用packet_set_ring在内核空间分配环形缓冲区
3. 将接收处理链路层报文的处理结构struct packet_type的func修改为tpacket_rcv，表明由其接收数据链路层报文。
## mmap系统调用
mmap(NULL, 3670016, PROT_READ|PROT_WRITE, MAP_SHARED, 3, 0)
mmap系统调用的内核call trace：
```
[ 85.216463] CPU: 2 PID: 1483 Comm: tcpdump Not tainted 4.19.16 #10
[ 85.218554] Hardware name: linux,dummy-virt (DT)
[ 85.219846] Call trace:
[ 85.221611] dump_backtrace+0x0/0x1a0
[ 85.225277] show_stack+0x14/0x20
[ 85.225758] dump_stack+0x90/0xbc
[ 85.225917] packet_mmap+0x40/0x1a0
[ 85.229538] sock_mmap+0x20/0x30
[ 85.230575] mmap_region+0x3c0/0x580
[ 85.231001] do_mmap+0x34c/0x4e0
[ 85.232663] vm_mmap_pgoff+0xe4/0x110
[ 85.233200] ksys_mmap_pgoff+0xa8/0x240
[ 85.233595] __arm64_sys_mmap+0x74/0x90
[ 85.236261] el0_svc_common+0x60/0x100
[ 85.237293] el0_svc_handler+0x2c/0x80
[ 85.238079] el0_svc+0x8/0xc
```
代码解读：
```
SYSCALL_DEFINE6(mmap, unsigned long
__arm64_sys_mmap
    ksys_mmap_pgoff/vm_mmap_pgoff/do_mmap_pgoff/do_mmap
        addr = get_unmapped_area//在用户虚拟地址空间分配一块空闲区域
        mmap_region
            file->f_op->mmap(file, vma)//调用驱动程序mmap指针packet_mmap
               vm_insert_page(vma, start, page)//建立映射
```
1. 首先get_unmapped_area在用户空间mmap区域获得一块尚未被映射的vm_area_struct对象；
2. 调用套接字文件处理函数packet_mmap将get_unmapped_area分配的用户空间mmap区域一段内存映射到由setsockopt(3, SOL_PACKET, PACKET_RX_RING系统调用分配的物理内存环形缓冲区上，调用vm_insert_page完成。
* 关于vm_insert_page，可参考 [What is the difference between vm_insert_page() and remap_pfn_range()?](https://stackoverflow.com/questions/27468083/what-is-the-difference-between-vm-insert-page-and-remap-pfn-range)

## setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER
```
__sys_setsockopt
    sock_setsockopt(sock, level, optname, optval,
        sk_attach_bpf(ufd, sk)
            __sk_attach_prog(prog, sk)
```
setsockopt系统调用设置BPF过滤器。
关于BPF过滤器，可参考:
* [使用socket BPF](https://www.cnblogs.com/rollenholt/articles/2585517.html)
* [libpcap BSD Packet Filter(BPF)](http://blog.chinaunix.net/uid-9950859-id-98958.html)

## 执行tpacket_rcv
所有的往外发的包和进来的包都会调到https://github.com/torvalds/linux/blob/master/net/packet/af_packet.c 这个文件里面 的tpacket_rcv函数（recvfrom方式调用packet_rcv），其中egress方向（出去的包）会在 dev_queue_xmit_nit里面遍历ptype_all链表进行所有网络协议处理的时候调用到tpacket_rcv；ingress方向（从外面其他机器进来的包会在__netif_receive_skb_core函数里面同样办法遍历 ptype_all进行处理的时候调用到packet_rcv。
### ingress方向
从__netif_receive_skb_core开始解读：
```
__netif_receive_skb_core
    list_for_each_entry_rcu(ptype, &ptype_all, list)
    deliver_skb
        pt_prev->func(skb, skb->dev, pt_prev, orig_dev)
            tpacket_rcv
```
### egress方向
```
dev_hard_start_xmit
    xmit_one(skb, dev, txq, next != NULL)
        dev_queue_xmit_nit
            deliver_skb
                pt_prev->func(skb, skb->dev, pt_prev, orig_dev)
                    tpacket_rcv
```
### tpacket_rcv详解
```
run_filter(skb, sk, snaplen)//运行run_filter，通过BPF过滤中我们设定条件的报文
packet_current_rx_frame //在ring buffer中查找TP_STATUS_KERNEL的frame
skb_copy_bits(skb, 0,  h.raw+macoff, snaplen)//将数据从skb拷贝到kernel ring Buffer中
flush_dcache_page(pgv_to_page(start)//把某页在data cache中的内容同步回内存
sk->sk_data_ready(sk)//调用sk_data_ready，通知睡眠进程
```

# 参考资料
* [tcpdump抓包对性能的影响](https://blog.csdn.net/dog250/article/details/52502623)，作者：csdn博主dog250
* [suricata抓包方式之一 AF_PACKET](https://www.cnblogs.com/Anker/p/6040873.html)
* [PACKET_MMAP实现原理分析](http://blog.chinaunix.net/uid-20357359-id-1963684.html),作者：cudev.blog.chinaunix.net，是内核文档Document/networking/packet_mmap.txt的翻译和注解
* [ linux/Documentation/networking/packet_mmap.txt](https://github.com/torvalds/linux/blob/master/Documentation/networking/packet_mmap.txt)
* [Linux packet mmap](https://sites.google.com/site/packetmmap/home)
