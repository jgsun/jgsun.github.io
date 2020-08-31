---
layout: post
title:  "DHCP relay实验"
categories: network
tags: dhcp, dhcp relay, dhcp server, dhcp client
author: jgsun
---


* content
{:toc}
# 1. Overview
项目需要研究验证open source的ISC DHCP功能，本文讲述了dhcp relay的实验过程。
* About ISC DHCP
ISC DHCP offers a complete open source solution for implementing DHCP servers, relay agents, and clients. ISC DHCP supports both IPv4 and IPv6, and is suitable for use in high-volume and high-reliability applications. DHCP is available for free download under the terms of the MPL 2.0 license.
官网及其文档：
https://www.isc.org/ 
https://kb.isc.org/docs/using-this-knowledgebase#

![image](/images/posts/network/dhcp/dhcp_logo.png)












# 2. 系统框图
本实验系统框图如下，使用linux 虚拟网络设备网桥bridge和veth，network namespace技术，在qemu-aarch64平台上面完成本实验。
![image](/images/posts/network/dhcp/sys_block.png)


* About network namespace 
network namespace 是实现网络虚拟化的重要功能，它能创建多个隔离的网络空间，它们有独自的网络栈信息。不管是虚拟机还是容器，运行的时候仿佛自己就在独立的网络中。

namespace s_ns运行DHCP server，网段是10.254.236.0/24，通过veth pair连接host；namespace r_ns运行DHCP relay；namespace c1_ns和c2_ns运行DHCP client；r_ns，c1_ns和c2_ns运行在同一个网段10.254.239.0/24，都通过veth pair连接host，并通过网桥br0互联；使能host  qemu-aarch64 ip_forward功能，让其成为路由器；设置s_ns和r_ns的路由，让他们space的网口s_veth和r_veth可以互相ping通。
实现上述操作的脚本如下：
下载地址：https://github.com/jgsun/buildroot/blob/master/board/qemu/common/target_skeleton_extras/root/dhcp/create_itf.sh
```
jgsun@VirtualBox:~/repo/buildroot-arm64/board/qemu/common/target_skeleton_extras$ cat root/dhcp/create_itf.sh 
#!/bin/sh
echo 1 > /proc/sys/net/ipv4/ip_forward

ip link add host_s_veth type veth peer name s_veth
ip link add host_c1_veth type veth peer name c1_veth
ip link add host_c2_veth type veth peer name c2_veth
ip link add host_r_veth type veth peer name r_veth

ip netns add s_ns
ip netns add r_ns
ip netns add c1_ns
ip netns add c2_ns

ip link set s_veth netns s_ns
ip link set r_veth netns r_ns
ip link set c1_veth netns c1_ns
ip link set c2_veth netns c2_ns

ip link set host_s_veth up
ip link set host_c1_veth up
ip link set host_c2_veth up
ip link set host_r_veth up

ip netns exec s_ns ip link set s_veth up
ip netns exec r_ns ip link set r_veth up
ip netns exec c1_ns ip link set c1_veth up
ip netns exec c2_ns ip link set c2_veth up

ip link add name br0 type bridge
ip link set br0 up
ip addr add 10.254.239.1/24 dev br0 
ip link set dev host_c1_veth master br0
ip link set dev host_c2_veth master br0
ip link set dev host_r_veth master br0

ip addr add 10.254.236.1/24 dev host_s_veth

ip netns exec s_ns ip addr add 10.254.236.2/24 dev s_veth
ip netns exec s_ns route add -net 10.254.239.0 netmask 255.255.255.0 dev s_veth
ip netns exec s_ns route add -net 10.254.239.0 netmask 255.255.255.0 gw 10.254.239.1 dev s_veth

ip netns exec r_ns ip addr add 10.254.239.2/24 dev r_veth
ip netns exec r_ns route add -net 10.254.236.0 netmask 255.255.255.0 dev r_veth
ip netns exec r_ns route add -net 10.254.236.0 netmask 255.255.255.0 gw 10.254.236.1 dev r_veth

# ip netns exec c1_ns ip addr add 10.254.239.41/24 dev c1_veth
ip netns exec c1_ns route add -net 10.254.236.0 netmask 255.255.255.0 dev c1_veth
ip netns exec c1_ns route add -net 10.254.236.0 netmask 255.255.255.0 gw 10.254.236.1 dev c1_veth

ip netns exec c2_ns route add -net 10.254.236.0 netmask 255.255.255.0 dev c2_veth
ip netns exec c2_ns route add -net 10.254.236.0 netmask 255.255.255.0 gw 10.254.236.1 dev c2_veth

ip netns exec s_ns /root/dhcp/S80dhcp-server start
ip netns exec r_ns /root/dhcp/S80dhcp-relay start
```




