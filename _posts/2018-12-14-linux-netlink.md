---
layout: post
title:  "ǳ��linux netlink"
categories: network
tags: linux netlink
author: jgsun
---

* content
{:toc}

# ����
���Ļ���linux 4.20��
netlinkЭ����һ�ֽ��̼�ͨ�ţ�Inter Process Communication,IPC�����ƣ�Ϊ���û��ռ���ں˿ռ��Լ��ں˵�ĳЩ����֮���ṩ��˫��ͨ�ŷ�����
���Ľ�����linux�ں���netlink�ĳ�ʼ����ͨ�Ź��̣���������ͨ��netlink�׽��֡�









# netlink��ʼ��
�ں������׶Σ�netlink��ϵͳ��ʼ����core_initcall(netlink_proto_init)��ʼ��
1. proto_register(&netlink_proto, 0)
ע�⣬����ڶ���������0����ʾ������alloc_slab��˵����û��ר�Ŷ���slab cache��
ע��netlink�����Э��ʵ�����������netlink_proto���ں�����Э��ջ�У��ӵ�proto_list����
��tcp_prot��ͬ��netlink_protoû�ж���Э����ص��շ�����������netlink��ʵ���йأ����������ƣ���ֻ�ṩһ�����õĳ�Աobj_size = sizeof(struct netlink_sock)�ڴ����û��ռ�netlink�׽��ֵ�ʱ���û������׽��ֶ���
`list_add(&prot->node, &proto_list)`
2. ��ʼ������nl_table
ÿ��netlinkЭ��ض�Ӧnl_table�����һ����Ŀ��struct netlink_table���ͣ���һ��32����nl_table��netlink��ϵͳ��ʵ�ֵ�һ���ؼ���ṹ����ʵ��һ��hash���ṹ��ֻҪ��netlink�׽����γɣ��������ں˵Ļ����û��ռ�ģ���Ҫ����netlink_insert��netlink�׽��ֱ����������Ϣһ�����뵽�������ṹ�У��û�̬�׽�����bindϵͳ���õ�ʱ�����netlink_insert����nl_table���ں��׽������ڴ�����ʱ�����netlink_insert����nl_table����Ȼ���ڷ���ʱ��ֻҪ����netlink_lookup���������Ϳ��Կ��ٶ�λҪ���͵�Ŀ���׽��֡�
3. sock_register(&netlink_family_ops)
����׽��ֲ�Э��handler�������û��ռ�netlink�׽��ֵ�ʱ�򽫵���netlink_family_ops�ṩ�ķ���netlink_create��
4. rtnetlink_init
��1������rtnetlink�ں��ں��׽���
rtnetlink�׽���ר����������netlink�׽��֣�����·����Ϣ���ڽ���Ϣ����·��Ϣ������������ϵͳ��Ϣ��
�ڴ����ں��׽���֮�󣬵���netlink_insert(sk, 0)�����׽��ֲ��뵽nl_table���ſ��Խ����û��ռ䷢�͵�netlink��Ϣ��
netlink_kernel_cfg�ṹinput�ص�����rtnetlink_rcv������ֵ��netlink_sock�ĳ�Աnetlink_rcv�����ڽ��մ��û��ռ䷢�͵����ݣ�
netlink_kernel_create�����ں��׽��ֲ����������������ռ����net�ı���rtnl
```
register_pernet_subsys(&rtnetlink_net_ops)
    rtnetlink_net_init
        sk = netlink_kernel_create(net, NETLINK_ROUTE, &cfg)
            __netlink_create(net, sock, cb_mutex, unit, 1)
            netlink_insert(sk, 0) //�����ں��׽��ֲ���nl_table�����׽��ֲſ��Խ����û��ռ䷢�͵�netlink��Ϣ
            nlk_sk(sk)->netlink_rcv = cfg->input //netlink_rcv�����մ��û��ռ�����ں˿ռ䷢�͵�����
            nl_table[unit].bind = cfg->bind
        net->rtnl = sk //�����׽��ָ�ֵ�����������ռ�ı���rtnl
```
��2������rtnl_registerΪnetlink��Ϣע��ص���������rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_newlink, NULL, 0)��Ϣ����rtnl_newlink��ΪRTM_GETLINK��Ϣ��rtnl_dump_ifinfo�ص��������뵽rtnl_msg_handlers����Ӧ����Ŀ�У���NETLINK_ROUTE�ں��׽��ֽ����û��ռ���Ϣ�Ĵ�����rtnetlink_rcv�У���������Ϣ����RTM_GETLINK��rtnl_msg_handlers��ȡ�䴦����rtnl_newlink����rtnl_dump_ifinfo��
```
rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_getlink,
       rtnl_dump_ifinfo, 0);
```
# NETLINK_ROUTE�׽���
iprout2���߼�ip�������NETLINK_ROUTE�׽��������ں�ͨ�ţ���ȡ������ϵͳ��·�ɣ���·����Ϣ��ip -s link ls eth0��ȡeth0����ӿ�ͳ����Ϣ���������
![image](/images/posts/network/netlink/ip_s_link_ls_eth0.png)

