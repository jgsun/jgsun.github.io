---
layout: post
title:  "Ç³Îölinux netlink"
categories: network
tags: linux netlink
author: jgsun
---

* content
{:toc}

# ¸ÅÊö
±¾ÎÄ»ùÓÚlinux 4.20¡£
netlinkĞ­ÒéÊÇÒ»ÖÖ½ø³Ì¼äÍ¨ĞÅ£¨Inter Process Communication,IPC£©»úÖÆ£¬ÎªµÄÓÃ»§¿Õ¼äºÍÄÚºË¿Õ¼äÒÔ¼°ÄÚºËµÄÄ³Ğ©²¿·ÖÖ®¼äÌá¹©ÁËË«ÏòÍ¨ĞÅ·½·¨¡£
±¾ÎÄ½²ÊöÁËlinuxÄÚºËÖĞnetlinkµÄ³õÊ¼»¯¼°Í¨ĞÅ¹ı³Ì£¬»¹½éÉÜÁËÍ¨ÓÃnetlinkÌ×½Ó×Ö¡£









# netlink³õÊ¼»¯
ÄÚºËÆô¶¯½×¶Î£¬netlink×ÓÏµÍ³³õÊ¼»¯´Ócore_initcall(netlink_proto_init)¿ªÊ¼¡£
1. proto_register(&netlink_proto, 0)
×¢Òâ£¬ÕâÀïµÚ¶ş¸ö²ÎÊıÊÇ0£¬±íÊ¾²»½øĞĞalloc_slab£¬ËµÃ÷ÆäÃ»ÓĞ×¨ÃÅ¶¨Òåslab cache¡£
×¢²ánetlink´«Êä²ãĞ­ÒéÊµÀıº¯Êı¿é±äÁ¿netlink_protoµ½ÄÚºËÍøÂçĞ­ÒéÕ»ÖĞ£º¼Óµ½proto_listÁ´±í£»
ºÍtcp_prot²»Í¬£¬netlink_protoÃ»ÓĞ¶¨ÒåĞ­ÒéÏà¹ØµÄÊÕ·¢º¯Êı£¬ÕâÓënetlinkµÄÊµÏÖÓĞ¹Ø£¨£¿£¿´ıÍêÉÆ£©£¬Ö»Ìá¹©Ò»¸öÓĞÓÃµÄ³ÉÔ±obj_size = sizeof(struct netlink_sock)ÔÚ´´½¨ÓÃ»§¿Õ¼änetlinkÌ×½Ó×ÖµÄÊ±ºòÓÃ»§·ÖÅäÌ×½Ó×Ö¶ÔÏó¡£
`list_add(&prot->node, &proto_list)`
2. ³õÊ¼»¯Êı×énl_table
Ã¿¸önetlinkĞ­Òé´Ø¶ÔÓ¦nl_tableÊı×éµÄÒ»¸öÌõÄ¿£¨struct netlink_tableÀàĞÍ£©£¬Ò»¹²32¸ö¡£nl_tableÊÇnetlink×ÓÏµÍ³µÄÊµÏÖµÄÒ»¸ö¹Ø¼ü±í½á¹¹£¬ÆäÊµÊÇÒ»¸öhashÁ´½á¹¹£¬Ö»ÒªÓĞnetlinkÌ×½Ó×ÖĞÎ³É£¬²»¹ÜÊÇÄÚºËµÄ»¹ÊÇÓÃ»§¿Õ¼äµÄ£¬¶¼Òªµ÷ÓÃnetlink_insert½«netlinkÌ×½Ó×Ö±¾ÉíºÍËüµÄĞÅÏ¢Ò»²¢²åÈëµ½Õâ¸öÁ´±í½á¹¹ÖĞ£¨ÓÃ»§Ì¬Ì×½Ó×ÖÔÚbindÏµÍ³µ÷ÓÃµÄÊ±ºòµ÷ÓÃnetlink_insert²åÈënl_table£»ÄÚºËÌ×½Ó×ÖÊÇÔÚ´´½¨µÄÊ±ºòµ÷ÓÃnetlink_insert²åÈënl_table£©£¬È»ºóÔÚ·¢ËÍÊ±£¬Ö»Òªµ÷ÓÃnetlink_lookup±éÀúÕâ¸ö±í¾Í¿ÉÒÔ¿ìËÙ¶¨Î»Òª·¢ËÍµÄÄ¿±êÌ×½Ó×Ö¡£
3. sock_register(&netlink_family_ops)
Ìí¼ÓÌ×½Ó×Ö²ãĞ­Òéhandler£¬´´½¨ÓÃ»§¿Õ¼änetlinkÌ×½Ó×ÖµÄÊ±ºò½«µ÷ÓÃnetlink_family_opsÌá¹©µÄ·½·¨netlink_create¡£
4. rtnetlink_init
£¨1£©´´½¨rtnetlinkÄÚºËÄÚºËÌ×½Ó×Ö
rtnetlinkÌ×½Ó×Ö×¨ÓÃÓÚÁªÍøµÄnetlinkÌ×½Ó×Ö£¬ÓÃÓÚÂ·ÓÉÏûÏ¢¡¢ÁÚ½ÓÏûÏ¢¡¢Á´Â·ÏûÏ¢ºÍÆäËûÍøÂç×ÓÏµÍ³ÏûÏ¢¡£
ÔÚ´´½¨ÄÚºËÌ×½Ó×ÖÖ®ºó£¬µ÷ÓÃnetlink_insert(sk, 0)½«´ËÌ×½Ó×Ö²åÈëµ½nl_table£¬²Å¿ÉÒÔ½ÓÊÕÓÃ»§¿Õ¼ä·¢ËÍµÄnetlinkÏûÏ¢¡£
netlink_kernel_cfg½á¹¹input»Øµ÷º¯Êırtnetlink_rcv£¬½«¸³Öµ¸ønetlink_sockµÄ³ÉÔ±netlink_rcv£¬ÓÃÓÚ½ÓÊÕ´ÓÓÃ»§¿Õ¼ä·¢ËÍµÄÊı¾İ£»
netlink_kernel_create´´½¨ÄÚºËÌ×½Ó×Ö²¢±£´æÔÚÍøÂçÃüÃû¿Õ¼ä¶ÔÏónetµÄ±äÁ¿rtnl
```
register_pernet_subsys(&rtnetlink_net_ops)
    rtnetlink_net_init
        sk = netlink_kernel_create(net, NETLINK_ROUTE, &cfg)
            __netlink_create(net, sock, cb_mutex, unit, 1)
            netlink_insert(sk, 0) //½«´ËÄÚºËÌ×½Ó×Ö²åÈënl_table±í£¬´ËÌ×½Ó×Ö²Å¿ÉÒÔ½ÓÊÕÓÃ»§¿Õ¼ä·¢ËÍµÄnetlinkÏûÏ¢
            nlk_sk(sk)->netlink_rcv = cfg->input //netlink_rcv½«½ÓÊÕ´ÓÓÃ»§¿Õ¼ä»òÕßÄÚºË¿Õ¼ä·¢ËÍµÄÊı¾İ
            nl_table[unit].bind = cfg->bind
        net->rtnl = sk //½«´ËÌ×½Ó×Ö¸³Öµ¸øÍøÂçÃüÃû¿Õ¼äµÄ±äÁ¿rtnl
```
£¨2£©µ÷ÓÃrtnl_registerÎªnetlinkÏûÏ¢×¢²á»Øµ÷º¯Êı£¬Èçrtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_newlink, NULL, 0)ÏûÏ¢£¬½«rtnl_newlink×÷ÎªRTM_GETLINKÏûÏ¢µÄrtnl_dump_ifinfo»Øµ÷º¯Êı¼ÓÈëµ½rtnl_msg_handlers±íÏàÓ¦µÄÌõÄ¿ÖĞ£»ÔÚNETLINK_ROUTEÄÚºËÌ×½Ó×Ö½ÓÊÕÓÃ»§¿Õ¼äÏûÏ¢µÄ´¦Àíº¯Êırtnetlink_rcvÖĞ£¬½«¸ù¾İÏûÏ¢ÀàĞÍRTM_GETLINK´Órtnl_msg_handlers»ñÈ¡Æä´¦Àíº¯Êırtnl_newlink»òÕßrtnl_dump_ifinfo¡£
```
rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_getlink,
       rtnl_dump_ifinfo, 0);
```
# NETLINK_ROUTEÌ×½Ó×Ö
iprout2¹¤¾ß¼¯ipÃüÁî²ÉÓÃNETLINK_ROUTEÌ×½Ó×ÖÀ´ÓëÄÚºËÍ¨ĞÅ£¬»ñÈ¡ÍøÂç×ÓÏµÍ³µÄÂ·ÓÉ£¬Á´Â·µÈĞÅÏ¢£»ip -s link ls eth0»ñÈ¡eth0ÍøÂç½Ó¿ÚÍ³¼ÆĞÅÏ¢£¬ÆäÊä³ö£º
![image](/images/posts/network/netlink/ip_s_link_ls_eth0.png)

