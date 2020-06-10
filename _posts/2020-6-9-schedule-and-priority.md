---
layout: post
title:  "linux 进程调度策略和优先级"
categories: process
tags: rt, nice, renice, chrt, SCHED_OTHER, SCHED_FIFO, SCHED_RR, SCHED_DEADLINE
author: jgsun
---


* content
{:toc}

# 1. Overview
linux进程有哪些调度策略？如何设置进程调度策略？Linux进程优先级如何划分的？如何设置进程优先级？top命令显示的PR和NI是什么关系？
















# 2. 进程调度策略
Linux调度策略分非实时和实时两大类([Scheduling - Policy and Priority](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/sched_policy_prio/start))

![image](/images/posts/process/police/schedule_policies.png)


###  2.2.1 最早截止时间优先EDF（Earliest DeadlineFirst）算法
最早截止时间优先EDF（Earliest DeadlineFirst）算法是非常著名的实时调度算法之一。在每一个新的就绪状态，调度器都是从那些已就绪但还没有完全处理完毕的任务中选择最早截止时间的任务，并将执行该任务所需的资源分配给它。在有新任务到来时，调度器必须立即计算EDF，排出新的定序，即正在运行的任务被剥夺，并且按照新任务的截止时间决定是否调度该新任务。如果新任务的最后期限早于被中断的当前任务，就立即处理新任务。按照EDF算法，被中断任务的处理将在稍后继续进行。

[A more detailed SCHED_DEADLINE explanation was published on LWN.net](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/sched_policy_prio/sched_deadline)

# 3. 进程优先级
进程优先级也分非实时和实时。
## 3.1 设置进程优先级
对非实时和实时进程，用户态设置进程优先级的命令分别是chrt和nice。两个命令的系统调用如下图所示：
![image](/images/posts/process/police/chrt-nice.png)

|type|command|范围|
|-----|----|----|
|非实时|nice -n 5 ./a.out|-20~19，数字越小优先级越高|
|非实时|renice 10 18625| 调整已运行进程优先级|
|实时|chrt -f 50 ./a.out |1~99，数字越大优先级越高|

* 用户通过chrt设置实时进程的优先级范围是1~99
  如果设置为0，chrt报错：
```
~ # chrt -p 0 1035
pid 1035's current scheduling policy: SCHED_RR
pid 1035's current scheduling priority: 1
chrt: number 0 is not in 1..99 range
```