> iproute2Դ��git://git.kernel.org/pub/scm/network/iproute2/iproute2.git

���ǽ�����������Ϊ����Χ����ͼ������NETLINK_ROUTE�׽��ִӳ�ʼ��������socket��bind��sendmsg��recvmsg���ں˿ռ�ȫ���̡�
![image](/images/posts/network/netlink/netlink_route.png)

����ͼ���Կ�����NETLINK_ROUTE�׽���ͨ�Ź�������Χ����������չ������nl_table��rtnl_msg_handlers��ͼ�б�ʶ���������̵���š�
��ʼ�������Ѿ������桰netlink��ʼ�������ֽ����ˣ������ظ���
## socketϵͳ����
socketϵͳ���ý������û��ռ�netlink�׽��֣���domain����AF_NETLINK����protocol��NETLINK_ROUTE��
```
rtnl_open(&rth, 0)
    rtnl_open_byproto(rth, subscriptions, NETLINK_ROUTE)
        rth->fd = socket(AF_NETLINK, SOCK_RAW | SOCK_CLOEXEC, protocol)
            __sys_socket/__sock_create/pf->create(net, sock, protocol, kern)
            netlink_create
                cb_mutex = nl_table[protocol].cb_mutex
                bind = nl_table[protocol].bind
                unbind = nl_table[protocol].unbind
                __netlink_create(net, sock, cb_mutex, protocol, kern)
                    sock->ops = &netlink_ops
                    sk_alloc(net, PF_NETLINK, GFP_KERNEL, &netlink_proto, kern) //����struct netlink_sock�׽��֣��̳�ͨ���׽���struct sock��
                    sock_init_data(sock, sk)
                        sk->sk_rcvbuf	=	sysctl_rmem_default //�׽��ֳ�ʼ��ʱ��/proc/sys/net/core/rmem_defaultΪ���ʼ���ջ��棬������������setsockopt���µ���
                        sk->sk_sndbuf	=	sysctl_wmem_default
                        sk->sk_data_ready	=	sock_def_readable //���׽��ֵ�sk_receive_queue�յ�����sock_def_readable������recvmsg�������ա�
        setsockopt(rth->fd, SOL_SOCKET, SO_SNDBUF //�����׽��ַ��ͻ����С
        setsockopt(rth->fd, SOL_SOCKET, SO_RCVBUF //�����׽��ֽ��ջ����С
        getsockname(rth->fd, (struct sockaddr *)&rth->local
                    init_waitqueue_head(&nlk->wait)
```

