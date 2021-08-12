---
title: Linux开启ssh root登陆&无密码登陆权限[转]
tags: Linux ssh root 小技巧
---

参考https://www.cnblogs.com/toughlife/p/5633510.html

## 开启ssh root登陆权限

修改sshd_config：

```shell
vim /etc/ssh/sshd_config
```

将PermitRootLogin参数改为yes

之后重启服务：

```shell
service sshd restart
```

## 允许无密码登陆

文件同上，将PermitEmptyPasswords参数改为yes，之后重启服务