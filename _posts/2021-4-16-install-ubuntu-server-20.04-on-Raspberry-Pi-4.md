---
layout: post
title: "Raspberry Pi4 安装 ubuntu server 20.04"
category: build
tags: Raspberry-Pi4 ubuntu-server
author: jgsun
---

* content
{:toc}

# 安装Ubuntu Server 20.04.2 LTS

ubuntu 官方guide： [How to install Ubuntu Server on your Raspberry Pi](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#5-install-a-desktop)
非常详细，这里就不再赘述，照着操作即可。
RaspberryPi wiki 提供更多细节，例如启用serial console等，感兴趣可以参考。
>For more details about Raspberry Pi specific packages included with this image and further customisations, such as accelerated video drivers and optional package repositories, you can refer to the [RaspberryPi wiki](https://wiki.ubuntu.com/ARM/RaspberryPi#Packages) .

![image](/images/posts/build/pi4.png)












所不同的是，这里在配置 network-config 的时候给eth0设置了固定IP，用来 SSH 连接树莓派。
network-config 文件设置如下：


    version: 2

    ethernets:
      eth0:
        dhcp4: true
        optional: true
        addresses:
           - 10.10.10.1/24
    wifis:
      wlan0:
        dhcp4: true
        optional: true
        access-points:
          CMCC-bEZb:
            password: "cthpa3q6"
    #      myworkwifi:
    #        password: "correct battery horse staple"
    #      workssid:
    #        auth:
    #          key-management: eap
    #          method: peap
    #          identity: "me@example.com"
    #          password: "passw0rd"
    #          ca-certificate: /etc/my_ca.pem


# 设置eth0 static IP，建立SSH连接

可能是家里路由器的原因，因为通过无线网卡 SSH 连接树莓派不少包`Connection timed ou`t就是报`reset by peer`；而且无线网卡采用 DHCP 获取 IP，每次开机都会变；而且无线网卡的速度不如有线网卡。所以装机的时候给 eth0 设置了 static IP，通过这个固定 IP 来  SSH 连接树莓派

ping 无线网卡：
   
     [jiangusu.N-20L6PF1QDMGJ] ➤ ping 192.168.1.7


    Pinging 192.168.1.7 with 32 bytes of data:
    Reply from 192.168.1.7: bytes=32 time=2ms TTL=64
    Reply from 192.168.1.7: bytes=32 time=2ms TTL=64
    Reply from 192.168.1.7: bytes=32 time=3ms TTL=64
    Reply from 192.168.1.7: bytes=32 time=4ms TTL=64


ping 有些网卡 eth0 time， 可见小于 1 ms：


    [jiangusu.N-20L6PF1QDMGJ] ➤ ping 10.10.10.1


    Pinging 10.10.10.1 with 32 bytes of data:
    Reply from 10.10.10.1: bytes=32 time<1ms TTL=64
    Reply from 10.10.10.1: bytes=32 time<1ms TTL=64
    Reply from 10.10.10.1: bytes=32 time<1ms TTL=64
    Reply from 10.10.10.1: bytes=32 time<1ms TTL=64


# 参考文档
[How to install Ubuntu Server on your Raspberry Pi](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#5-install-a-desktop)

[RaspberryPi wiki](https://wiki.ubuntu.com/ARM/RaspberryPi#Packages)

[ cloud-init  network-config-format-v2 ](https://cloudinit.readthedocs.io/en/latest/topics/network-config-format-v2.html)