## bindϵͳ����
local�Ǳ���socket��ַ��struct sockaddr_nl�ṹ��,��nl_pid����Ϊ0��ͨ��Ӧ��Ϊ��ǰ����id��������Ϊ0Ҳû�й�ϵ������bindϵͳ���ô�socket��ops��netlink_ops����socketϵͳ���ô���socket��ʱ��ֵ������ָ��netlink_bind��netlink_bind������netlink_autobind����Ӧnetlink_sock�ĳ�Աportid��bound������Ϊ��ǰ�̵߳�thread group id(tgid)��������__netlink_insert�����û�̬�׽���sock�������Ϣportid���뵽nl_table[sk->sk_protocol]�У��������û�̬�׽��־Ϳ��Խ��������ں˵�netlink��Ϣ��
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

## sendmsgϵͳ����
```
do_iplink
    iplink_modify(RTM_NEWLINK,
        iplink_parse
        rtnl_talk
            sendmsg(rtnl->fd, &msg, 0)
```
ip link��ϵͳ������sendmsg����Ӧ�ں˺�����__sys_sendmsg
```
__sys_sendmsg(fd, msg, flags, true)
    ___sys_sendmsg(sock, msg, &msg_sys, flags, NULL, 0)
        sock_sendmsg(sock, msg_sys)
            sock_sendmsg_nosec(sock, msg)
                sock->ops->sendmsg(sock, msg, msg_data_left(msg))
                    netlink_sendmsg
                        netlink_unicast
                            sk = netlink_getsockbyportid(ssk, portid)
                                netlink_lookup(sock_net(ssk), ssk->sk_protocol, portid)//��nl_table���Ұ󶨵��ں�netlink�׽���
                            netlink_unicast_kernel(sk, skb, ssk)
                                nlk->netlink_rcv(skb) 
                                    rtnetlink_rcv/netlink_rcv_skb/rtnetlink_rcv_msg//ִ���ں�netlink�׽��ֵĽ��պ���rtnetlink_rcv�����û���Ϣ��
```
## recvmsgϵͳ����
```
recvmsg
    __sys_recvmmsg
        ___sys_recvmsg
            sock->ops->recvmsg(sock
            netlink_recvmsg
                skb_recv_datagram
                    __skb_wait_for_more_packets
                        prepare_to_wait_exclusive(sk_sleep(sk), &wait, TASK_INTERRUPTIBLE)/schedule_timeout(*timeo_p)
                    __skb_try_recv_datagram
                        __skb_try_recv_from_queue //��socket buffer (&sk->sk_receive_queue)�??��
```
�ں˷���netlink��Ϣ���û��Ĺ��̣�����skb_buffд���û�netlink�׽��ֵ�sk_receive_queue�������rtnetlink_rcv_msg��ʼ������
```
rtnetlink_rcv_msg
    rtnl_get_link(family, type)//��rtnl_msg_handlers���в�ѯ��rtnl_registerע���type��Ϣ���͵Ļص��������˴���RTM_GETLINK��Ϣ�Ļص�����rtnl_dump_ifinfo��
        netlink_dump_start(rtnl, skb, nlh, &c)
            netlink_lookup(sock_net(ssk), ssk->sk_protocol, NETLINK_CB(skb).portid) //��nl_table���Ұ󶨵��û�̬netlink�׽���
            netlink_dump(sk) //�˴��Ĳ����Ѿ����û�̬�׽���
                skb = alloc_skb(alloc_size  //����skb������
                skb_reserve(skb, skb_tailroom(skb) - alloc_size)
                netlink_skb_set_owner_r(skb, sk) //���û�̬sock��ֵ��skb->sk
                nlk->dump_done_errno = cb->dump(skb, cb) //���ｫִ��rtnl_dump_ifinfo�ص�����
                __netlink_sendskb(sk, skb)
                    netlink_deliver_tap(sock_net(sk), skb)
                    skb_queue_tail(&sk->sk_receive_queue, skb) //��skb���뵽�û�̬sock��sk_receive_queue����
                    sk->sk_data_ready(sk) //����sk_data_ready����netlink_recvmsg���н���
                        wake_up_interruptible_sync_poll(&wq->wait
```
# ͨ��NETLINK_GENERIC�׽���
netlinkЭ��������32����MAX_LINKS����Ϊ֧�ָ����Э��أ�������ͨ��netlink��NETLINK_GENERIC��ͨ��netlink��netlinkЭ��Ϊ������ʹ����API������netlink��·��������ͨ��netlinkЭ���ѱ������ڶ���ں���ϵͳ����ACPI��ϵͳ������ͳ����Ϣ���룬�����¼���wireless������ϵͳ�ȡ�
��ȡͨ��netlink�������ز�������`genl ctrl getname nlctrl`������ǣ�
![image](/images/posts/network/netlink/genl_ctrl_getname_nlctrl.png)

