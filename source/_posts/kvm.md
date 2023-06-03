---
title: Centos7.9虚拟化部署和使用
date: '2023-1-14 08:37:00'
tags: Kvm
categories: Linux运维
cover: 'https://data-static.netdun.net/Fomalhaut/img/default_cover_126.webp'
abbrlink: 9442
---
## 1、Centos7.9虚拟化kvm部署

### 1.1 实验环境

**1.检查宿主机是否支持虚拟化**

```Shell
[root@kvm ~]# egrep -o 'vmx | svm' /proc/cpuinfo | wc -l
```

如果显示数值是 0，则表示该 CPU 不支持虚拟化。

**2.虚拟机开启虚拟化**

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=Mzk0Y2U5MWUxZjZhZjI3NWU3Y2VmNjhkN2ZjZTA3YTBfc3hqOWdhVUV3VjVDSkFjVWNLaEU2Q3BZdTMyQWNvcktfVG9rZW46Ym94Y25iMVJ1ZnNIMTdaTUNnbDJoQzV1REFNXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

**3.关闭iptables 和 selinux**

```Shell
关闭 iptables 服务：
[root@kvm ~]# service iptables stop
关闭 selinux：
[root@kvm ~]# setenforce 0
[root@kvm ~]# vi /etc/selinux/config
SELINUX=disabled
```

### 1.2 安装和配置 kvm 环境

1.查看是否加载kvm

```Shell
[root@kvm ~]# lsmod | grep kvm
kvm_intel             188644  0 
kvm                   621480  1 kvm_intel
irqbypass              13503  1 kvm
```

2.安装kvm相关软件包

```Shell
yum -y install qemu-kvm python-virtinst libvirt libvirt-python virt-manager libguestfs-tools bridge-utils virt-install
```

3.开启kvm服务libvirt并设置开机自启

```Shell
[root@kvm ~]# systemctl start libvirtd
[root@kvm ~]# systemctl enable libvirtd
```

4.设置机器的存储

在/home目录下创建两个目录，一个存放系统镜像，一个做虚拟机的存储盘

```Shell
[root@kvm ~]# mkdir /home/iso
[root@kvm ~]# mkdir /home/images
```

上传镜像并将镜像放到/home/iso目录下

5.创建物理桥接设备

```Shell
[root@kvm ~]# cd /etc/sysconfig/network-scripts/
[root@kvm ~]# cp ifcfg-ens33 ifcfg-br0
[root@kvm ~]# vi ifcfg-ens33
DEVICE=ens33
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
BRIDGE=br0

[root@kvm ~]# vi ifcfg-br0
DEVICE=br0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=static
IPADDR=172.50.20.15
PREFIX=24
GATEWAY=172.50.20.1
DNS1=114.114.114.114
NAME=br0

# 重启网络
[root@kvm ~]# service network restart
```

桥接设备关联网卡

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=MTJiNGRhMTE0MDg5ZTdiY2M1MmZhNjE3ZGRkNDk4NzJfcEh2RExCeWwxdTBpUUxURzVFNDJ4ZkVoOERLb0JCdUZfVG9rZW46Ym94Y25UOUoxRkFGV05qRWVvcW1sOWlEY3JjXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

### 1.3 创建虚拟机

1.使用virt-manager命令打开虚拟机管理器

