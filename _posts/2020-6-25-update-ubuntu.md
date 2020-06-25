---
layout: post
title:  "升级ubuntu20.04"
categories: build
tags: ubuntu, wsl2, virtualbox
author: jgsun
---


* content
{:toc}

# 1. overview
几个月前，ubuntu20.04就面世了，不想从零安装，从现用的18.04升级到20.04。
![image](/images/posts/build/wsl2/neofetch.png)










# 2. 升级过程
需要先将系统软件升级到最新版本。
```
jgsun@DESKTOP-RGKRB2C:~$ sudo do-release-upgrade
[sudo] password for jgsun:
Checking for a new Ubuntu release
Please install all available updates for your release before upgrading.
jgsun@DESKTOP-RGKRB2C:~$ sudo apt full-upgrade
```
#3. ubuntu20.04 openssh问题
升级过程中，弹出窗口提示是否更新openssh的config，选择的保留现在的配置（/etc/ssh/sshd_config，因为原来修改过端口），升级完成之后，使用srcureCRT连接ubuntu虚拟机，出现提示：
```
No compatible key exchange method. The server supports these methods: curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256
```
网上搜索发现有说是secureCRT的版本太低了，查看opssh状态正常：
```
jgsun@VirtualBox:/etc/ssh$ sudo service ssh status
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2020-06-22 08:53:10 CST; 23s ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 5057 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 5077 (sshd)
      Tasks: 1 (limit: 2879)
     Memory: 1.2M
     CGroup: /system.slice/ssh.service
             └─5077 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups

6月 22 08:53:10 VirtualBox systemd[1]: Starting OpenBSD Secure Shell server...
6月 22 08:53:10 VirtualBox sshd[5077]: Server listening on 0.0.0.0 port 2222.
6月 22 08:53:10 VirtualBox sshd[5077]: Server listening on :: port 2222.
6月 22 08:53:10 VirtualBox systemd[1]: Started OpenBSD Secure Shell server.\
```
不能更新secureCRT，想到嵌入式板卡使用dropbear作为ssh-server，查看系统原来dropbear已经在运行了，端口是22：
```
jgsun@VirtualBox:/etc/ssh$ ps -elf | grep dropbear
1 S root 650 1 0 80 0 - 1140 - 6月21 ? 00:00:00 /usr/sbin/dropbear -p 22 -W 65536
5 S root 4929 650 0 80 0 - 1249 - 08:45 ? 00:00:00 /usr/sbin/dropbear -p 22 -W 65536
```
修改virtualbox的设置，将端口映射改为22，使用secureCRT ssh连接utunu成功！

最后一步，卸载openssh-server：`jgsun@VirtualBox:~$ sudo apt remove openssh-server`
