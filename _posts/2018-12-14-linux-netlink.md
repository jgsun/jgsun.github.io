---
layout: post
title:  "图解linux netlink"
categories: network
tags: linux netlink
author: jgsun
---

* content
{:toc}

# 概述
netlink协议是一种进程间通信（Inter Process Communication,IPC）机制，为的用户空间和内核空间以及内核的某些部分之间提供了双向通信方法。
本文围绕两张图介绍iprout2命令"ip -s link ls eth0"和"genl ctrl getname nlctrl"的内核实现，来解析netlink套接字协议簇和通用netlink协议，包括初始化、套接字系统调用socket，bind，sendmsg和recvmsg、核心数据结构等。通过分析netlink的实现机制，达到能在生产实践中正确使用netlink的目的。本文基于linux 4.20。







> 关于iprout2：iprout2工具集ip命令采用netlink套接字来与内核通信，访问和设置网络子系统的路由，链路等信息，可参考[https://wiki.linuxfoundation.org/networking/iproute2](https://wiki.linuxfoundation.org/networking/iproute2)获取源码和更多信息。
# netlink协议簇
netlink套接字支持最大32个协议簇，iprout2采用NETLINK_ROUTE协议簇和内核通信，其中命令："ip -s link ls eth0"获取eth0网络接口统计信息，其输出：
![image](/images/posts/network/netlink/ip_s_link_ls_eth0.png)

下面以这条命令为例，围绕下图来讲述NETLINK_ROUTE套接字从初始化，创建socket，bind，sendmsg到recvmsg的内核空间全过程。
![image](/images/posts/network/netlink/netlink_route.png)

可以看出，NETLINK_ROUTE套接字通信过程总体围绕两个数组展开，即nl_table和rtnl_msg_handlers，图中黑色数字标识出整个过程的序号。
> 图中红色数字和黑色虚线标识了socket设计的思路：①初始化调用sock_register注册协议簇套接字操作函数块，提供创建套接字的回调函数netlink_create；②socket系统调用创建套接字，将套接字层对应系统调用的套接字操作函数块struct proto_ops netlink_ops赋值给套接字socket。

## netlink初始化
内核启动阶段，netlink子系统初始化从core_initcall(netlink_proto_init)开始：
1. proto_register(&netlink_proto, 0)
注意，这里第二个参数是0，表示不进行alloc_slab，说明其没有专门定义slab cache。
注册netlink协议实例函数块变量struct proto netlink_proto到内核网络协议栈中：加到proto_list链表（cat cat /proc/net/protocols可查看系统中注册的所有网络协议实例）；struct proto网络协议实例是socket层和传输层之间的接口，包含协议收发数据包的函数指针。netlink没有传输层功能，所以其函数块netlink_proto没有函数指针，只有一个变量obj_size = sizeof(struct netlink_sock)用于在创建用户空间netlink套接字的时候指定分配的套接字对象大小。
`list_add(&prot->node, &proto_list)`
2. 初始化数组nl_table
每个netlink协议簇对应nl_table数组的一个条目（struct netlink_table类型），一共32个。nl_table是netlink子系统的实现的一个关键表结构，其实是一个hash链结构，只要创建netlink套接字，不管是内核的还是用户空间的，都要调用netlink_insert将netlink套接字本身和它的信息一并插入到这个链表结构中（用户态套接字在bind系统调用的时候调用netlink_insert插入nl_table；内核套接字是在创建的时候调用netlink_insert插入nl_table），然后在发送时，只要调用netlink_lookup遍历这个表就可以快速定位要发送的目标套接字。
3. sock_register(&netlink_family_ops)
注册netlink协议簇套接字操作函数块，创建用户空间netlink套接字的时候将调用netlink_family_ops提供的方法netlink_create。
4. rtnetlink_init
(1) 创建rtnetlink内核内核套接字
rtnetlink套接字是NETLINK_ROUTE协议簇，专用于联网的netlink套接字，用于路由消息、邻接消息、链路消息和其他网络子系统消息。netlink_kernel_create在创建内核套接字时，调用netlink_insert(sk, 0)将此套接字插入到nl_table，才可以接收用户空间发送的netlink消息。
netlink_kernel_cfg结构的input回调函数rtnetlink_rcv，将赋值给netlink_sock的成员netlink_rcv，用于接收从用户空间发送的消息；netlink_kernel_create创建的内核套接字并保存在网络命名空间对象net的变量rtnl。

rtnetlink的netlink_kernel_cfg结构体：
```
struct netlink_kernel_cfg cfg = {
	.groups		= RTNLGRP_MAX,
	.input		= rtnetlink_rcv,
	.cb_mutex	= &rtnl_mutex,
	.flags		= NL_CFG_F_NONROOT_RECV,
	.bind		= rtnetlink_bind,
};

	sk = netlink_kernel_create(net, NETLINK_ROUTE, &cfg);
```
register_pernet_subsys(&rtnetlink_net_ops)将调用rtnetlink_net_init：
```
register_pernet_subsys(&rtnetlink_net_ops)
    rtnetlink_net_init
        sk = netlink_kernel_create(net, NETLINK_ROUTE, &cfg)
            __netlink_create(net, sock, cb_mutex, unit, 1)
            netlink_insert(sk, 0) //将此内核套接字插入nl_table表，此套接字才可以接收用户空间发送的netlink消息
            nlk_sk(sk)->netlink_rcv = cfg->input //netlink_rcv将接收从用户空间或者内核空间发送的数据
            nl_table[unit].bind = cfg->bind
        net->rtnl = sk //将此套接字赋值给网络命名空间的变量rtnl
```
(2) 调用rtnl_register为NETLINK_ROUTE套接字消息注册回调函数，如rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_newlink, NULL, 0)消息，将rtnl_newlink作为RTM_GETLINK消息的rtnl_dump_ifinfo回调函数加入到rtnl_msg_handlers表相应的条目中；在NETLINK_ROUTE内核套接字接收用户空间消息的处理函数rtnetlink_rcv中，将根据消息类型RTM_GETLINK从rtnl_msg_handlers获取其处理函数rtnl_newlink或者rtnl_dump_ifinfo。
```
rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_getlink,
       rtnl_dump_ifinfo, 0);
```