virt-manager

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=MGM2NjdkMmE2Mzg1MzhiZDkzMTE4MjNjZTgxYWNmMmNfeUdKVWZLQk1nWjJsZ1F5Tjd6NG5Pd3BpMGNpZElFeU9fVG9rZW46Ym94Y25FTkI5WmpQaWdXTEtGVlZ4V3lERUhoXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=N2QwNjkxYmU5MmYzYmVkNzIzZGRiOTY1YzAxMGYxOThfOTV4OXNqcU1JWGVMYW1pUWxBZnFtb0I3cm96bElrOVJfVG9rZW46Ym94Y244OVRyMEdSdzNwbm9NTXFBbWp4UlJoXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjBjNWVjOWMzODE4ZTYxYTM3MTg3ZDBiZGM5N2M5ZDlfV2ZCWjQ1eXFlV3hWTUdvaWQ4QXlTYzNBQ1lXbEh0TklfVG9rZW46Ym94Y25aSFFNUHVjcE4zUHdWbk13eEE4QkdlXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjhmM2RlNjM1MGU5ODZhNzU4YmJlZGFlMWMwNjgzNmJfZTQxcWFxYTBZeUROZU9HcWprbld5QnRIVlhYcW9lbm5fVG9rZW46Ym94Y240MjJ1TDBaNUhqdDg2Y2p4dVlBSzViXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDQwN2U2YjdjMjkyYTA4MmJjMTcwZjA0NTE0YjQ2ZGFfeVo4eXZvcGxjZFFmOW5qTnRCZ2lqcE5HUEhxOW5JOG5fVG9rZW46Ym94Y255aXBqREpXNmhQazZOUjVMQVcwenliXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGFhZDc2ODRhYzNkYjUzMzU1ZmJiYzA0MmVlNDk2MTJfb0hsQ1FRalJHbjN5OTNLaVQwSkVwdm9SSzhZQlpTQW1fVG9rZW46Ym94Y25yY2hlZHdmQk9ZWUhrR0NpR3VsMnRkXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDQwMGU0YzQ4MDU4MjdkMmJjYmQxMWY0ZDVhMzNlYjNfRTFIRUc4SEVueEJYRUh5RUpoTFYyREo0U3FKbVNJdWRfVG9rZW46Ym94Y25hZVBLZmNoRXpHUVRIYkNQMmJOdmhkXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=MjYyMWZlYWZmNGE1MTQxZjFjNmJhZTFjYTdlOTA4MjBfR3pPNTYwRklXOEVoOWN5SkNrZ3loSXZrS2JTM2VEdlpfVG9rZW46Ym94Y25CNHoyZ2FmRVo4emF3Q2pUQWdhVzdmXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTliNTk0YmZjNzY0MDRkN2RkZWNkNDQ1OWI1ODdiOThfS0ZzZ2RYTGF1blVSUUZLVVIzTFNwaXNIWUpFWXpnY0lfVG9rZW46Ym94Y25ocWF3MVRzRkd0QlAwZDhVOWJXbk5oXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

##  2、libvirtd服务配置文件参数含义

### 2.1 libvirtd配置文件

libvirtd.conf是libvirt的**守护进程libvirtd的配置文件**，被修改后需要让**libvirtd重新加载配置文件**（或**重启libvirtd**）才会生效。

在libvirtd.conf中配置了libvirtd启动时的许多设置，包括是否建立TCP、UNIX domain socket等连接方式及其最大连接数，以及这些连接的认证机制，设置libvirtd的日志级别等

qemu.conf是libvirt对**QEMU的驱动的配置文件**，包括VNC、SPICE等，以及连接它们时采用的权限认证方式的配置，也包括内存大页、SELinux、Cgroups等相关配置。

在qemu目录下存放的是**使用QEMU驱动的域的配置文件**。查看qemu目录如下：

其中包括了**两个域的XML配置文件**这是用virt-manager工具创建的两个域，默认会将其配置文件保存到/etc/libvirt/qemu/目录下。而其中的**networks目录**保存了**创建一个域时默认使用的网络配置**。

libvirtd是一个作为libvirt虚拟化管理系统中的**服务器端的守护程序**，要让某个节点能够利用libvirt进行管理（无论是**本地还是远程管理！**），都需要在这个节点上**运行libvirtd这个守护进程**，以便让其他**上层管理工具可以连接到该节点**，**libvirtd**负责**执行**其他管理工具发送给它的**虚拟化管理操作指令**。

而libvirt的**客户端工具**（包括**virsh**、**virt-manager**等）可以**连接**到**本地或远程的libvirtd进程**，以便**管理节点上的客户机**（启动、关闭、重启、迁移等）、收集节点上的**宿主机****！和客户机的配置和资源使用**状态。

