---
layout: post
title:  "升级github buildroot自建仓库"
categories: build
tags: git rebase, remote upstream, buildroot, github
author: jgsun
---


* content
{:toc}
# 1. Overview
去年在自己的github账号创建了一个buildroot仓库，有一些自己的commit，今年官方buildroot稳定版本升级到了2020.02，遂也想升级自建的buiildrot仓库，今天尝试了使用git rebase将自建仓库升级到2020.02，并保留了自己的commit历史。本文记录了升级log。













# 2. rebase 步骤
# 2.1 add upstream并fetch
```
jgsun@VirtualBox:~/work/buildroot-jgsun$ git remote add upstream https://github.com/buildroot/buildroot.git
jgsun@VirtualBox:~/work/buildroot-jgsun$ git fetch upstream
jgsun@VirtualBox:~/work/buildroot-jgsun$ git checkout master
```
# 2.2 rabase 到upstream stable branch remotes/upstream/2020.02.x
```
jgsun@VirtualBox:~/work/buildroot-jgsun$ git rebase remotes/upstream/2020.02.x
First, rewinding head to replay your work on top of it...
Applying: configs: add qemu_arm_vexpress-fun_defconfig
Applying: board: arm-vexpress: add scripts
Applying: configs: add qemu_aarch64_virt-fun_defconfig
Applying: board: qemu: add scripts for QEMU practice
Applying: board: qemu: rename qemu-ifup -> qemu-ifup_virbr0
Applying: board: qemu: init /etc/qemu-ifup and /etc/qemu-ifdown
Applying: board: qemu: restore qemu-ifup
Applying: board: qemu: del scripts/qemu-ifdowna,scripts/qemu-ifdown and add start_qemu_aarch64.sh
Applying: system: profile: sepcify the PS1 env variable
Applying: board: add qemu/common/target_skeleton_extras/etc/fstab
Applying: board: update board/qemu/aarch64-virt/linux.config
Applying: configs: update qemu_aarch64_virt-fun_defconfig
Applying: board: update start_qemu_aarch64.sh
Applying: package: add if-stat
Using index info to reconstruct a base tree...
M package/Config.in
Falling back to patching base and 3-way merge...
Auto-merging package/Config.in
Applying: package: ifstat: remove comments
Applying: configs: qemu_aarch64_virt-fun_defconfig: add u-boot
```
# 2.3 更新github远程库
```
jgsun@VirtualBox:~/work/buildroot-jgsun$ git push -f origin master
Username for 'https://github.com': jgsun
Password for 'https://jgsun@github.com': 
Counting objects: 59317, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (27010/27010), done.
Writing objects: 100% (59317/59317), 15.40 MiB | 253.00 KiB/s, done.
Total 59317 (delta 38621), reused 52186 (delta 32034)
remote: Resolving deltas: 100% (38621/38621), completed with 3791 local objects.
remote: Checking connectivity: 59317, done.
To https://github.com/jgsun/buildroot
 + df19c94b18...c9d38f43bb master -> master (forced update)
```