## socket系统调用
socket系统调用将创建用户空间netlink套接字，其domain参数AF_NETLINK，是protocol是NETLINK_ROUTE。
```
rtnl_open/rtnl_open_byproto/socket
    __sys_socket/__sock_create/pf->create(net, sock, protocol, kern)
    netlink_create
        cb_mutex = nl_table[protocol].cb_mutex
        bind = nl_table[protocol].bind
        unbind = nl_table[protocol].unbind
        __netlink_create(net, sock, cb_mutex, protocol, kern)
            sock->ops = &netlink_ops //赋值套接字层对应系统调用的套接字操作函数块struct proto_ops
            sk_alloc(net,//分配struct netlink_sock套接字（继承通用套接字struct sock）
            sock_init_data(sock, sk)
                sk->sk_rcvbuf = sysctl_rmem_default//初始化接收/发送缓存为rmem_default/wmem_default
                sk->sk_sndbuf = sysctl_wmem_default//可用setsockopt调整
                sk->sk_data_ready = sock_def_readable//当收到包，sk_data_ready用于唤醒recvmsg接收函数
setsockopt(rth->fd, SOL_SOCKET, SO_SNDBUF //设置套接字发送缓存大小
setsockopt(rth->fd, SOL_SOCKET, SO_RCVBUF //设置套接字接收缓存大小
getsockname(rth->fd, (struct sockaddr *)&rth->local
            init_waitqueue_head(&nlk->wait)
```
socket系统调用初始化过程中sock_register(&netlink_family_ops)注册的netlink协议簇的函数指针netlink_create。netlink_create函数读取nl_table对应protocol的cb_mutex，bind和unbind（创建内核套接字的时候写入nl_table）赋值给netlink_sock；接着调用__netlink_create，分配struct netlink_sock套接字，初始化套接字收发缓存等，将套接字层对应系统调用的套接字操作函数块struct proto_ops netlink_ops赋值给套接字socket，这样套接字其他系统调用将执行此套接字操作函数块的函数（如bind系统调用执行netlink_bind，sendmsg系统调用执行netlink_sendmsg等）。

