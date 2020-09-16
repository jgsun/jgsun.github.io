---
layout: post
title:  "DHCP options实验"
categories: network
tags: dhcp, dhcp options, dhcp-client-identifier
author: jgsun
---


* content
{:toc}
# 1. Overview
在DHCP协议中，定义了一个option字段，该字段主要是用来扩展DHCP协议的。
本文学习了man dhcp-options(5) 和man dhcp-eval(5)，并实验用 option dhcp-client-identifier控制dhcp server分配地址。
![image](/images/posts/network/dhcp/options-logo.jpg)





















# 2.  man page: [dhcp-options(5) - Linux man page](https://linux.die.net/man/5/dhcp-options)
# 2.1 man dhcp-options(5) 内容
![image](/images/posts/network/dhcp/man-dhcp-options.png)

## 2.2 option data类型
![image](/images/posts/network/dhcp/option-data.png)

## 2.3 man dhcp-eval(5)
The dhcp-eval(5) manual page describes how to write expressions.
![image](/images/posts/network/dhcp/man-dhcp-eval.png)


# 3. 用 option dhcp-client-identifier控制dhcp server分配地址
## 3.1 dhcpd.conf
https://github.com/jgsun/buildroot/blob/master/board/qemu/common/target_skeleton_extras/root/dhcp/dhcpd.conf
```
class "slot1" {
  match if option dhcp-client-identifier = "slot1";
}
class "slot2" {
  match if option dhcp-client-identifier = "slot2";
}
subnet 10.254.239.0 netmask 255.255.255.0 {
  pool {
    allow members of "slot1";
    range 10.254.239.81 10.254.239.81;
  }
  pool {
    allow members of "slot2";
    range 10.254.239.82 10.254.239.82;
  }
}
```
## 3.2 dhclient.conf
https://github.com/jgsun/buildroot/blob/master/board/qemu/common/target_skeleton_extras/root/dhcp/dhclient-c1.conf
https://github.com/jgsun/buildroot/blob/master/board/qemu/common/target_skeleton_extras/root/dhcp/dhclient-c2.conf
```
send dhcp-client-identifier "slot2";
send vendor-class-identifier "lt_type=lwlt-c sw_ver=2012.888";
```
## 3.3 启动脚本
https://github.com/jgsun/buildroot/blob/master/board/qemu/common/target_skeleton_extras/root/dhcp/create_itf.sh
https://github.com/jgsun/buildroot/blob/master/board/qemu/common/target_skeleton_extras/root/dhcp/S80dhcp-client
## 3.4 实验结果
```
root@~/dhcp# vi /var/lib/dhcp/dhcpd.leases
# This lease file was written by isc-dhcp-4.4.1                            
                                               
# authoring-byte-order entry is generated, DO NOT DELETE
authoring-byte-order little-endian;                     
                                   
server-duid "\000\001\000\001\307\222\274\233\002\330\360h\036\020";
                                                                    
lease 10.254.239.81 {
  starts 4 1970/01/01 00:01:33;
  ends 4 1970/01/01 00:11:33;  
  cltt 4 1970/01/01 00:01:33;
  binding state active;      
  next binding state free;
  rewind binding state free;
  hardware ethernet 0a:19:e4:41:d7:69;
  uid "slot1";                        
  set vendor-class-identifier = "lt_type=lwlt-c sw_ver=2012.888";
  option agent.circuit-id "r_veth";                              
}                                  
lease 10.254.239.82 {
  starts 4 1970/01/01 00:01:50;
  ends 4 1970/01/01 00:11:50;  
  cltt 4 1970/01/01 00:01:50;
  binding state active;      
  next binding state free;
  rewind binding state free;
  hardware ethernet 8a:a5:0f:65:8a:96;
  uid "slot2";                        
  set vendor-class-identifier = "lt_type=lwlt-c sw_ver=2012.888";
  option agent.circuit-id "r_veth";                              
}   
```