> iproute2Ô´Âëgit://git.kernel.org/pub/scm/network/iproute2/iproute2.git

ÎÒÃÇ½«ÒÔÕâÌõÃüÁîÎªÀı£¬Î§ÈÆÏÂÍ¼À´½²ÊöNETLINK_ROUTEÌ×½Ó×Ö´Ó³õÊ¼»¯£¬´´½¨socket£¬bind£¬sendmsgµ½recvmsgµÄÄÚºË¿Õ¼äÈ«¹ı³Ì¡£
![image](/images/posts/network/netlink/netlink_route.png)

´ÓÉÏÍ¼¿ÉÒÔ¿´³ö£¬NETLINK_ROUTEÌ×½Ó×ÖÍ¨ĞÅ¹ı³Ì×ÜÌåÎ§ÈÆÁ½¸öÊı×éÕ¹¿ª£¬¼´nl_tableºÍrtnl_msg_handlers£¬Í¼ÖĞ±êÊ¶³öÕû¸ö¹ı³ÌµÄĞòºÅ¡£
³õÊ¼»¯¹ı³ÌÒÑ¾­ÔÚÉÏÃæ¡°netlink³õÊ¼»¯¡±²¿·Ö½²¹ıÁË£¬²»ÔÙÖØ¸´¡£
## socketÏµÍ³µ÷ÓÃ
socketÏµÍ³µ÷ÓÃ½«´´½¨ÓÃ»§¿Õ¼änetlinkÌ×½Ó×Ö£¬Æädomain²ÎÊıAF_NETLINK£¬ÊÇprotocolÊÇNETLINK_ROUTE¡£
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
                    sk_alloc(net, PF_NETLINK, GFP_KERNEL, &netlink_proto, kern) //·ÖÅästruct netlink_sockÌ×½Ó×Ö£¨¼Ì³ĞÍ¨ÓÃÌ×½Ó×Östruct sock£©
                    sock_init_data(sock, sk)
                        sk->sk_rcvbuf	=	sysctl_rmem_default //Ì×½Ó×Ö³õÊ¼»¯Ê±½«/proc/sys/net/core/rmem_defaultÎªÆä³õÊ¼½ÓÊÕ»º´æ£¬ÊÓÇé¿öºóÃæ»áÓÃsetsockoptÖØĞÂµ÷Õû
                        sk->sk_sndbuf	=	sysctl_wmem_default
                        sk->sk_data_ready	=	sock_def_readable //µ±Ì×½Ó×ÖµÄsk_receive_queueÊÕµ½°ü£¬sock_def_readable½«»½ĞÑrecvmsgº¯Êı½ÓÊÕ¡£
        setsockopt(rth->fd, SOL_SOCKET, SO_SNDBUF //ÉèÖÃÌ×½Ó×Ö·¢ËÍ»º´æ´óĞ¡
        setsockopt(rth->fd, SOL_SOCKET, SO_RCVBUF //ÉèÖÃÌ×½Ó×Ö½ÓÊÕ»º´æ´óĞ¡
        getsockname(rth->fd, (struct sockaddr *)&rth->local
                    init_waitqueue_head(&nlk->wait)
```

## bindÏµÍ³µ÷ÓÃ
localÊÇ±¾µØsocketµØÖ·£¨struct sockaddr_nl½á¹¹£©,Æänl_pidÉèÖÃÎª0£¨Í¨³£Ó¦¸ÃÎªµ±Ç°½ø³Ìid£©£¬ÉèÖÃÎª0Ò²Ã»ÓĞ¹ØÏµ£¬ºóÃæbindÏµÍ³µ÷ÓÃ´ËsocketµÄops£¨netlink_ops£¬ÔÚsocketÏµÍ³µ÷ÓÃ´´½¨socketµÄÊ±ºò¸³Öµ£©º¯ÊıÖ¸Õënetlink_bind£¬netlink_bind½«µ÷ÓÃnetlink_autobind½«¶ÔÓ¦netlink_sockµÄ³ÉÔ±portidºÍbound¶¼ÉèÖÃÎªµ±Ç°Ïß³ÌµÄthread group id(tgid)£»²¢µ÷ÓÃ__netlink_insert½«´ËÓÃ»§Ì¬Ì×½Ó×ÖsockºÍÏà¹ØĞÅÏ¢portid²åÈëµ½nl_table[sk->sk_protocol]ÖĞ£¬ÕâÑù´ËÓÃ»§Ì¬Ì×½Ó×Ö¾Í¿ÉÒÔ½ÓÊÕÀ´×ÔÄÚºËµÄnetlinkÏûÏ¢¡£
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

## sendmsgÏµÍ³µ÷ÓÃ
```
do_iplink
    iplink_modify(RTM_NEWLINK,
        iplink_parse
        rtnl_talk
            sendmsg(rtnl->fd, &msg, 0)
```
ip linkµÄÏµÍ³µ÷ÓÃÊÇsendmsg£¬¶ÔÓ¦ÄÚºËº¯ÊıÊÇ__sys_sendmsg
```
__sys_sendmsg(fd, msg, flags, true)
    ___sys_sendmsg(sock, msg, &msg_sys, flags, NULL, 0)
        sock_sendmsg(sock, msg_sys)
            sock_sendmsg_nosec(sock, msg)
                sock->ops->sendmsg(sock, msg, msg_data_left(msg))
                    netlink_sendmsg
                        netlink_unicast
                            sk = netlink_getsockbyportid(ssk, portid)
                                netlink_lookup(sock_net(ssk), ssk->sk_protocol, portid)//ÔÚnl_table²éÕÒ°ó¶¨µÄÄÚºËnetlinkÌ×½Ó×Ö
                            netlink_unicast_kernel(sk, skb, ssk)
                                nlk->netlink_rcv(skb) 
                                    rtnetlink_rcv/netlink_rcv_skb/rtnetlink_rcv_msg//Ö´ĞĞÄÚºËnetlinkÌ×½Ó×ÖµÄ½ÓÊÕº¯Êırtnetlink_rcv½ÓÊÕÓÃ»§ÏûÏ¢¡£
```
## recvmsgÏµÍ³µ÷ÓÃ
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
                        __skb_try_recv_from_queue //´Ósocket buffer (&sk->sk_receive_queue)æ??°ü
```
ÄÚºË·¢ËÍnetlinkÏûÏ¢¸øÓÃ»§µÄ¹ı³Ì£¬¼´½«skb_buffĞ´ÈëÓÃ»§netlinkÌ×½Ó×ÖµÄsk_receive_queue¡£ÏÂÃæ´Órtnetlink_rcv_msg¿ªÊ¼·ÖÎö£º
```
rtnetlink_rcv_msg
    rtnl_get_link(family, type)//ÔÚrtnl_msg_handlers±íÖĞ²éÑ¯ÓÉrtnl_register×¢²áµÄtypeÏûÏ¢ÀàĞÍµÄ»Øµ÷º¯Êı£¨´Ë´¦ÊÇRTM_GETLINKÏûÏ¢µÄ»Øµ÷º¯Êırtnl_dump_ifinfo£©
        netlink_dump_start(rtnl, skb, nlh, &c)
            netlink_lookup(sock_net(ssk), ssk->sk_protocol, NETLINK_CB(skb).portid) //ÔÚnl_table²éÕÒ°ó¶¨µÄÓÃ»§Ì¬netlinkÌ×½Ó×Ö
            netlink_dump(sk) //´Ë´¦µÄ²ÎÊıÒÑ¾­ÊÇÓÃ»§Ì¬Ì×½Ó×Ö
                skb = alloc_skb(alloc_size  //·ÖÅäskb»º³åÇø
                skb_reserve(skb, skb_tailroom(skb) - alloc_size)
                netlink_skb_set_owner_r(skb, sk) //½«ÓÃ»§Ì¬sock¸³Öµ¸øskb->sk
                nlk->dump_done_errno = cb->dump(skb, cb) //ÕâÀï½«Ö´ĞĞrtnl_dump_ifinfo»Øµ÷º¯Êı
                __netlink_sendskb(sk, skb)
                    netlink_deliver_tap(sock_net(sk), skb)
                    skb_queue_tail(&sk->sk_receive_queue, skb) //½«skb¼ÓÈëµ½ÓÃ»§Ì¬sockµÄsk_receive_queue¶ÓÁĞ
                    sk->sk_data_ready(sk) //µ÷ÓÃsk_data_ready»½ĞÑnetlink_recvmsg½øĞĞ½ÓÊÕ
                        wake_up_interruptible_sync_poll(&wq->wait
```
# Í¨ÓÃNETLINK_GENERICÌ×½Ó×Ö
netlinkĞ­Òé´ØÊı×î´ó32¸ö£¨MAX_LINKS£©£¬ÎªÖ§³Ö¸ü¶àµÄĞ­Òé´Ø£¬¿ª·¢ÁËÍ¨ÓÃnetlink´ØNETLINK_GENERIC¡£Í¨ÓÃnetlinkÒÔnetlinkĞ­ÒéÎª»ù´¡£¬Ê¹ÓÃÆäAPI£¬¾ÍÏñnetlink¶àÂ·¸´ÓÃÆ÷¡£Í¨ÓÃnetlinkĞ­ÒéÒÑ±»ÓÃÓÚÖÚ¶àµÄÄÚºË×ÓÏµÍ³£¬ÈçACPI×ÓÏµÍ³£¬ÈÎÎñÍ³¼ÆĞÅÏ¢´úÂë£¬¹ıÈÈÊÂ¼ş£¬wirelessÎŞÏß×ÓÏµÍ³µÈ¡£
»ñÈ¡Í¨ÓÃnetlink¿ØÖÆÆ÷´Ø²ÎÊıÃüÁî`genl ctrl getname nlctrl`µÄÊä³öÊÇ£º
![image](/images/posts/network/netlink/genl_ctrl_getname_nlctrl.png)

ÎÒÃÇ½«ÒÔÕâ¸öÃüÁîÎªÀı£¬Î§ÈÆÏÂÍ¼À´½²ÊöÍ¨ÓÃNETLINK_GENERICÌ×½Ó×ÖµÄÍ¨ĞÅ¹ı³Ì¡£
´ÓÉÏÍ¼¿ÉÒÔ¿´³ö£¬Í¨ÓÃnetlinkÌ×½Ó×ÖÍ¨ĞÅ¹ı³ÌÒ²ÊÇÎ§ÈÆÁ½¸öÊı×éÕ¹¿ª£¬¼´nl_tableºÍgenl_fam_idr£¬Í¼ÖĞ±êÊ¶³öÕû¸ö¹ı³ÌµÄĞòºÅ¡£
![image](/images/posts/network/netlink/netlink_generic.png)
## Í¨ÓÃnetlinkÌ×½Ó×Ö³õÊ¼»¯
Í¨ÓÃnetlinkĞ­Òé´ØÊ¹ÓÃnetlinkĞ­ÒéµÄAPI£¬Æä³õÊ¼»¯Í¬ÑùĞèÒªcore_initcall(netlink_proto_init)£¬»¹°üº¬ÆäÌØÓĞµÄ³õÊ¼»¯²¿·Ösubsys_initcall(genl_init)¡£genl_init×îÖØÒªµÄ¹¤×÷ÊÇ´´½¨Í¨ÓÃNETLINK_GENERICÄÚºËÌ×½Ó×Ö£¬´Ë´¦½ÓÊÕÓÃ»§¿Õ¼äÏûÏ¢µÄinputº¯ÊıÊÇgenl_rcv£»´ËÍâ»¹×¢²áÁËÍ¨ÓÃnetlinkÌ×½Ó×Ö¿ØÖÆÆ÷´Øgenl_ctrl£¬´Ë¿ØÖÆÆ÷´Øgenl_ctrlÊÇÍ¨ÓÃnetlinkĞ­Òé»úÖÆµÄµÚÒ»¸öÓÃ»§£¬ÆäÓĞÒ»¸öÖØÒª×÷ÓÃ£¬¾ÍÊÇÆäËûÍ¨ÓÃÌ×½Ó×Ö´ØµÄÓÃ»§¿Õ¼äÓ¦ÓÃ³ÌĞòÒªÊ¹ÓÃ´Ë¿ØÖÆÆ÷´ØÀ´»ñÈ¡idr±íµÄid²ÅÄÜÓëÄÚºËÍ¨ĞÅ¡£iproute2µÄgenlÃüÁî¾ÍÊÇÍ¨¹ı´Ë¿ØÖÆÆ÷´Øgenl_ctrlÀ´²éÑ¯ÄÚºËËùÓĞ×¢²áµÄÍ¨ÓÃnetlink´ØµÄ¸÷ÖÖ²ÎÊı£¬Èçid£¬±¨Í·³¤¶È£¬×î´óÊôĞÔÊıµÈ¡£ÆäËûĞèÒªÊ¹ÓÃÍ¨ÓÃnetlinkĞ­ÒéµÄ×ÓÏµÍ³Ö»ĞèÏÈ¶¨Òågenl_family¶ÔÏó£¬È»ºóµ÷ÓÃgenl_register_familyÏòidr±ígenl_fam_idr½øĞĞ×¢²á¼´¿É¡£
```
subsys_initcall(genl_init)
    genl_register_family(&genl_ctrl) //×¢²áÍ¨ÓÃnetlinkÌ×½Ó×Ö¿ØÖÆÆ÷´Ø£¨genl_family£©
        idr_alloc(&genl_fam_idr, family //½«¿ØÖÆÆ÷´ØÌí¼Óµ½idr±ígenl_fam_idr
        genl_ctrl_event(CTRL_CMD_NEWFAMILY, family, NULL, 0)
    register_pernet_subsys(&genl_pernet_ops)
        net->genl_sock = netlink_kernel_create(net, NETLINK_GENERIC, &cfg) //´´½¨Í¨ÓÃnetlinkÄÚºËÌ×½Ó×Ö
```
Í¨ÓÃnetlink»úÖÆÊ¹ÓÃidr»úÖÆ½øĞĞĞ­Òé´Øgenl_familyµÄ¹ÜÀí£º
1. ÔÚ genl_register_family½«genl_familyÌí¼Óµ½idr±ígenl_fam_idr£¬²¢»ñÈ¡id£»
2. ÓÃ»§¿Õ¼ä¸ù¾İgenl_familyµÄÃû×ÖÀ´²éÑ¯»ñÈ¡ÆäÔÚgenl_fam_idrµÄid£¬²¢×÷Îª²ÎÊı·¢¸øÄÚºËÍ¨ÓÃnetlinkÌ×½Ó×Ö£¬ÕâÀïÓÃÁËÍ¨ÓÃnetlinkĞ­Òé¿ØÖÆÆ÷´Ø£¬ËùÒÔÆäidĞèÒª±»¹Ì¶¨ÎªGENL_ID_CTRL(0x10)¡£Õâ²¿·ÖÀı×Ó¿ÉÒÔ²Î¿¼ÄÚºË¹¤¾ßgetdelayµÄ´úÂë[tools/accouting/getdelay.c](https://github.com/torvalds/linux/blob/master/tools/accounting/getdelays.c)
3. Í¨ÓÃnetlinkÄÚºËÌ×½Ó×Öinputº¯Êıgenl_rcv½«¾İ´Ëid²éÑ¯idr±ígenl_fam_idr»ñÈ¡¶ÔÓ¦µÄgenl_family£¬´Ó¶ø»ñÈ¡ºóĞø²Ù×÷º¯Êıgenl_ops¡£
¹ØÓÚidr»úÖÆ£¬²Î¿¼[idr»úÖÆ](https://blog.csdn.net/coldsnow33/article/details/13503183)
## Í¨ÓÃnetlinkÌ×½Ó×ÖÊÕ·¢ÏûÏ¢
```
genl_rcv
    genl_rcv_msg
        family = genl_family_find_byid(nlh->nlmsg_type)
            idr_find(&genl_fam_idr, id)//¸ù¾İÍ¨ÓÃnetlinkÌ×½Ó×Ö´Øid´Óidr±ígenl_fam_idr²éÕÒ»ñÈ¡genl_family
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
                                        sock = netlink_lookup(sock_net(ssk), ssk->sk_protocol, portid)//ÔÚnl_table²éÕÒ°ó¶¨µÄÓÃ»§Ì¬netlinkÌ×½Ó×Ö                                    netlink_attachskb(sk, skb, &timeo, ssk) //½«sockºÍsk_buff°ó¶¨ÔÚÒ»Æğ
                                    netlink_sendskb(sk, skb)
                                        skb_queue_tail(&sk->sk_receive_queue, skb)
```

# ÆäËûnetlinkĞ­Òé´ØÌ×½Ó×Ö
## NETLINK_NETFILTER socket
ÓÉnfnetlink_net_init´´½¨¡£
```
<net/netfilter/nfnetlink.c>
module_init(nfnetlink_init)
    register_pernet_subsys(&nfnetlink_net_ops) -> nfnetlink_net_init
```
## NETLINK_KOBJECT_UEVENT socket
ÓÃÓÚÉè±¸¹ÜÀí£¬ÓÉuevent_net_init´´½¨¡£
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
## NETLINK_SOCK_DIAG socket Ì×½Ó×Ö¼àÊÓÌ×½Ó×Ö
ÓÉsock_diag_init´´½¨¡£

# netlinkÓÃ»§¿Õ¼ä³ÌĞò
## netlinkÌ×½Ó×Ö¿âlibnl
netlinkÌ×½Ó×Ö¿âlibnlÌá¹©API·ÃÎÊ»ùÓÚnetlinkĞ­ÒéµÄÄÚºË½Ó¿Ú£¬Æä¹Ù·½ÍøÕ¾ÊÇhttps://www.infradead.org/~tgr/libnl/¡£³ıºËĞÄ¿âlibnlÖ®Íâ£¬»¹ÓĞÍ¨ÓÃnetlink´Ølibnl-genl£¬Â·ÓÉÑ¡Ôñ´Ølibnl-route¼°netfilter´Ølibnl-nfµÈ¡£


## netlinkÏûÏ¢±¨Í·ºÍÊı¾İ½á¹¹
²Î¿¼¡¶¾«Í¨linuxÄÚºËÍøÂç¡·¼°½áºÏÓÃ»§Ó¦ÓÃ³ÌĞòºÍÄÚºË´úÂë½øĞĞÀí½â¡££¨´ıÍêÉÆ£©

## Í¨ÓÃnetlink±¨Í·ºÍÊı¾İ½á¹¹
²Î¿¼¡¶¾«Í¨linuxÄÚºËÍøÂç¡·¼°½áºÏÓÃ»§Ó¦ÓÃ³ÌĞòºÍÄÚºË´úÂë½øĞĞÀí½â¡££¨´ıÍêÉÆ£©

# ²Î¿¼×ÊÁÏ
* ¡¶¾«Í¨linuxÄÚºËÍøÂç¡·£¬Ó¢ÎÄÃû¡¶Linux Kernel Networking - Implementation and Theory¡·
×÷Õß£ºÒÔÉ«ÁĞÍøÂç×¨¼ÒRami Rosen£¬ÏÖintel DPDK Tech Leader£¬¸öÈËÖ÷Ò³http://ramirose.wixsite.com/ramirosen
* [linuxµÄnetlink»úÖÆ](https://blog.csdn.net/dog250/article/details/5303430)
×÷Õß£ºcsdn²©¿ÍÅÅÃû14µÄdog250

