---
layout: post
title:  "图解linux ping"
categories: network
tags: linux ping socket ICMP
author: jgsun
---

* content
{:toc}

# 概述
ping命令采用ICMP协议，是一个用户空间程序，它打开一个SOCK_RAW套接字或者ICMP套接字发送ICMP_ECHO消息，收到ICMP_ECHOREPLY的消息来相应。本文讲述了ping命令的内核实现。






# ping用户空间程序
busybox有ping命令，iputils也有ping命令，本文研究的是iputils的ping命令。
> 关于iputils： The iputils package is set of small useful utilities for Linux networking.
These tools are included in iputils：arping clockdiff ninfod ping rarpd rdisc tftpd tracepath traceroute6
可从项目github主页：https://github.com/iputils/iputils下载源码。

iputils的ping命令的简单流程：
```
create_socket
    socket(family, socktype, protocol) //socket系统调用，打开SOCK_RAW套接字或者ICMP套接字
set_socket_option
ping4_run
    main_loop
        pinger(fset, sock)
            ping4_send_probe
                sendto //send系统调用
        poll(&pset, 1, next) //poll系统调用
        recvmsg(sock->fd, &msg, polling) //recvmsg系统调用
```
# SOCK_RAW套接字实现的ping
ping命令发送端内核实现图，可见发送端发送ICMP_ECHO消息，接收ICMP_ECHOREPLY消息。
![image](/images/posts/network/ping/raw_socket_tx.png)

ping命令接收端的内核实现图，可见接收端接收ICMP_ECHO消息，发送ICMP_ECHOREPLY消息。
![image](/images/posts/network/ping/ping_reply.png)