# 3. 实验步骤及抓包结果
```
tcpdump -i host_s_veth -w host_s_veth-1.pcap &
tcpdump -i br0 -w br0-1.pcap &
ip netns exec c1_ns tcpdump -i c1_veth -w c1_veth-1.pcap &
ip netns exec r_ns tcpdump -i r_veth -w r_veth-1.pcap &
ip netns exec s_ns tcpdump -i s_veth -w s_veth-1.pcap &

root@~/dhcp# ./create_itf.sh 
root@~/dhcp# ip netns exec c1_ns ./S80dhcp-client start c1
Starting DHCP client: cat: can't open '/etc/resolv.conf.*': No such file or directory
OK
root@~/dhcp# ip netns exec c1_ns ./S80dhcp-client stop
Stopping DHCP server: OK
root@~/dhcp# ip netns exec c2_ns ./S80dhcp-client start c2
Starting DHCP client: cat: can't open '/etc/resolv.conf.*': No such file or directory
OK

root@~/dhcp# killall tcpdump
19 packets captured13 packets captured
13 packets received by filter19 packets received by filter
0 packets dropped by kernel
0 packets dropped by kernel
14 packets captured
14 packets received by filter
0 packets dropped by kernel
11 packets captured
11 packets received by filter
0 packets dropped by kernel
20 packets capturedroot@~/dhcp# 
20 packets received by filter
0 packets dropped by kernel
[5]+  Done                       tcpdump -i host_s_veth -w host_s_veth-1.pcap
[4]+  Done                       tcpdump -i br0 -w br0-1.pcap
[3]+  Done                       ip netns exec c1_ns tcpdump -i c1_veth -w c1_veth-1.pcap
[2]+  Done                       ip netns exec r_ns tcpdump -i r_veth -w r_veth-1.pcap
[1]+  Done                       ip netns exec s_ns tcpdump -i s_veth -w s_veth-1.pcap
```
## 3.1 查看c1_ns和c2_ns的网口被分配IP
```
root@~/dhcp# ip netns exec c1_ns ifconfig
c1_veth   Link encap:Ethernet  HWaddr 06:AB:3A:D4:75:65  
          inet addr:10.254.239.41  Bcast:10.254.239.255  Mask:255.255.255.0
          inet6 addr: fe80::4ab:3aff:fed4:7565/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:64 errors:0 dropped:0 overruns:0 frame:0
          TX packets:21 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:6108 (5.9 KiB)  TX bytes:2898 (2.8 KiB)
root@~/dhcp# ip netns exec c2_ns ifconfig
c2_veth   Link encap:Ethernet  HWaddr 1A:5A:FA:96:FE:77  
          inet addr:10.254.239.42  Bcast:10.254.239.255  Mask:255.255.255.0
          inet6 addr: fe80::185a:faff:fe96:fe77/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:69 errors:0 dropped:0 overruns:0 frame:0
          TX packets:18 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:6878 (6.7 KiB)  TX bytes:1872 (1.8 KiB)
```
## 3.2 c1_ns和c2_ns的网口互相ping通：
```
root@~/dhcp# ip netns exec c2_ns ping -c 2 10.254.239.41
PING 10.254.239.41 (10.254.239.41): 56 data bytes
64 bytes from 10.254.239.41: seq=0 ttl=64 time=4.562 ms
64 bytes from 10.254.239.41: seq=1 ttl=64 time=1.202 ms

--- 10.254.239.41 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 1.202/2.882/4.562 ms
```
## 3.3 DHCP client 网口c1_veth报文
![image](/images/posts/network/dhcp/c1_veth.png)


## 3.4 DHCP relay网口r_veth报文
![image](/images/posts/network/dhcp/r_veth.png)

## 3.5 DHCP server网口s_veth报文
![image](/images/posts/network/dhcp/s_veth.png)

本实验DHCP所抓报文下载地址：https://github.com/jgsun/jgsun.github.io/tree/master/doc/dhcp
所用脚本下载地址：https://github.com/jgsun/buildroot/tree/master/board/qemu/common/target_skeleton_extras/root/dhcp