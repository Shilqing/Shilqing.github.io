---
title: PXE+ Kickstart无人值守系统
date: '2022-11-24 00:37:00'
tags: PXE+ Kickstart
sticky: 1
categories: Linux运维
cover: 'https://data-static.netdun.net/Fomalhaut/img/default_cover_19.webp'
abbrlink: 24431
---


## PXE+ Kickstart无人值守系统



关闭防护墙，SELinux策略

```
systemctl stop firewalld
systemctl disable firewalld
vim /etc/selinux/config  改成disabled
setenforce 0
```

## 配置DHCP服务程序

安装dhcp，修改配置文件/etc/dhcp/dhcpd.conf

```
allow bootp;
ddns-update-style none;
ignore client-updates;
subnet 192.168.137.0 netmask 255.255.255.0 {
        range                           192.168.137.100   192.168.137.200;
        option subnet-mask              255.255.255.0;
        option domain-name-servers      192.168.137.100;
        default-lease-time              21600;
        max-lease-time                  43200;
        next-server                     192.168.137.100;
        filename                        "pxelinux.0";
```

```
可指定主机名与固定ip地址一一对应
host Client101{
                hardware ethernet 00:50:56:34:C0:1D;
                fixed-address 192.168.137.101;
}
 host Client102{
                hardware ethernet 00:50:56:34:C6:03;
                fixed-address 192.168.137.102;
}

        host Client103{
                hardware ethernet 00:50:56:23:1A:05;
                fixed-address 192.168.137.103;
}
```



重启该服务程序，并将其添加到开机启动项中

```
[root@slqxhhtt ~]# vim /etc/dhcp/dhcpd.conf
[root@slqxhhtt ~]# systemctl restart dhcpd
[root@slqxhhtt ~]# systemctl enable dhcpd
[root@slqxhhtt ~]# systemctl status dhcpd
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2022-11-17 11:55:49 CST; 1min 45s ago
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)
```



## 配置TFTP服务程序

安装tftp-server 和 xinetd

配置FTP服务程序，为客户端主机提供引导及驱动文件的传输

```
[root@slqxhhtt ~]# vim /etc/xinetd.d/tftp
service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -s /var/lib/tftpboot
        disable                 = no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
```

重启xinetd服务程序，并将其加入到开机启动项中

```
[root@slqxhhtt ~]# systemctl restart xinetd
[root@slqxhhtt ~]# systemctl enable  xinetd
```



## 配置SYSLinux服务程序

安装SYSLinux：提供驱动及引导文件

```
[root@slqxhhtt ~]# cd /var/lib/tftpboot/
[root@slqxhhtt tftpboot]# cp /usr/share/syslinux/pxelinux.0 .
[root@slqxhhtt tftpboot]# cp /mnt/cdrom/images/pxeboot/vmlinuz . # 内核镜像文件
[root@slqxhhtt tftpboot]# cp /mnt/cdrom/images/pxeboot/initrd.img . # 文件系统镜像
```

配置开机时的选择菜单

```
[root@slqxhhtt tftpboot]# mkdir pxelinux.cfg
[root@slqxhhtt tftpboot]# cp /mnt/cdrom/isolinux/isolinux.cfg pxelinux.cfg/default
[root@slqxhhtt tftpboot]# vim pxelinux.cfg/default
# 第1行
default linux
# 第64行
label linux
  menu label ^Install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=ftp://192.168.137.100/centos7 ks=ftp://192.168.137.100/ks.cfg quiet
```

编辑这个default文件，把第1行的default参数修改为linux，这样系统在开机时就会默认执行那个名称为linux的选项了。对应的linux选项大约在第64行，将默认的光盘镜像安装方式修改成FTP文件传输方式，并**指定好光盘镜像的获取网址以及Kickstart应答文件的获取路径**

##  配置VSFtpd服务程序

VSFtpd作用：提供完整系统镜像的传输

安装vsftpd，并配置/etc/vsftpd/vsftpd.conf，开启匿名公开访问模式

```
anonymous_enable=YES
```

重启vsftpd服务程序，并将其加入到开机启动项中

```
[root@slqxhhtt ~]# systemctl restart vsftpd
[root@slqxhhtt ~]# systemctl enable vsftpd
[root@slqxhhtt ~]# cp -rf /mnt/cdrom/* /var/ftp/centos7/
```

## 创建KickStart应答文件

使用system-config-kickstart图形化界面配置

```bash
yum install -y system-config-kickstart
```



自动生成ks.cfg

```
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
keyboard 'us'
# Root password
rootpw --iscrypted $1$n2x9Y5Qb$JJAW78gaggFWDB6HvhSmR.
# System language
lang zh_CN
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
firstboot --disable
# SELinux configuration
selinux --disabled


# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens33
# Reboot after installation
reboot
# System timezone
timezone Asia/Shanghai
# Use network installation
url --url="ftp://192.168.137.100/centos7"
# System bootloader configuration
bootloader --location=mbr
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --fstype="xfs" --size=1024
part swap --fstype="swap" --size=4096
part / --fstype="xfs" --grow --size=1

%packages
@^graphical-server-environment
@base
@core
@desktop-debugging
@development
@dial-up
@fonts
@gnome-desktop
@guest-agents
@guest-desktop-agents
@hardware-monitoring
@input-methods
@internet-browser
@multimedia
@print-client
@x11
chrony

%end

%post --interpreter=/bin/bash
/bin/echo "Client"`ifconfig|grep "137" | cut -d "." -f 4|cut -d " " -f 1` > /etc/hostname
%end

```