���ǽ����������Ϊ����Χ����ͼ������ͨ��NETLINK_GENERIC�׽��ֵ�ͨ�Ź��̡�
����ͼ���Կ�����ͨ��netlink�׽���ͨ�Ź���Ҳ��Χ����������չ������nl_table��genl_fam_idr��ͼ�б�ʶ���������̵���š�
![image](/images/posts/network/netlink/netlink_generic.png)
## ͨ��netlink�׽��ֳ�ʼ��
ͨ��netlinkЭ���ʹ��netlinkЭ���API�����ʼ��ͬ����Ҫcore_initcall(netlink_proto_init)�������������еĳ�ʼ������subsys_initcall(genl_init)��genl_init����Ҫ�Ĺ����Ǵ���ͨ��NETLINK_GENERIC�ں��׽��֣��˴������û��ռ���Ϣ��input������genl_rcv�����⻹ע����ͨ��netlink�׽��ֿ�������genl_ctrl���˿�������genl_ctrl��ͨ��netlinkЭ����Ƶĵ�һ���û�������һ����Ҫ���ã���������ͨ���׽��ִص��û��ռ�Ӧ�ó���Ҫʹ�ô˿�����������ȡidr���id�������ں�ͨ�š�iproute2��genl�������ͨ���˿�������genl_ctrl����ѯ�ں�����ע���ͨ��netlink�صĸ��ֲ�������id����ͷ���ȣ�����������ȡ�������Ҫʹ��ͨ��netlinkЭ�����ϵͳֻ���ȶ���genl_family����Ȼ�����genl_register_family��idr��genl_fam_idr����ע�ἴ�ɡ�
```
subsys_initcall(genl_init)
    genl_register_family(&genl_ctrl) //ע��ͨ��netlink�׽��ֿ������أ�genl_family��
        idr_alloc(&genl_fam_idr, family //������������ӵ�idr��genl_fam_idr
        genl_ctrl_event(CTRL_CMD_NEWFAMILY, family, NULL, 0)
    register_pernet_subsys(&genl_pernet_ops)
        net->genl_sock = netlink_kernel_create(net, NETLINK_GENERIC, &cfg) //����ͨ��netlink�ں��׽���
```
ͨ��netlink����ʹ��idr���ƽ���Э���genl_family�Ĺ���
1. �� genl_register_family��genl_family��ӵ�idr��genl_fam_idr������ȡid��
2. �û��ռ����genl_family����������ѯ��ȡ����genl_fam_idr��id������Ϊ���������ں�ͨ��netlink�׽��֣���������ͨ��netlinkЭ��������أ�������id��Ҫ���̶�ΪGENL_ID_CTRL(0x10)���ⲿ�����ӿ��Բο��ں˹���getdelay�Ĵ���[tools/accouting/getdelay.c](https://github.com/torvalds/linux/blob/master/tools/accounting/getdelays.c)
3. ͨ��netlink�ں��׽���input����genl_rcv���ݴ�id��ѯidr��genl_fam_idr��ȡ��Ӧ��genl_family���Ӷ���ȡ������������genl_ops��
����idr���ƣ��ο�[idr����](https://blog.csdn.net/coldsnow33/article/details/13503183)
## ͨ��netlink�׽����շ���Ϣ
```
genl_rcv
    genl_rcv_msg
        family = genl_family_find_byid(nlh->nlmsg_type)
            idr_find(&genl_fam_idr, id)//����ͨ��netlink�׽��ִ�id��idr��genl_fam_idr���һ�ȡgenl_family
        genl_family_rcv_msg(family, skb, nlh, extack)
            ops = genl_get_cmd(hdr->cmd, family)
                ops->doit(skb, &info)
                ctrl_getfamily
                    msg = ctrl_build_family_msg(res, info->snd_portid
                        skb = nlmsg_new(NLMSG_DEFAULT_SIZE, GFP_KERNEL)
                            alloc_skb(nlmsg_total_size(payload), flags)
                    genlmsg_reply(msg, info)
                        genlmsg_unicast(genl_info_net(info), skb, info->snd_portid)
                            nlmsg_unicast(net->genl_sock, skb, portid)
                                netlink_unicast(sk, skb, portid, MSG_DONTWAIT)
                                    netlink_getsockbyportid(ssk, portid)
                                        sock = netlink_lookup(sock_net(ssk), ssk->sk_protocol, portid)//��nl_table���Ұ󶨵��û�̬netlink�׽���                                    netlink_attachskb(sk, skb, &timeo, ssk) //��sock��sk_buff����һ��
                                    netlink_sendskb(sk, skb)
                                        skb_queue_tail(&sk->sk_receive_queue, skb)
```

# ����netlinkЭ����׽���
## NETLINK_NETFILTER socket
��nfnetlink_net_init������
```
<net/netfilter/nfnetlink.c>
module_init(nfnetlink_init)
    register_pernet_subsys(&nfnetlink_net_ops) -> nfnetlink_net_init
```
## NETLINK_KOBJECT_UEVENT socket
�����豸������uevent_net_init������
```
static int uevent_net_init(struct net *net)
{
 struct uevent_sock *ue_sk;
 struct netlink_kernel_cfg cfg = {
  .groups	= 1,
  .input = uevent_net_rcv,
  .flags	= NL_CFG_F_NONROOT_RECV
 };

 ue_sk = kzalloc(sizeof(*ue_sk), GFP_KERNEL);
 if (!ue_sk)
  return -ENOMEM;

 ue_sk->sk = netlink_kernel_create(net, NETLINK_KOBJECT_UEVENT, &cfg);
```
## NETLINK_SOCK_DIAG socket �׽��ּ����׽���
��sock_diag_init������

# netlink�û��ռ����
## netlink�׽��ֿ�libnl
netlink�׽��ֿ�libnl�ṩAPI���ʻ���netlinkЭ����ں˽ӿڣ���ٷ���վ��https://www.infradead.org/~tgr/libnl/�������Ŀ�libnl֮�⣬����ͨ��netlink��libnl-genl��·��ѡ���libnl-route��netfilter��libnl-nf�ȡ�


## netlink��Ϣ��ͷ�����ݽṹ
�ο�����ͨlinux�ں����硷������û�Ӧ�ó�����ں˴��������⡣�������ƣ�

## ͨ��netlink��ͷ�����ݽṹ
�ο�����ͨlinux�ں����硷������û�Ӧ�ó�����ں˴��������⡣�������ƣ�

# �ο�����
* ����ͨlinux�ں����硷��Ӣ������Linux Kernel Networking - Implementation and Theory��
���ߣ���ɫ������ר��Rami Rosen����intel DPDK Tech Leader��������ҳhttp://ramirose.wixsite.com/ramirosen
* [linux��netlink����](https://blog.csdn.net/dog250/article/details/5303430)
���ߣ�csdn��������14��dog250