## bind系统调用
local是本地socket地址（struct sockaddr_nl结构）,其nl_pid设置为0（通常应该为当前进程id），设置为0也没有关系，后面bind系统调用此socket的ops（netlink_ops，在socket系统调用创建socket的时候赋值）函数指针netlink_bind，netlink_bind将调用netlink_autobind将对应netlink_sock的成员portid和bound都设置为当前线程的thread group id(tgid)；并调用__netlink_insert将此用户态套接字sock和相关信息portid插入到nl_table[sk->sk_protocol]中，这样此用户态套接字就可以接收来自内核的netlink消息。
```
bind(rth->fd, (struct sockaddr *)&rth->local
    netlink_bind
        netlink_autobind(sock)
            s32 portid = task_tgid_vnr(current)
            err = netlink_insert(sk, portid)
                nlk_sk(sk)->portid = portid
                __netlink_insert(table, sk)
                nlk_sk(sk)->bound = portid
```

## sendmsg系统调用
```
do_iplink
    iplink_modify(RTM_NEWLINK,
        iplink_parse
        rtnl_talk
            sendmsg(rtnl->fd, &msg, 0)
```
ip link的系统调用是sendmsg，对应内核函数是__sys_sendmsg
```
__sys_sendmsg/___sys_sendmsg/sock_sendmsg/sock->ops->sendmsg(sock, msg, msg_data_left(msg))
    netlink_sendmsg
        netlink_unicast
            sk = netlink_getsockbyportid(ssk, portid)
                netlink_lookup(sock_net(ssk), ssk->sk_protocol, portid)//在nl_table查找绑定的内核netlink套接字
            netlink_unicast_kernel(sk, skb, ssk)
                nlk->netlink_rcv(skb)
                netlink_rcv/netlink_rcv_skb/rtnetlink_rcv_msg//rtnetlink_rcv接收用户netlink消息
```
## recvmsg系统调用
```
recvmsg/__sys_recvmmsg/___sys_recvmsg/sock->ops->recvmsg(sock
    netlink_recvmsg
        skb_recv_datagram
            __skb_wait_for_more_packets
                prepare_to_wait_exclusive(sk_sleep(sk), &wait, TASK_INTERRUPTIBLE)/schedule_timeout(*timeo_p)
            __skb_try_recv_datagram
                __skb_try_recv_from_queue //从socket buffer (&sk->sk_receive_queue)??包
```
内核发送netlink消息给用户的过程，即将skb_buff写入用户netlink套接字的sk_receive_queue。下面从rtnetlink_rcv_msg开始分析：
```
rtnetlink_rcv_msg
    rtnl_get_link(family, type)//在rtnl_msg_handlers表中查询由rtnl_register注册的type消息类型的回调函数（此处是RTM_GETLINK消息的回调函数rtnl_dump_ifinfo）
        netlink_dump_start(rtnl, skb, nlh, &c)
            netlink_lookup(sock_net(ssk), ssk->sk_protocol, NETLINK_CB(skb).portid) //在nl_table查找绑定的用户态netlink套接字
            netlink_dump(sk) //此处的参数已经是用户态套接字
                skb = alloc_skb(alloc_size  //分配skb缓冲区
                skb_reserve(skb, skb_tailroom(skb) - alloc_size)
                netlink_skb_set_owner_r(skb, sk) //将用户态sock赋值给skb->sk
                nlk->dump_done_errno = cb->dump(skb, cb) //这里将执行rtnl_dump_ifinfo回调函数
                __netlink_sendskb(sk, skb)
                    netlink_deliver_tap(sock_net(sk), skb)
                    skb_queue_tail(&sk->sk_receive_queue, skb) //将skb加入到用户态sock的sk_receive_queue队列
                    sk->sk_data_ready(sk) //调用sk_data_ready唤醒netlink_recvmsg进行接收
                        wake_up_interruptible_sync_poll(&wq->wait
```
# 通用netlink协议
netlink协议簇数最大32个（MAX_LINKS），为支持更多的协议簇，开发了通用netlink簇NETLINK_GENERIC。通用netlink以netlink协议为基础，使用其API，就像netlink多路复用器。通用netlink协议已被用于众多的内核子系统，如ACPI子系统，任务统计信息代码，过热事件，wireless无线子系统等。
获取通用netlink控制器簇参数命令`genl ctrl getname nlctrl`的输出是：
![image](/images/posts/network/netlink/genl_ctrl_getname_nlctrl.png)