### 2.2 libvirtd命令参数含义

（1）-d或--daemon

表示让libvirtd作为**守护进程**（daemon）在**后台运行**。

（2）-f或--config FILE

指定**libvirtd的配置文件为FILE**，而不是使用默认值（通常是/**etc/libvirt/libvirtd.conf**）。

（3）-l或--listen

开启配置文件中配置的TCP/IP连接。

（4）-p或--pid-file FILE

将l**ibvirtd进程的PID写入FILE文件**中，而**不是使用默认值**（通常是/**var/run/libvirtd.pid**）。

（5）-t或--timeout SECONDS

设置**对libvirtd连接的超时时间**为SECONDS秒。

（6）-v或--verbose

执行命令**输出详细的输出信息**。特别是在运行出错时，详细的输出信息便于用户查找原因。

（7）--version

显示**libvirtd程序的版本信息**。

## 3、创建虚拟磁盘文件

- 通过qemu-img创建虚拟磁盘文件，给虚拟机创建（共享）存储

### 3.1 虚拟机配置文件位置

通过修改配置文件来添加硬盘，我们首先要关闭虚拟机，否则无法正常添加。

关闭虚拟机，然后使用virsh edit命令修改虚拟机的主配置文件。

虚拟机的所有配置文件都存放在/etc/libvirt/qemu

```Shell
[root@kvm qemu]# ll
总用量 16
drwx------. 3 root root   42 12月 14 10:39 networks
-rw-------  1 root root 4546 12月 14 15:56 node01.xml
-rw-------  1 root root 4288 12月 14 14:30 node02.xml
[root@kvm qemu]# 
```

### 3.2 编辑虚拟机配置文件

```Shell
[root@kvm qemu]# qemu-img create -f qcow2 /home/images/add1.qcow2 5G
Formatting '/home/images/add1.qcow2', fmt=qcow2 size=5368709120 encryption=off cluster_size=65536 lazy_refcounts=off 
[root@kvm qemu]# virsh edit node01
<disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/images/node01.qcow2'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/images/add1.qcow2'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </disk>

编辑了域 node01 XML 配置。

[root@kvm qemu]# virsh start node01
域 node01 已开始
```

### 3.3 查看添加的磁盘

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=Mzk4YmQ0ZWQyMGYzZDdkZGVjMWU1MzdhZWY1ZGQxMzFfbmVldkhGbVpuZ1Y2MmQ0WFA0T0dORDY5WXdEQ2pDNEZfVG9rZW46Ym94Y25XREQwR0NpUEZneXRGTk8xdGx5TE5kXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

vda磁盘添加成功

## 4、虚拟机的克隆

将虚拟机 centos7 克隆为虚拟机 node01

注意：克隆前需要先关闭虚拟机；克隆完毕，一般需要设置虚拟机的网络。

- 通过virt-clone进行虚拟机的克隆

```Shell
[root@kvm images]# virt-clone -o centos7 -n node02 -f /home/images/node02.qcow2
正在分配 'node02.qcow2'                                     |  30 GB  00:01:47     

成功克隆 'node02'。
[root@kvm images]# virsh list --all
 Id    名称                         状态
----------------------------------------------------
 -     centos7                        关闭
 -     node01                         关闭
 -     node02                         关闭
```

## 5、创建虚拟机快照和恢复快照

### **5.1 创建快照的条件**

- 虚拟机是关机状态。
- 虚拟机镜像格式是 qcow2。

### 5.2 创建、恢复快照

- 通过virsh 命令创建虚拟机快照和恢复快照

```Shell
 [root@kvm ~]# virsh snapshot-create-as centos7 02
已生成域快照 02
 [root@kvm ~]# virsh snapshot-list centos7
 名称               生成时间              状态
------------------------------------------------------------
 01                   2022-12-13 17:12:23 +0800 shutoff
 02                   2022-12-13 17:16:20 +0800 shutoff

 [root@kvm ~]# virsh snapshot-revert centos7 02
```

### **5.3  删除快照**

```Shell
[root@kvm ~]# virsh snapshot-delete centos7 02
```

### **5.4 快照文件存储位置**

```Shell
/var/lib/libvirt/qemu/snapshot
```

## 6、kvm虚拟网络配置

### 6.1 固定虚拟机分配固定的ip地址

1. 首先用virsh dumpxml查看网卡的MAC地址：

```Shell
[root@kvm networks]# virsh dumpxml node02 |grep 'mac address'
      <mac address='52:54:00:2b:3b:d7'/>
```

1. 修改libvirt网络配置：

```Shell
[root@kvm networks]# virsh net-edit default 
<ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
      <host mac='52:54:00:2b:3b:d7' name='node02' ip='192.168.122.100'/>
    </dhcp>
  </ip>
  已编辑网络 default XML 配置
```

1. 存盘退出。重启网络：

```Shell
virsh net-destroy default
virsh net-start default
```

1. 查看配置结果

![img](https://dnllxg8y7u.feishu.cn/space/api/box/stream/download/asynccode/?code=YTllNjE1YTMxMjk5NTEzMjU1MjVlZDkxZDY5YmNjM2JfbDFONHJiV1R0OE5mVVU4elQ5YzJSWmVHMHh0Nlo0V1NfVG9rZW46Ym94Y252cDNhMmlGUVA0TjhpTWN3dlp1SDVmXzE2NzYxMDM5MTg6MTY3NjEwNzUxOF9WNA)

### 6.2 使用iptables转发虚拟机端口

- 在虚拟机使用iptables转发虚拟机端口，暴露22端口

1. 在nat表中添加了这2条规则：
   1.  iptables -t nat -A PREROUTING -d 192.168.137.105 -p tcp -m tcp --dport 8022 -j DNAT --to-destination       192.168.122.100:22

   2.  iptables -t nat -A POSTROUTING -s 192.168.122.0/255.255.255.0 -d 192.168.122.100 -p tcp -m tcp --dport 22 -j SNAT --to-source 192.168.122.1

   3.  注意这两条IPtables规则：

   4.  第一条规则很好理解，就是把所有访问192.168.137.105:8022的请求转发到192.168.122.100:22的端口上。

   5.  第二条规则我的理解是，把所有来自192.168.122.0/255.255.255.0网段访问192.168.122.100：22的数据全部通过192.168.122.1这个网关转发出去。
2. 删除了表格过滤器FORWARD链的规则4和5

```Shell
[root@kvm ~]# $sudo iptables -nL -v --line-numbers -t filter
Chain INPUT (policy ACCEPT 294 packets, 25782 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1       14   901 ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
2        0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53
3        1   328 ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:67
4        0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:67

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1       77  5884 ACCEPT     all  --  *      virbr0  0.0.0.0/0            192.168.122.0/24     ctstate RELATED,ESTABLISHED
2       86  6568 ACCEPT     all  --  virbr0 *       192.168.122.0/24     0.0.0.0/0           
3        0     0 ACCEPT     all  --  virbr0 virbr0  0.0.0.0/0            0.0.0.0/0           
4        9   500 REJECT     all  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable
5        0     0 REJECT     all  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable
6        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.122.100      state NEW tcp dpt:22

Chain OUTPUT (policy ACCEPT 232 packets, 27575 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        1   328 ACCEPT     udp  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            udp dpt:68
[root@kvm ~]# $sudo iptables -D FORWARD 5 -t filter
[root@kvm ~]# $sudo iptables -D FORWARD 4 -t filter
```

1. 连接虚拟机node02

```Shell
[root@slqxhhtt ~]# ssh -p 8022 root@192.168.137.105
Warning: Permanently added '[192.168.137.105]:8022' (ECDSA) to the list of known hosts.
root@192.168.137.105's password: 
Last login: Thu Dec 15 10:50:04 2022
[root@node02 ~]# 
```