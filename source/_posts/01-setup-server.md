---
title: 始 - 从零开始配置自己的服务器
---
## 前言
这个服务器运行MC服务器，所以选用了单核强的cpu。
服务大部分运行于虚拟机中以保证安全，同时也方便快照备份（我才不会告诉你是怕我手残玩炸了host机环境呢）

## 硬件配置

| CPU | 内存 | 存储 | 网络 |
| ----- | ----- | ----- | ----- |
| 13600KF | DDR4 32G | 1T SSD | 有公网ip的家用宽带 |

## 步骤

### 安装系统
使用Ubuntu22.04 Server，经过BalenaEtcher烧录进U盘后无脑安装
### 配置
#### 1. 设置KVM:
安装：
``` bash
$ sudo apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils
```
启用：
``` bash
$ sudo systemctl enable libvirtd
$ sudo systemctl start libvirtd
```

配置虚拟网络：
``` bash
$ sudo nano /etc/sysctl.conf
# 取消注释 "net.ipv4.ip_forward = 1"，用来允许把外界的数据包转发给虚拟机。

$ sudo virsh net-start default
```