chrt源码在util-linux软件包，下载地址：[http://www.kernel.org/pub/linux/utils/util-linux/](http://www.kernel.org/pub/linux/utils/util-linux/);

nice源码在coreutils软件包，下载地址：[https://www.gnu.org/software/coreutils/](https://www.gnu.org/software/coreutils/)；

嵌入式系统多用busybox软件包简化版的chrt和nice命令。

chrt可设置实时进程调度策略，加-p pid可改变当前已运行进程的优先级和调度策略。

nice命令设置已经运行的非实时进程的优先级，比如要使得top命令运行时候的优先级是5而不是默认的0，则可以使用nice -n 5 top，来使得top命令运行在5的优先级别。

如果top命令已经在运行，则有两个办法可以动态的调整进程的级别。

可以在top中输入r命令，然后按照提 示输入top命令对应的进程号，再按照提示输入要调整到哪个级别（嵌入式系统不支持）。

另一个方法是使用renice命令，此 命令使用也很简单，可以调整单个进程，一个用户或者一个组的所有进程的优先级。示例如下：

`#renice +10 -u jgsun`此命令把jgsun用户的所有进程的优先级全部调为10，包括新创建的和已经在运行的jgsun用户的所有进程。此处的+10并不是表 示在现有级别上再往上调整10个级别，而是调整到正10的级别，所以多次运行此命令，进程的优先级不会再发生变化，将一直停留在+10级别。

`#renice 10 18625 `将PID为18625的进程优先级调整为10
注意：如果不是root权限,则侄只能降调度优先级而不能提高，即使是自己用户的进程，自己把它调高了后，优先级也不能再被调会原来的值了，除非使用root用户来调回去。

系统重启后，对进程优先级的调整全部失效，所有进程的调度回到默认的初始级别。
chrt命令示例：
```
~ # chrt -p 16513
pid 16513's current scheduling policy: SCHED_RR
pid 16513's current scheduling priority: 30
~ # chrt -p 31 16513
pid 16513's current scheduling policy: SCHED_RR
pid 16513's current scheduling priority: 30
pid 16513's new scheduling policy: SCHED_RR
pid 16513's new scheduling priority: 31
~ # chrt -p -f 31 16513
pid 16513's current scheduling policy: SCHED_RR
pid 16513's current scheduling priority: 31
pid 16513's new scheduling policy: SCHED_FIFO
pid 16513's new scheduling priority: 31
```
## 3.2 用户命令显示进程优先级
top和ps命令都可显示进程优先级，这两个命令都来自procps软件包，可从 [https://gitlab.com/procps-ng/procps.git](https://gitlab.com/procps-ng/procps.git) 下载源码。
### 3.2.1 top命令
其中PR和NI两列表示进程优先级，分别表示用户视角优先级和nice值，可以看出两者 相差20
![image](/images/posts/process/police/top.png)

top命令显示的进程优先级如下图所示，数字越大优先级越低。
实时进程优先级是负值，范围在-100和-2之间，其中最高优先级-100显示为RT；
非实时进程优先级范围是0到39，对应用户设置的nice值是-20到19。
![image](/images/posts/process/police/top-prio.png)

## 3.2.1 ps命令显示的进程优先级
下图是ubuntu ps -elf的截图，其中PRI和NI表示进程优先级，两者相差80？？知乎上面一篇文章解释这个疑问：[ps 命令的 PRI 值和 task_struct 的 prio 值的关系是怎么样？](https://www.zhihu.com/question/272086181)
![image](/images/posts/process/police/ps.png)

## 3.3 内核表示进程优先级
内核struct task_struct结构体prio成员表示进程优先级，取值范围在0-139，
## 3.4 各种优先级的相互转换
从上面可以看出各种linux进程的优先级是很混乱的，命令设置的优先级和top命令显示的优先级不同，它们又不同于内核变量表示的优先级，ps命令显示的优先级更加难以理解，比如:
```
jgsun@VirtualBox:~$ ps -o pri,opri,intpri,priority,ni,pcpu,pid,comm       
PRI PRI PRI PRI NI %CPU PID COMMAND
 19 80 80 20 0 0.1 26508 bash
 19 80 80 20 0 0.0 26520 ps
```

内核头文件[linux/sched/prio.h](https://elixir.bootlin.com/linux/latest/source/include/linux/sched/prio.h)定义了关于进程优先级的宏，其中用于转换关系是MAX_RT_PRIO和DEFAULT_PRIO：

|MAX_RT_PRIO|100|
|---|---|
|DEFAULT_PRIO |120|

用户命令设置的优先级，top命令显示的优先级和内核变量表示的优先级转换关系表：

|用户命令设置的优先级|top命令显示的优先级|内核变量 p->prio表示的优先级|
|---|---|---|
|范围实时1~99(数字大优先级高)，非实时-20~19|范围RT,-99~-2，0~39|范围0~139|
|rt|-1 - rt | MAX_RT_PRIO - 1 - rt|
|nice|nice + 20|nice + DEFAULT_PRIO |
|30|-31|69|
|0|20|120|

内核函数task_prio用于将内核变量 p->prio表示的优先级转换为top命令显示的优先级：
```
<linux/kernel/sched/core.c>
int task_prio(const struct task_struct *p)
{
 return p->prio - MAX_RT_PRIO;
}
```
内核nice和内核变量 p->prio互相转换的宏定义：
```
<linux/include/linux/sched/prio.h>
#define NICE_TO_PRIO(nice)	((nice) + DEFAULT_PRIO)
#define PRIO_TO_NICE(prio)	((prio) - DEFAULT_PRIO)
```

# 参考资料
* [Scheduling - Policy and Priority](https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/sched_policy_prio/start)
* [宋宝华： 关于Linux进程优先级数字混乱的彻底澄清](https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652665181&idx=2&sn=ab1d35dfa913fedbc453f6efea4eab6d&chksm=810f33c0b678bad69be45d1a4aad1695f1670af5f20711d6befb5e3d053b037f404d8267f80d&scene=21#wechat_redirect)
* [混乱的Linux内核实时线程优先级 ](https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652667392&idx=1&sn=f78c876d2e8f40d67eeb3c9366da9588&chksm=810f389db678b18b21684a2fcee5ddb13b0f0ae23f74d6afe418317a09f094bdf21b779de3f3&scene=126&sessionid=1591496879&key=36b538e0ffa7935cc08b69ad7d7be294d9a6c106ee70650262839e34b4d0df53f5ae57ee0e78a0cc77bcc16253d668e0acb0d1fa3423c2ba779a037850dc472aed5fc7ad36b406a8bac7e4a3342f7637&ascene=1&uin=MTA4ODcwMzIyMQ%3D%3D&devicetype=Windows+10+x64&version=62090070&lang=en&exportkey=AYtTG%2B4pmHHtnoiwZnDTkuE%3D&pass_ticket=BO8015qHHAs76MXOIsfm5IB5tvHnJRRmSlPhNAohpuig3o2dqLPRjatZWZ01MxVK)