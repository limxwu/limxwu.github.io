---
title: Debian关闭交换内存
date: 2023-10-18
---

临时关闭：`sudo swapoff -a `

永久关闭：`sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`

systemctl关闭：

```shell
systemctl --type swap --all

systemctl mask dev-xxx.swap
```