## icmp相关初始化
```
fs_initcall(inet_init)
    inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) //注册ICMP协议，icmp_rcv函数接收网络层报文
    sock_register(&inet_family_ops) //注册inet套接字
    inet_register_protosw(q) //初始化协议交换哈希链表inetsw[]
    dev_add_pack(&ip_packet_type) //注册L3协议，ip_rcv接收链路层报文
    ping_init //初始化ping_table哈希链表和读写锁，用于icmp套接字
    icmp_init //为每一个cpu创建内核ICMPv4套接字，存储在percpu变量net->ipv4.icmp_sk，用于回ping消息用
        inet_ctl_sock_create  
```
## ping命令发送端内核实现
ping命令发送端将依次执行socket系统调用打开RAW套接字，执行sendto系统调用发送ICMP_ECHO消息，执行poll睡眠等待ICMP_ECHOREPLY消息，执行recvmsg系统调用接收ICMP_ECHOREPLY消息，最终打印出'64 bytes from 192.168.122.1: icmp_seq=1 ttl=64 time=11.1 ms'
### socket系统调用
```
create_socket(&sock4, AF_INET, hints.ai_socktype, IPPROTO_ICMP, hints.ai_family == AF_INET)
    sock->fd = socket(family, SOCK_RAW, protocol)
        sys_socket/sock_create
            sock = sock_alloc()
            pf->create(net, sock, protocol, kern)
                inet_create
                    list_for_each_entry_rcu(answer, &inetsw[sock->type], list)//查找协议交换表inetsw[]
                    sock->ops = answer->ops//将struct proto_ops赋值给socket的变量ops
                    sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern)
                        sk->sk_prot = sk->sk_prot_creator = prot//将struct proto赋值给sock的变量sk_prot
                    sock_init_data(sock, sk)
                    sk->sk_prot->init(sk) //调用raw_init
```
### sendto系统调用
```
ping4_run/main_loop/pinger(fset, sock)/fset->send_probe
    ping4_send_probe/sendto(sock->fd, icp, cc, 0, (struct sockaddr*)&whereto, sizeof(whereto))
        __sys_sendto/sock_sendmsg(sock, &msg)/sock->ops->sendmsg
            inet_sendmsg/sk->sk_prot->sendmsg
                raw_sendmsg
                    ip_route_output_flow
                    ip_append_data
                        __skb_queue_tail(queue, skb) //enqueque to sk_write_queue
                    ip_push_pending_frames
                        skb = __skb_dequeue(queue) //dequeue from sk_write_queue
                        ip_local_out(net, skb->sk, skb)                  
```
### poll系统调用
```
poll(&pset, 1, next)
    socket_file_ops.sock_poll
        inet_sockraw_ops.datagram_poll
            poll_wait(filp, &sock->wq->wait, p)//睡眠等待ICMP_ECHOREPLY消息
```
### 接收ICMP_ECHOREPLY消息
```
el1_irq/gic_handle_irq/__handle_domain_irq/irq_exit
    __do_softirq/net_rx_action
        virtnet_poll/receive_buf
            napi_gro_receive/netif_receive_skb_internal/__netif_receive_skb_one_core
                ip_rcv/ip_rcv_finish/ip_local_deliver/ip_local_deliver_finish
                    raw_local_deliver/raw_rcv/raw_rcv_skb
                        __skb_queue_tail //enqueque to sk_receive_queue
                        sk_data_ready //call sock_def_readable
                            sk_wake_async/kill_fasync //唤醒ping用户空间程序
                    icmp_rcv/ping_rcv 
```
### recvmsg系统调用
ping用户空间程序被唤醒之后，执行recvmsg系统调用：
```
recvmsg(sock->fd, &msg, polling)
    __sys_recvmsg/sock->ops->recvmsg
        inet_recvmsg/sk->sk_prot->recvmsg
            raw_recvmsg
                skb_recv_datagram
                    __skb_try_recv_from_queue //recv from sk_receive_queue
                skb_copy_datagram_msg
                    copy_to_iter //拷贝到用户空间
fset->parse_reply
    ping4_parse_reply
        gather_statistics
            printf(_("%d bytes from %s:"), cc, from) //打印出64 bytes from 192.168.122.1: icmp_seq=2 ttl=64 time=1.09 ms
```
## ping命令接收端内核实现
```
el1_irq/gic_handle_irq/__handle_domain_irq/irq_exit //hardirq
    __do_softirq/net_rx_action //softirq
        virtnet_poll/receive_buf //virtnet driver
            napi_gro_receive/__netif_receive_skb_one_core
                ip_rcv/ip_rcv_finish/ip_local_deliver_finish
                    raw_local_deliver
                    icmp_rcv/icmp_echo/icmp_reply                     
                        ip_route_output_key
                        icmp_push_reply
                            sk = icmp_sk()//获取icmp_init创建的内核icmp套接字
                            ip_append_data
                                __skb_queue_tail(queue, skb) //enqueque to sk_write_queue
                            ip_push_pending_frames
                                skb = __skb_dequeue(queue) //dequeue from sk_write_queue
                                ip_local_out(net, skb->sk, skb)                  
```
# ICMP套接字(ping套接字)实现的ping
Openwall GNU/linux为了安全，在内核实现了ICMP套接字(ping套接字)，这样ping用户空间程序可以打开一个ping套接字来实现setuid-less ping。默认是禁止使用ICMP套接字，可通过设置profs条目'/proc/sys/net/ipv4/ping_group_range'来启用。创建ICMP套接字的时候，将调用ping_init_sock来进行权限检查，只有用户组gid在ping_group_range范围之内才有权限创建。ICMP套接字导出procfs条目：`/proc/net/icmp`和`/proc/net/icmp6`，
> 关于ICMP套接字，可参考：
* [net: ipv4: add IPPROTO_ICMP socket kind](https://lwn.net/Articles/443051/)

ICMP套接字ping命令发送端内核实现图，接收端的内核实现和RAW套接字是一样的。
![image](/images/posts/network/ping/icmp_socket_tx.png)

# 参考资料
* 《精通linux内核网络》
作者：以色列网络专家Rami Rosen，现intel DPDK Tech Leader，个人主页[http://ramirose.wixsite.com/ramirosen](http://ramirose.wixsite.com/ramirosen)
英文名为《Linux Kernel Networking - Implementation and Theory》[点击在线阅读英文原版](http://apprize.info/linux/kernel/)

