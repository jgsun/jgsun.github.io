---
layout: post
title: "SCSI UFS init view"
category: process_irq
tags: tmux tabby .tmux.conf
author: jgsun
---

* content
{:toc}

# Overview
本文简要介绍了 Linux SCSI UFS(Universal Flash Storage) 的初始化。契机是解一个 kernel boot 阶段 mount rootfs 失败的的问题： "VFS: Unable to mount root fs on unknown-block(8,6)"。所以，本文最后也介绍了这个问题的原因和解法。









# SCSI UFS init view
![image](/images/posts/scsi/scsi_ufs_init.png)


# Rootfs mount issue
在某一个项目中，我们使用 UFS LUN0 的 partition 6 作为 rootfs 的分区，启动参数是 "root=/dev/sda6"。但是在 Kernel 启动过程中，有较大概率发现此分区的分配盘符是 "/dev/sdb6", 从而导致系统 panic： "VFS: Unable to mount root fs on unknown-block(8,6)"。

SCSI UFS 初始化过程中，循环给每个 LUN 注册 scsi_device, 然后匹配 scsi_driver 从而调用 sd_probe 分配 index(sda, sdb, ...) 添加 block device。每个 LUN 是顺序添加，但是 sd_probe 是异步执行，这样就导致在 sd_probe 中分配的 index 可能失序，LUN0 所获分配的 index 是 sdb，其 partition 6 盘符就是sdb6， 导致 kernel 因为找不到 sda6 而挂载 rootfs 失败。
```
<drivers/scsi/sd.c>
static struct scsi_driver sd_template = {
	.gendrv = {
		.name		= "sd",
		.owner		= THIS_MODULE,
		.probe		= sd_probe,
		.probe_type	= PROBE_PREFER_ASYNCHRONOUS,
		.remove		= sd_remove,
		.shutdown	= sd_shutdown,
		.pm		= &sd_pm_ops,
	},
	.rescan			= sd_rescan,
	.init_command		= sd_init_command,
	.uninit_command		= sd_uninit_command,
	.done			= sd_done,
	.eh_action		= sd_eh_action,
	.eh_reset		= sd_eh_reset,
};
```
考虑到循环扫描 LUN，添加 scsi_device 是在一个 worker 执行，可以保证顺序，我一开始的解法是：将给每个 LUN 分配 index 挪到注册 scsi_device 的时候，这样就能保证盘符不会失序，从而解决了问题。

之后将这个 patch 提到了 Linux SCSI subsystem 社区 <https://patchwork.kernel.org/project/linux-scsi/patch/20220401005928.24140-1-quic_jianguos@quicinc.com/>, 被大牛们拒绝了，原因是 Kernel 已有更好的解决方案：
> Specifying a /dev/sd* device name as root device is wrong since such a 
name will change if a disk is added to the system or removed from the 
system. Please use one of the /dev/disk/by-*/* names as root device.

后来在 name_to_dev_t 函数的注释中也找到了相同的答案，这告诉我们：要注意读内核中的 comments，他们和代码一样重要，可能有你需要的答案！
```
init/do_mounts.c 
/*
 *	Convert a name into device number.  We accept the following variants:
 *
 *	1) <hex_major><hex_minor> device number in hexadecimal represents itself
 *         no leading 0x, for example b302.
 *	2) /dev/nfs represents Root_NFS (0xff)
 *	3) /dev/<disk_name> represents the device number of disk
 *	4) /dev/<disk_name><decimal> represents the device number
 *         of partition - device number of disk plus the partition number
 *	5) /dev/<disk_name>p<decimal> - same as the above, that form is
 *	   used when disk name of partitioned disk ends on a digit.
 *	6) PARTUUID=00112233-4455-6677-8899-AABBCCDDEEFF representing the
 *	   unique id of a partition if the partition table provides it.
 *	   The UUID may be either an EFI/GPT UUID, or refer to an MSDOS
 *	   partition using the format SSSSSSSS-PP, where SSSSSSSS is a zero-
 *	   filled hex representation of the 32-bit "NT disk signature", and PP
 *	   is a zero-filled hex representation of the 1-based partition number.
 *	7) PARTUUID=<UUID>/PARTNROFF=<int> to select a partition in relation to
 *	   a partition with a known unique id.
 *	8) <major>:<minor> major and minor number of the device separated by
 *	   a colon.
 *	9) PARTLABEL=<name> with name being the GPT partition label.
 *	   MSDOS partitions do not support labels!
 *
 *	If name doesn't have fall into the categories above, we return (0,0).
 *	block_class is used to check if something is a disk name. If the disk
 *	name contains slashes, the device name has them replaced with
 *	bangs.
 */

dev_t name_to_dev_t(const char *name)
{
	char s[32];
```

# References
[SCSI Interfaces Guide — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/driver-api/scsi.html)
[Universal Flash Storage — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/scsi/ufs.html?highlight=ufs)
[https://elinux.org/images/6/64/Introduction_to_UFS.pdf](https://elinux.org/images/6/64/Introduction_to_UFS.pdf)