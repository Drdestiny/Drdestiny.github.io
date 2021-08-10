---
title: Linux Kernel添加syscall
tags: Linux syscall
---

部门的培训课程有一个实验，要求是这样的：请在内核中增加一个系统调用，获取系统启动以来，经过了多少 tick（jiffies），根据内核配置换算成毫秒。

本文的配置为**ARM64**+**Linux 4.19**。不同的架构以及内核版本，过程不完全相同，请注意。

1. 在./arch/arm64/include/asm/unistd32.h查看下一个可用的系统调用号是多少：

```c
...
#define __NR_pkey_alloc 395
__SYSCALL(__NR_pkey_alloc, sys_pkey_alloc)
#define __NR_pkey_free 396
__SYSCALL(__NR_pkey_free, sys_pkey_free)
#define __NR_statx 397
__SYSCALL(__NR_statx, sys_statx)
#define __NR_rseq 398
__SYSCALL(__NR_rseq, sys_rseq)
```

所以大于398的系统调用号都可以用。

2. 在./include/uapi/asm-generic/unistd.h中增加系统调用号：

```c
...
#define __NR_statx 291
__SYSCALL(__NR_statx,     sys_statx)
#define __NR_io_pgetevents 292
__SC_COMP(__NR_io_pgetevents, sys_io_pgetevents, compat_sys_io_pgetevents)
#define __NR_rseq 293
__SYSCALL(__NR_rseq, sys_rseq)
/* 增加一个名为getjiffies的系统调用，系统调用号399 */
#define __NR_getjiffies 399
__SYSCALL(__NR_getjiffies, sys_getjiffies)

#undef __NR_syscalls
#define __NR_syscalls 400
/*
 * #define __NR_syscalls 294
 * __NR_syscalls比最大的系统调用号大1。原先的值为294，因为我们添加了一个调用
 * 号为399的系统调用，所以将_NR_syscalls改成400
 */
```

3. 在./include/linux/syscalls.h中增加函数声明：

```c
asmlinkage long sys_getjiffies(unsigned long __user *jiffies, unsigned int __user *msecs);
```

include/linux/syscall.h中是各种系统调用的声明，格式都是sys_xxxx（xxxx为该系统调用在用户空间的名字，如kill）。函数原型前面有**asmlinkage**字段，它是gcc标签，代表函数读取的参数**来自于栈中，而非寄存器**（因为在执行服务例程之前系统已经将通过寄存器传递过来的参数值压入内核堆栈了）。

这些声明会在不同的.c文件中实现，如/kernel/signal.c中的sys_kill()：

```c
/**
 *  sys_kill - send a signal to a process
 *  @pid: the PID of the process
 *  @sig: signal to be sent
 */
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct siginfo info;

	clear_siginfo(&info);
	info.si_signo = sig;
	info.si_errno = 0;
	info.si_code = SI_USER;
	info.si_pid = task_tgid_vnr(current);
	info.si_uid = from_kuid_munged(current_user_ns(), current_uid());

	return kill_something_info(sig, &info, pid);
}
```

这些实现都是通过定义在/include/linux/syscall.h中的**SYSCALL_DEFINEx**（x可以取0-6）宏来定义的，x为该系统调用的参数数量，SYSCALL_DEFINEx的第一个参数为系统调用名，后面分别是参数类型、参数名，上图中的函数签名翻译一下就是：

```c
long sys_kill(pid_t pid, int sig)
```

4. 最后增加实现，这里我们选择./kernel/signal.c：

```c
/*
 * sys_getjiffies 
 * @jiffies: return the value of jiffies
 * @msecs: return the value of msecs corresponding to jiffies
 */
#include <linux/jiffies.h>
SYSCALL_DEFINE2(getjiffies, unsigned long __user *, jiffies, unsigned int __user *, msecs)
{
	unsigned long kjiffies = get_jiffies_64() - INITIAL_JIFFIES;
	unsigned int kmsecs = jiffies_to_msecs(kjiffies);
	copy_to_user(jiffies, &kjiffies, sizeof(kjiffies));
	copy_to_user(msecs, &kmsecs, sizeof(kmsecs));
	return 0;
}
```

然后编译内核，就成功地增加了我们自己定制的系统调用，接下来在用户态测试一下：

```c
#include <stdio.h>
#include <sys/syscall.h> /* 该头文件提供了syscall()函数 */

int main()
{
    unsigned long jiffies;
    unsigned int msecs;
    syscall(399, &jiffies, &msecs);
    printf("%u msecs (%lu jiffies) have elapsed since boot.", msecs, jiffies);
    return 0;
}
```

成功输出。自此，该实验成功完成。