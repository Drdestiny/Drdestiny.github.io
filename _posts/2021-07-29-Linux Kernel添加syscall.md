---
title: Linux Kernel添加syscall
tags: Linux syscall
---

部门的培训课程有一个实验，要求是这样的：**在内核中增加一个系统调用，获取系统启动以来，经过了多少 tick（jiffies），根据内核配置换算成毫秒**。

环境为**ARM64** + **Linux 5.14-rc4**（如果之前是按照教程用git clone下载了最新版本的内核代码，那么版本就是一样的，无须担心；你也可以查看内核代码顶层目录下的Makefile的前几行判断内核版本）。不同的架构以及内核版本，过程不一定完全相同，请注意。

本文分成三个部分，分别对应该实验的三个步骤：**添加系统调用**、**编译与文件传输**、**patch提交**。

## 添加系统调用

首先要明确，添加一个系统调用可以分为三个部分：**添加系统调用号**、**添加函数声明**、**函数实现**。

### 添加系统调用号

内核代码中有关于添加系统调用的文档（./Documentation/process/adding-syscalls.rst），其中Generic System Call Implementation一节有提到（下文中的xyzzy是添加的系统调用名）：

> Some architectures (e.g. x86) have their own architecture-specific syscall tables, but several other architectures share a generic syscall table. Add your new syscall to the generic list by adding an entry to the list in `include/uapi/asm-generic/unistd.h`::
```c
#define __NR_xyzzy 292
__SYSCALL(__NR_xyzzy, sys_xyzzy)
```
> Also update the __NR_syscalls count to reflect the additional system call......

所以修改**include/uapi/asm-generic/unistd.h**（其实我也不知道ARM64是否属于上文提到的“other architectures”，但是这么做管用:)。如果有了解这方面的小伙伴，请指教！）：

一直往下翻，翻到最大的系统调用号：

```c
...
#define __NR_landlock_restrict_self 446
__SYSCALL(__NR_landlock_restrict_self, sys_landlock_restrict_self)

#ifdef __ARCH_WANT_MEMFD_SECRET
#define __NR_memfd_secret 447
__SYSCALL(__NR_memfd_secret, sys_memfd_secret)
#endif

#undef __NR_syscalls
#define __NR_syscalls 448
...
```

当前最大的系统调用号是447，对应`sys_memfd_secret`这个系统调用。而`__NR_syscalls`比最大的系统调用号大1，可以看做是系统调用的数量。我们要添加系统调用号，也要根据这样的格式，然后更新`__NR_syscalls`，让它比我们添加的系统调用号大1，具体过程就不再赘述。

### 函数声明

之后的步骤就与硬件平台无关了。系统调用的声明都在./include/linux/syscalls.h中添加，随便找个位置：

```c
asmlinkage long sys_getms_sinceboot(unsigned int __user *msecs);
```

系统调用的格式都是`sys_xxxx`，`xxxx`为该系统调用的名字，这里`getms_sinceboot`就是我们新增系统调用的名字。

函数签名前面有`asmlinkage`字段，它是一个gcc标签，代表函数读取的参数来自于栈，而非寄存器。

参数中的__user字段表示该参数来自用户空间。执行系统调用的时候系统处于内核空间，而内核空间和用户空间所处的内存地址是不一样的，需要通过`copy_to_user()`将内核空间的数据拷贝到用户空间。

ps：只有参数是指针类型的时候需要加`__user`，因为指针类型存储的是地址；

pps：我建议用指针获得返回地址，在需要两个或者更多返回值的情况下，指针的优势就凸显出来了。

这里用`msecs`这个参数来获取最终结果，我们在用户空间写测试程序的时候，将`msecs`指针传进系统调用，系统调用经过一系列操作，将数据通过`copy_to_user()`传递给`msecs`，最后将`msecs`打印出来即可得到最终结果。

### 函数实现

函数实现会根据函数的功能分类，分布在不同的.c文件里。在`./include/linux/syscalls.h`中，函数实现在同一个.c文件的函数声明是放在一起的，这里我们随便挑一个文件`./kernel/signal.c`（自己新建一个.c文件添加函数实现也行，但是要把这个文件放进Makefile里，有些麻烦），添加这样一段代码：

```c
/*
 * sys_getms_sinceboot - get how many milliseconds have elapsed since boot
 * @msecs: return the value of msecs corresponding to jiffies
 */
#include <linux/jiffies.h>
SYSCALL_DEFINE1(getms_sinceboot, unsigned int __user *, msecs)
{
        unsigned long kjiffies = get_jiffies_64() - INITIAL_JIFFIES;
        unsigned int kmsecs = jiffies_to_msecs(kjiffies);
        copy_to_user(msecs, &kmsecs, sizeof(kmsecs));
        return 0;
}
```

这些实现都是通过`/include/linux/syscall.h`中的**SYSCALL_DEFINEx**（x可以取0-6）宏来定义的，x为该系统调用的参数数量，`SYSCALL_DEFINEx`的第一个参数为系统调用名，后面分别是参数类型、参数名。

获取jiffies可以用`linux/jiffies.h`提供的接口`get_jiffies_64()`（`jiffies_64`是64位，关于jiffies请各位自行搜索）。

因为jiffies的初始值并非0，所以需要减去一个初始值，`linux/jiffies.h`中提供了这个初始值：`INITIAL_JIFFIES`。



完成以上步骤后，编译即可。

## 编译与文件传输

内核的编译这里就不再赘述，参考：Qemu虚拟机运行ARM64架构的Linux。

这里说一下测试程序的编译——之前为了编译内核，已经安装过了。用法和gcc相同：

```shell
aarch64-linux-gnu-gcc file.c -o file
```

接下来将文件传输到qemu虚拟机中，首先开启qemu虚拟机。

用ssh连接到Debian，输入命令：

```shell
sftp -P 10023 root@localhost
```

在sftp连接完成之后，输入任何命令都相当于在qemu虚拟机中执行，如果你想在Debian执行命令，就在命令前面加上l，如`lls`。

然后用`put`命令将编译好的可执行文件传到qemu虚拟机中（注意Debian当前所在路径），之后执行即可。

## patch提交

首先确定你的内核代码里是否有.git目录（用ls -a查看），如果没有，在对内核进行修改之前，在内核代码目录下git init也可以。然后用git branch查看当前所在分支，一般情况下只有master分支。我们先创建一个自己的分支，方便之后生成patch，也方便之后其他功能的开发：

```shell
git branch <your_branch_name>
```

就生成了一个新分支，然后要切换到这个分支：

```shell
git checkout <your_branch_name>
```

对内核的修改结束之后，cd到顶层目录，先将代码提交，步骤如下：

```shell
git add .                                            # .表示当前目录下的所有文件，这个命令是将当前目录下所有文件的修改放到暂存区
git commit -m "<your_commit_message>"                # 将暂存区的所有修改提交到版本库
git format-patch HEAD~1                              # 生成patch，HEAD始终指向当前最新的commit，HEAD~x表示以HEAD之前第x个commit为基准，将之后的所有修改生成patch
```

随后就得到了patch文件。