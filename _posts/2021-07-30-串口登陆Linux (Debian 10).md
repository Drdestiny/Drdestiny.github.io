---
layout: article
title: Ubuntu 16.04 LTS下安装MATLAB R2016b
lang: zh-Hans
show_date: true
show_tags: true
author: Drdestiny
---

### 太长不看

- 用minicom -s进入configuration页面，选择Serial port setup，将Serial Device改为你选定的串口设备，我这里选择/dev/ttyS1
- 修改/etc/default/grub，在文件末尾加上这几行：

```shell
# 9600为波特率，n8表示数据位为8bit
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS1,9600n8"
GRUB_TERMINAL=serial
# parity：是否奇偶校验；stop：停止位位数
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```

- 刷新grub，命令行输入：

```shell
update-grub
```

- 创建/etc/init.d/ttyS1.conf（第一步选定哪个串口设备就填哪个），加上这几行

```shell
start on stopped rc RUNLEVEL=[2345]
stop on runlevel [!2345]

respawn
exec /sbin/getty -L 9600 ttyS1 vt100
```

- 串口另一端打开，重启Linux，等待串口输出

*reference:*
*1. 使用串口登录Ubuntu http://iqotom.com/?p=146*
*2. 虚拟机串口配置及其导出到主机pts和console.log https://www.hanbaoying.com/2017/04/07/config-vm-serial-and-rediect-to-pts.html*

---

### 探索过程

一开始在网上搜索串口登陆的教程，都是修改/etc/inittab，但是Debian 10并没有这个文件。

这里还是把关于修改这个文件的过程贴上来：

- 修改/etc/inittab文件，添加一行：

```shell
s1:2345:respawn:/sbin/agetty -L 9600 ttyS1 vt100
```

（关于最后一个冒号之后的命令，参考agetty的手册——man 8 agetty）

- 修改/etc/securetty，添加一行`ttyS1`，使**root用户**也可以通过串口登陆（**未经过验证**）
- 重启

#### 关于/etc/inittab

- 用init或者shutdown命令启动系统时，init守护进程从/etc/initab中读取信息来启动进程，该文件决定了init进程的三个重要因素:

  - init进程将重新启动的项

  - 在终止时要启动、监视和重新启动的进程
  - 系统进入一个新的运行级别时执行的操作

  每一项都是如下格式：

```shell
id:runlevel:action:process
```

**id**：每项的标识符，不能重复，可以自己取一个id

**runlevel**：该项action在哪个运行级别下运行，该段可以定义多个运行级别，每个级别之间不用分隔符，如2345，如果为空，表示在所有运行级别运行。Linux的运行级别有：

```
0：关机
1：单用户模式，这个模式下用户登陆不需要密码
2：多用户模式，NFS服务不开启
3：命令行模式
4：该模式保留
5：图形用户模式
6：重启系统
```

**action**：该项对应的process在一定条件下执行的动作，如：

- respawn：该项对应的process终止后立刻启动一个新的
- sysinit：系统初始化，只有系统开机或重新启动的时候，这个process才会被执行一次
- ...

*reference:* 

*1. /etc/initab文件详解  https://blog.51cto.com/leejia/788895*

#### init.d目录

实际上教程中创建ttyS1.conf的路径是/etc/init/ttyS1/ttyS1.conf，但是Debian中没有/etc/init目录，只有/etc/init.d目录，关于init.d目录，请参考[linux中init.d文件夹的说明](https://www.huaweicloud.com/articles/425a3c915a3310b1f699f91fcc4bd001.html)。当然文章也有不对的地方，比如开头说到/etc/init.d是/etc/rc.d/init.d的软链接，但是我在Debian 10中发现它就是一个目录，所以仅供参考。