下面以这个命令为例，围绕下图来讲述通用NETLINK_GENERIC套接字的通信过程。
可以看出，通用netlink套接字通信过程也是围绕两个数组展开，即nl_table和genl_fam_idr，图中标识出整个过程的序号。
![image](/images/posts/network/netlink/netlink_generic.png)

## 初始化
通用netlink协议簇使用netlink协议的API，其初始化同样需要core_initcall(netlink_proto_init)，还包含其特有的初始化部分subsys_initcall(genl_init)。genl_init最重要的工作是创建通用NETLINK_GENERIC内核套接字，此处接收用户空间消息的input函数是genl_rcv；此外还注册了通用netlink套接字控制器簇genl_ctrl，此控制器簇genl_ctrl是通用netlink协议机制的第一个用户，其有一个重要作用，就是其他通用套接字簇的用户空间应用程序要使用此控制器簇来获取idr表的id才能与内核通信，所以其id需要被固定为GENL_ID_CTRL(0x10)。iproute2的genl命令就是通过此控制器簇genl_ctrl来查询内核所有注册的通用netlink簇的各种参数，如id，报头长度，最大属性数等。其他需要使用通用netlink协议的子系统只需先定义genl_family对象，然后调用genl_register_family向idr表genl_fam_idr进行注册即可。
```
subsys_initcall(genl_init)
    genl_register_family(&genl_ctrl) //注册通用netlink套接字控制器簇（genl_family）
        idr_alloc(&genl_fam_idr, family //将控制器簇添加到idr表genl_fam_idr
        genl_ctrl_event(CTRL_CMD_NEWFAMILY, family, NULL, 0)
    register_pernet_subsys(&genl_pernet_ops)
        net->genl_sock = netlink_kernel_create(net, NETLINK_GENERIC, &cfg) //创建通用netlink内核套接字
```
通用netlink机制使用idr机制进行协议簇genl_family的管理：
1. 在 genl_register_family将genl_family添加到idr表genl_fam_idr，并获取id；
2. 用户空间根据genl_family的名字来查询获取其在genl_fam_idr表注册的id（这里用了“初始化部分”提到的通用netlink协议控制器簇），将id作为消息的一部分发给内核通用netlink套接字。内核工具getdelay（[tools/accouting/getdelay.c](https://github.com/torvalds/linux/blob/master/tools/accounting/getdelays.c)）的get_family_id函数给出了详细用法。
3. 通用netlink内核套接字input函数genl_rcv将据此id查询idr表genl_fam_idr获取对应的genl_family，从而获取后续操作函数集genl_ops。
## 收发消息
从通用netlink簇接收函数genl_rcv开始解析：
```
genl_rcv/genl_rcv_msg
    family = genl_family_find_byid(nlh->nlmsg_type)
        idr_find(&genl_fam_idr, id)//根据通用netlink套接字簇id从idr表genl_fam_idr查找获取genl_family
    genl_family_rcv_msg(family, skb, nlh, extack)/
        ctrl_getfamily
            msg = ctrl_build_family_msg(res, info->snd_portid
                skb = nlmsg_new(NLMSG_DEFAULT_SIZE, GFP_KERNEL)
                    alloc_skb(nlmsg_total_size(payload), flags)
            genlmsg_reply(msg, info)
                genlmsg_unicast(genl_info_net(info), skb, info->snd_portid)/netlink_unicast
                    netlink_getsockbyportid(ssk, portid)
                        netlink_lookup(sock_net(ssk), ssk->sk_protocol, portid)//在nl_table查找用户态套接字
                    netlink_attachskb(sk, skb, &timeo, ssk)//将sock和sk_buff绑定在一起
                    netlink_sendskb(sk, skb)
                        skb_queue_tail(&sk->sk_receive_queue, skb)
```
# netlink用户空间程序
开发使用netlink套接字用户空间应用程序时，可以使用libnl API；netlink必须采用特定的格式，开头是netlink消息报头（struct nlmsghdr），通用netlink还有一个通用netlink消息报头（struct genlmsghdr）。
## netlink套接字库libnl
netlink套接字库libnl提供API访问基于netlink协议的内核接口，其官方网站是[https://www.infradead.org/~tgr/libnl/](https://www.infradead.org/~tgr/libnl/)。除核心库libnl之外，还有通用netlink簇libnl-genl，路由选择簇libnl-route及netfilter簇libnl-nf等。
![image](/images/posts/network/netlink/libnl.png)

## netlink消息报头和数据结构
![image](/images/posts/network/netlink/nlmsghdr.jpg)

参考《精通linux内核网络》及结合用户应用程序和内核代码。
## 通用netlink报头和数据结构
![image](/images/posts/network/netlink/genlmsg.jpg)

参考《精通linux内核网络》及结合用户应用程序和内核代码。
# 总结
1. 用户空间使用netlink套接字和内核通信，和传统的套接字是一样首先使用socket系统调用要创建用户空间套接字，不同的是内核也要创建对应的内核套接字，两者通过nl_table链表进行绑定；创建内核套接字时，要定义接收用户空间netlink消息的input函数，如NETLINK_ROUTE簇的input函数就是rtnetlink_rcv。
nl_table是netlink机制的核心数据结构，围绕此结构的内核活动有：
(1) 用户空间应用程序使用socket系统调用创建套接字，然后在bind系统调用时，内核netlink_bind函数将调用netlink_insert(sk, portid)将此用户态套接字和应用程序的进程pid插入nl_table，这里参数portid就是进程pid；
(2) 创建内核套接字时，调用netlink_insert(sk, 0)将此用户态套接字插入nl_table（因为是内核套接字，这里portid是0）；
(3) 用户空间向内核发送netlink消息时，调用netlink_lookup函数，根据协议簇和portid在nl_table快速查找对应的内核套接字对象；
(4) 当内核空间向用户空间发送netlink消息时，调用调用netlink_lookup函数，根据协议簇和portid在nl_table快速查找对应的用户套接字对象.
2. 使用NETLINK_ROUTE套接字初始化的时候调用rtnl_register为NETLINK_ROUTE套接字消息注册回调函数：将处理消息的回调函数加入到rtnl_msg_handlers表相应的条目中；接收到消息之后将根据消息类型从rtnl_msg_handlers获取其处理函数。
3. netlink协议簇数最大32个（MAX_LINKS等于32），[include/uapi/linux/netlink.h
](https://github.com/torvalds/linux/blob/master/include/uapi/linux/netlink.h)定义了内核支持的协议簇，如果我们想定义新的协议簇，需要先在此文件中定义，例如`#define NETLINK_TEST	25`，然后定义接收处理消息的函数，并创建对应的内核套接字。但是，我们一般不这样用，因为内核开发了通用netlink簇NETLINK_GENERIC来进行netlink协议簇的扩展，支持多达1023个子协议簇（#define GENL_MAX_ID	1023），足够我们使用了。
4. 在生产实践中，在内核中使用通用netlink的步骤是：先创建一个genl_family对象（包含genl_ops对象，处理通用netlink消息的操作函数），然后调用genl_register_family向内核注册。现有内核中已经有很多使用通用netlink的子系统，如taskstats子系统：内核代码是[kernel/taskstats.c](https://github.com/torvalds/linux/blob/master/kernel/taskstats.c)，内核工具集提供了其对应的用户空间应用程序[tools/accouting/getdelay.c](https://github.com/torvalds/linux/blob/master/tools/accounting/getdelays.c)，这为我们提供了极好的参考示例。


# 参考资料
* 《精通linux内核网络》
作者：以色列网络专家Rami Rosen，现intel DPDK Tech Leader，个人主页[http://ramirose.wixsite.com/ramirosen](http://ramirose.wixsite.com/ramirosen)
英文名为《Linux Kernel Networking - Implementation and Theory》[点击在线阅读英文原版](http://apprize.info/linux/kernel/)
* [linux的netlink机制](https://blog.csdn.net/dog250/article/details/5303430)
作者：csdn博客排名14的dog250

