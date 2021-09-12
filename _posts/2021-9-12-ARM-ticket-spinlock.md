---
layout: post
title: "ARM ticket spinlock 的实现"
category: process
tags: ARM ARM64 ticket-spinlock
author: jgsun
---

* content
{:toc}

# Overview
这是一个有点过时的题目，因为 spin_lock 已经演进到性能更好的 qsbinlock：X86 enabled qspinlock support in kernel  v4.2. [Locking changes for v4.2](http://lkml.iu.edu/hypermail/linux/kernel/1506.2/04205.html), Arm64 changed the default spinlock to qspinlock after the kernel 4.16rc6. 但是研究 ARM ticket spinlock 的实现可以学习 spin_lock 的进化，学习ARM atomic 操作和 ARM inline 汇编，所以记录下来方便自己以后查阅。
![image](/images/posts/process/lock/ticket-spinlock.png)




















# ARM V6
Arm 的 arch_spin_lock 还是才有的 ticket spinlock， 源码位于 [5.14.2: arch/arm/include/asm/spinlock.h](https://elixir.bootlin.com/linux/v5.14.2/source/arch/arm/include/asm/spinlock.h#L56)
## 加锁 arch_spin_lock
加锁时先将 next 保存到本地变量 lockval.tickets.next，然后将 next +1， owne保持不变。
（1）初始化 lock.tickets.next 和lock.tickets.owner 都等于0；
（2）C0第一个加锁的立即获取锁，同时next +1， 所以这时候 lock.tickets.next =1，lock.tickets.owner = 0；
（3）C1试图获取锁，lockval.tickets.next =1，而 lockval.tickets.owner = 0；所以 wfe 进入low-power模式
（4）C0释放锁，owner++， lock.tickets.owner = 1，发 SEV 唤醒 C1
（5）C1 读取 lock.tickets.owner =1 和 lock.tickets.next 相等从而获取锁。
```
static inline void arch_spin_lock(arch_spinlock_t *lock)
{
    unsigned long tmp;
    u32 newval;
    arch_spinlock_t lockval;

    prefetchw(&lock->slock);    // 软件预读 &lock->slock，提前读到 cache
  // [jgsun] LL/SC(Load-Link/Store-Conditional)，实现原子操作
    __asm__ __volatile__(
"1:    ldrex    %0, [%3]\n"         // 将 lock->slock 加载到 lockval, monitor 标记为 exclusive
"    add    %1, %0, %4\n"         // 将 lockval + 1, 保存到 newval
"    strex    %2, %1, [%3]\n"     // 将 newval store 到 lock->slock，如 monitor 仍为 exclusive，
                                       // 则 store 成功并置 tmp 为0，否则 store 不成功置tmp为非0
"    teq    %2, #0\n"               // 判断tmp是否为0
"    bne    1b"                       // 如果非0，返回1继续
    : "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
    : "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
    : "cc"); // The "cc" clobber indicates that the assembler code modifies the flags register. 

    while (lockval.tickets.next != lockval.tickets.owner) {
        wfe(); // 如果 lockval 中的 next 不等了 owner
        lockval.tickets.owner = READ_ONCE(lock->tickets.owner);
    }

    smp_mb();
}
```
### 结构体定义 arch_spinlock_t 
arch_spinlock_t 的定义在文件[arch/arm/include/asm/spinlock_types.h](https://elixir.bootlin.com/linux/latest/source/arch/arm/include/asm/spinlock_types.h#L24)
```
#define TICKET_SHIFT    16

typedef struct {
    union {
        u32 slock;
        struct __raw_tickets {
#ifdef __ARMEB__
            u16 next;
            u16 owner;
#else
            u16 owner;
            u16 next;
#endif
        } tickets;
    };
} arch_spinlock_t;
```
## 解锁 arch_spin_unlock
```
static inline void arch_spin_unlock(arch_spinlock_t *lock)
{
    smp_mb(); //[jgsun]dsb memory barrier
    lock->tickets.owner++; // [jgsun] owner +1 
    dsb_sev(); //[jgsun] 先调用dsb memory barrier，再调用 __asm__(SEV) 唤醒因为 WFE 指令进入 low-power 模式其他处理器
}
```
# ARM64
ARM64 在3.13 实现ticket lock，Arm64 changed the default spinlock to qspinlock after the kernel 4.16rc6, 4.19 Arm64 ticket lock 从内核消失。
最新版本 ARM64 的 arch_spin_lock 已经定义为 queued_spin_lock。
[v5.14.2: arch/arm64/include/asm/spinlock.h](https://elixir.bootlin.com/linux/v5.14.2/source/arch/arm64/include/asm/spinlock.h) `#include <asm/qspinlock.h>`  
[v5.14.2: include/asm-generic/qspinlock.h](https://elixir.bootlin.com/linux/v5.14.2/source/include/asm-generic/qspinlock.h)  `#define arch_spin_lock(l)        queued_spin_lock(l)`
所以 ARM64 ticket lock的实现在 4.18.20 版本，4.19 以后的内核删除了ARM64 ticket lock. [4.18.20: arch/arm64/include/asm/spinlock.h](https://elixir.bootlin.com/linux/v4.18.20/source/arch/arm64/include/asm/spinlock.h)
## 加锁 arch_spin_lock
ARMv8.1开始，ARM推出了用于原子操作的LSE(Large System Extension)指令集扩展，比LL/SC的实现方式简洁了很多，arch_spin_lock 全部使用汇编实现。
```
static inline void arch_spin_lock(arch_spinlock_t *lock)
{
    unsigned int tmp;
    arch_spinlock_t lockval, newval;

    asm volatile(
    /* Atomically increment the next ticket. */
    ARM64_LSE_ATOMIC_INSN(
    /* LL/SC */
"    prfm    pstl1strm, %3\n"
"1:    ldaxr    %w0, %3\n"
"    add    %w1, %w0, %w5\n"
"    stxr    %w2, %w1, %3\n"
"    cbnz    %w2, 1b\n",
    /* LSE atomics */
"    mov    %w2, %w5\n"
"    ldadda    %w2, %w0, %3\n"
    __nops(3)
    )

    /* Did we get the lock? */
"    eor    %w1, %w0, %w0, ror #16\n"
"    cbz    %w1, 3f\n"
    /*
     * No: spin on the owner. Send a local event to avoid missing an
     * unlock before the exclusive load.
     */
"    sevl\n"
"2:    wfe\n"
"    ldaxrh    %w2, %4\n"
"    eor    %w1, %w2, %w0, lsr #16\n"
"    cbnz    %w1, 2b\n"
    /* We got the lock. Critical section starts here. */
"3:"
    : "=&r" (lockval), "=&r" (newval), "=&r" (tmp), "+Q" (*lock)
    : "Q" (lock->owner), "I" (1 << TICKET_SHIFT) //Q: A memory address which uses a single base register with no offset
    : "memory");
}
```
###  结构体定义 arch_spinlock_t
```
#define TICKET_SHIFT    16

typedef struct {
#ifdef __AARCH64EB__
    u16 next;
    u16 owner;
#else
    u16 owner;
    u16 next;
#endif
} __aligned(4) arch_spinlock_t;
```
## 解锁 arch_spin_unlock
```
static inline void arch_spin_unlock(arch_spinlock_t *lock)
{
    unsigned long tmp;

    asm volatile(ARM64_LSE_ATOMIC_INSN(
    /* LL/SC */
    "    ldrh    %w1, %0\n"
    "    add    %w1, %w1, #1\n"
    "    stlrh    %w1, %0",
    /* LSE atomics */
    "    mov    %w1, #1\n"
    "    staddlh    %w1, %0\n"
    __nops(1))
    : "=Q" (lock->owner), "=&r" (tmp)
    :
    : "memory");
}
```
# 参考资料
* [Arm Architecture Reference Manual Armv8, for A-profile architecture](https://developer.arm.com/documentation/ddi0487/gb/)  B2.9 Synchronization and semaphores
* [ARM Synchronization Primitives Development Article](https://developer.arm.com/documentation/dht0008/a/)
* [ARMv8-A synchronization primitives](https://developer.arm.com/documentation/100934/0100)
* [6.47 How to Use Inline Assembly Language in C Code](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html#Using-Assembly-Language-with-C)
* [6.47.3 Constraints for asm Operands](https://gcc.gnu.org/onlinedocs/gcc/Constraints.html#Constraints)
* [What happens in the assembly output when we add "cc" to clobber list](https://stackoverflow.com/questions/59656857/what-happens-in-the-assembly-output-when-we-add-cc-to-clobber-list)
* [ARM GCC 内嵌（inline）汇编手册](https://blog.csdn.net/lhf_tiger/article/details/32343851)
* [ARM GCC Inline Assembler Cookbook](http://www.ethernut.de/en/documents/arm-inline-asm.html)
* [ARM64基础4：在C语言中嵌入ARM64汇编代码](https://blog.csdn.net/luteresa/article/details/119327138)
* [ARM64基础11:GCC内嵌汇编补充](https://blog.csdn.net/luteresa/article/details/120140887)
* [对优化说不 - Linux中的Barrier](https://zhuanlan.zhihu.com/p/96001570)
* [读写一气呵成 - Linux中的原子操作](https://zhuanlan.zhihu.com/p/89299392)