---
title: 始 - 从零开始配置自己的服务器 虚拟机篇
---
# 前言
这个服务器运行MC服务器，所以选用了单核强的cpu。
服务大部分运行于虚拟机中以保证安全，同时也方便快照备份（我才不会告诉你是怕我手残玩炸了host机环境呢）

# 硬件配置

| CPU | 内存 | 存储 | 网络 |
| ----- | ----- | ----- | ----- |
| 13600KF | DDR4 32G | 1T SSD | 有公网ip的家用宽带 |

# 步骤

## 安装系统
使用Ubuntu22.04 Server，经过BalenaEtcher烧录进U盘后无脑安装
## 设置KVM
### 安装
``` bash
sudo apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils
```
### 启用&自启
``` bash
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

### 配置虚拟网络
``` bash
sudo nano /etc/sysctl.conf
# 取消注释 "net.ipv4.ip_forward = 1"，用来允许把外界的数据包转发给虚拟机。

sudo virsh net-start default
```

### 安装虚拟机
``` bash
#先把Debian Server镜像下载至/var/lib/libvirt/images/ 或者 /tmp/xxxx/
sudo virt-install --name server_network_stuff --memory 4096 --vcpus 4 --disk size=16 --location /tmp/linux_installer_image/debian-11.6.0-amd64-DVD-1.iso --os-variant debian11 --graphics none --extra-args 'console=ttyS0,115200n8 --- console=ttyS0,115200n8'

# 或者从网络镜像安装Debian
sudo virt-install --name server_network_stuff --memory 4096 --vcpus 4 --disk size=16 --location http://ftp.us.debian.org/debian/dists/stable/main/installer-amd64/ --os-variant debian11 --graphics none --extra-args 'console=ttyS0,115200n8 --- console=ttyS0,115200n8'

# 如果是Ubuntu，则需要指定kernel，否则无法启动
sudo virt-install --name server_blog --memory 4096 --vcpus 4 --disk size=16 --location /var/lib/libvirt/images/ubuntu-20.04.5-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd --os-variant ubuntu20.04 --graphics none --extra-args 'console=ttyS0,115200n8 --- console=ttyS0,115200n8'
```

### 配置虚拟机，设置KVM中虚拟机在局域网中的IP，并绑定核心关联性

#### 设置自启
```bash
virsh autostart server_network_stuff
```

#### 编辑虚拟机配置文件
```bash
virsh edit server_network_stuff
```
1. 修改"<vcpu placement='static'>4</vcpu>" 为 "<vcpu placement='static' cpuset='12-15'>4</vcpu>"，绑定CPU核心。这里我希望这个服务跑在13600K的小核上面，不会去占用大核的性能。
2. 获取MAC地址，在
        <interface type='network'>
            <mac address='52:54:00:a7:26:ca'/>

#### 编辑虚拟网络配置文件
```bash
virsh net-edit default
```
在
` <dhcp>  <\dhcp> `
当中加入一行
	`<host mac='52:54:00:a7:26:ca' name='server_network_stuff' ip='192.168.122.20'/>`
即可绑定静态ip地址；
将
	`<forward mode='bridge'/>`
改为
	`<forward mode='route'/>`

#### 重启虚拟网络
```bash
virsh net-destroy default
virsh net-start default
```
由于奇怪的原因，这样重启虚拟网络之后虚拟机都连不上网了。我的解决办法是重启主机；网上也有一些脚本可以帮助解决这个问题。
```bash
sudo reboot
```


## 配置网络 端口映射

#### 禁用UFW
```bash
sudo ufw disable
```

#### 安装 启用firewalld
有时候会遇到firewalld自动把22端口屏蔽的情况，因此推荐先用FRP将SSH端口转发，以防万一。
```bash
# ...frp略
sudo apt install firewalld
sudo systemctl enable firewalld
sudo systemctl start firewalld
```
#### 修改nftable至iptable
不知道为什么nftable不兼容virsh，所以就改成iptable了。有可能不用改。
```bash
sudo nano /etc/firewalld/firewalld.conf
# 然后把nftables改成iptables
```
#### 添加端口&端口映射

```bash
sudo firewall-cmd --add-masquerade --permanent
    # to allow multiple ports forwarding into one

sudo firewall-cmd --add-forward-port=port=14022:proto=tcp:toport=22:toaddr=192.168.122.20 --permanent --zone=public
    # 端口映射
sudo firewall-cmd --add-port=xx/tcp

```
#### 结束
至此就可以在虚拟机中运行服务了。


