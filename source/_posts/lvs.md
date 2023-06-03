---
title: 构建lvs负载均衡
date: '2022-12-14 08:37:00'
tags: LVS
categories: Linux运维
cover: 'https://data-static.netdun.net/Fomalhaut/img/default_cover_88.webp'
abbrlink: 11391
---

## 构建LVS负载均衡集群（DR模式）

## 1、实验环境

三台机器：

Director节点：  (ens33 192.168.137.100 vip ens33:0 192.168.137.90)

Real server1： (ens33 192.168.137.101 vip lo:0 192.168.137.90)

Real server2： (ens33 192.168.137.102 vip lo:0 192.168.137.90)

Real server3： (ens33 192.168.137.103 vip lo:0 192.168.137.90)

## 2、配置Director节点

```
[root@slqxhhtt ~]# yum -y install ipvsadm
[root@slqxhhtt ~]# cd /etc/sysconfig/network-scripts/
[root@slqxhhtt network-scripts]# vim ifcfg-ens33:0
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME=ens33:0
DEVICE=ens33:0
ONBOOT=yes
IPADDR=192.168.137.90
NETMASK=255.255.255.0
```

```
[root@slqxhhtt ~]# systemctl restart network
[root@slqxhhtt ~]# ifconfig
```

调整 proc 响应参数并刷新(关闭icmp重定向，不充当路由)

```
[root@slqxhhtt ~]# vim /etc/sysctl.conf
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.ens33.send_redirects = 0
[root@slqxhhtt ~]# sysctl -p
```

加载模块。

配置负载分配策略并启动服务。

设置开机服务自启动。

```shell
[root@slqxhhtt ~]# modprobe ip_vs
[root@slqxhhtt ~]# ipvsadm -C  #清空策略
[root@slqxhhtt ~]# ipvsadm -A -t 192.168.137.90:22 -s rr
[root@slqxhhtt ~]# ipvsadm -a -t 192.168.137.90:22 -r 192.168.137.101 -g
[root@slqxhhtt ~]# ipvsadm -a -t 192.168.137.90:22 -r 192.168.137.102 -g
[root@slqxhhtt ~]# ipvsadm -a -t 192.168.137.90:22 -r 192.168.137.103 -g
[root@slqxhhtt ~]# ipvsadm -ln    #查看策略
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.137.90:22 rr
  -> 192.168.137.101:22           Route   1      0          0
  -> 192.168.137.102:22           Route   1      0          0
  -> 192.168.137.103:22           Route   1      0          0
[root@slqxhhtt ~]# ipvsadm-save >/etc/sysconfig/ipvsadm  #保存配置
[root@slqxhhtt ~]# systemctl enable ipvsadm
```

### 自动化脚本

```shell
#vim lvs_dr.sh
#!/bin/bash
#description: this script to add lvs IP

VIP=192.168.137.90
DIP=192.168.137.100
RIP1=192.168.137.101
RIP2=192.168.137.102
RIP3=192.168.137.103
PORT=22
SCHELE=rr
LOCKFILE=/var/lock/subsys/ipvsadm

case $1 in
start)
#增加vip地址
 /sbin/ifconfig ens33:0 $VIP broadcast $VIP netmask 255.255.255.0
 /sbin/route add -host $VIP dev ens33:0
#清除防火墙规则
 /sbin/iptables -F
 /sbin/iptables -X
 /sbin/iptables -Z
#开启ip转发功能
 echo 1 > /proc/sys/net/ipv4/ip_forward
#清除ipvsadm 规则
 /sbin/ipvsadm -C
#增加ipvsadm direcotor规则
 /sbin/ipvsadm -A -t $VIP:$PORT -s $SCHELE
#增加realserver 规则
 /sbin/ipvsadm -a -t $VIP:$PORT -r $RIP1 -g
 /sbin/ipvsadm -a -t $VIP:$PORT -r $RIP2 -g
/sbin/ipvsadm -a -t $VIP:$PORT -r $RIP3 -g
#增加ipvsadm 锁文件
 /bin/touch  $LOCKFILE
;;
stop)
 if [ ! -e $LOCKFILE ];then
  echo "the ipvsadm is stopped..."
 else
 #删除vip地址
  /sbin/ifconfig ens33:0 down
 #关闭ip转发
  echo 0 > /proc/sys/net/ipv4/ip_forward
 #清除ipvsadm 规则
  /sbin/ipvsadm -C
 #删除锁文件
  /bin/touch $LOCKFILE
 fi
;;
status)
 if [ ! -e $LOCKFILE ];then
  echo "the ipvsadm is stopped..."
 else
  echo "the ipvsadm is running..."
 fi
;;
*)
 echo "Usage；$0:{start|stop|status}"
;;
esac

```



## 3、 配置realserver服务器

关闭防火墙、Seliux服务

添加回环网卡,修改回环网卡名，IP地址，子网掩码。

```
[root@slqxhhtt ~]# cd /etc/sysconfig/network-scripts/
[root@Client101 network-scripts]# cp ifcfg-lo ifcfg-lo:0
[root@Client101 network-scripts]# vim ifcfg-lo:0
DEVICE=lo:0
IPADDR=192.168.137.90
NETMASK=255.255.255.255
NETWORK=127.0.0.0
ONBOOT=yes
[root@Client101 ~]# systemctl restart network
```

设置路由。

开机执行添加路由命令，增加。

```Shell
[root@Client101 ~]# route add -host 192.168.137.90 dev lo:0
[root@Client101 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.137.2   0.0.0.0         UG    100    0        0 ens33
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
192.168.137.0   0.0.0.0         255.255.255.0   U     100    0        0 ens33
192.168.137.90  0.0.0.0         255.255.255.255 UH    0      0        0 lo
[root@Client101 ~]# vim /etc/rc.d/rc.local
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local
/usr/sbin/route add -host 192.168.137.90 dev lo:0
[root@Client101 ~]# chmod +x /etc/rc.d/rc.local #给予执行权限
```

调整 proc 响应参数。

```Shell
[root@Client101 ~]# vim /etc/sysctl.conf
net.ipv4.conf.lo.arp_ignore = 1          #系统只响应目的IP为本地IP的ARP请求
net.ipv4.conf.lo.arp_announce = 2        #系统不使用IP包的源地址来设置ARP请求的源地址，而选择发送接口的IP地址
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
[root@Client101 ~]# sysctl -p  #刷新配置
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

其余两台realserver按照上述同样步骤配置。

### 自动化脚本

```sh
#vim lvs_rs.sh
#!/bin/bash
#description: this script to add real server 
VIP=192.168.137.90
case $1 in
start)
#arp_ignore: 定义接收到ARP请求时的响应级别；1表示仅在请求的目标地址配置请求到达的接口上的时候，才给予响应
#arp_announce：定义将自己地址向外通告时的通告级别：2表示仅向与本地接口上地址匹配的网络进行通告；
 echo 1 >/proc/sys/net/ipv4/conf/lo/arp_ignore
 echo 1 >/proc/sys/net/ipv4/conf/all/arp_ignore
 echo 2 >/proc/sys/net/ipv4/conf/lo/arp_announce
 echo 2 >/proc/sys/net/ipv4/conf/all/arp_announce
#增加VIP地址到lo:0接口,增加路由条目：目的地址为VIP,由lo:0接口响应（即：源地址为VIP作为响应报文给客户端）

 /sbin/ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255  && /sbin/route add -host $VIP dev lo:0

#新建一个锁文件,前面执行成功则建立锁文件
 if [  $? -eq 0 ];then
  /bin/touch /var/lock/subsys/ipvsreal
 else
  echo "fail to add vip address and route."
 fi
;;
stop)
#恢复arp响应级别
    echo 0 >/proc/sys/net/ipv4/conf/lo/arp_ignore
 echo 0 >/proc/sys/net/ipv4/conf/all/arp_ignore
 echo 0 >/proc/sys/net/ipv4/conf/lo/arp_announce
 echo 0 >/proc/sys/net/ipv4/conf/all/arp_announce
#剔除VIP地址（路由地址自动删掉）
  loip=`/sbin/ifconfig lo:0 |grep $VIP`
  if [ '$loip'  == '' ];then
   echo "VIP address not found."
  else
   /sbin/ifconfig lo:0 down && rm -rf /var/lock/subsys/ipvsreal
if [ $? -eq 0 ] ;then
    echo "VIP address had been deled."
   else
    echo "VIP address del failly."
    exit 1
   fi
  fi
;;
status)
 if [ ! -e /var/lock/subsys/ipvsreal ];then
  echo "LVS-DR real server stoped."
 else
  echo "LVS-DR real server is running."
 fi
;;
*)
 echo "Usage: $0 {start | stop |status}"
 exit 1
;;
esac

```



## 4、 监控realserver状态

安装nmap工具（nmap可扫描22端口状态）。

在后台运行脚本。

```Shell
[root@slqxhhtt ~]# yum -y install nmap
[root@slqxhhtt ~]# vim ipvsadm.sh
#!/bin/bash

VIP=192.168.137.90

#nmap判断节点状态
#删除负载均衡节点
judge_clientStatus(){
for ip in {101..103}
do
status=$(nmap -n -sT -p 22 192.168.137.${ip} | grep open)
        if [ -z "${status}" ];
        then
            lvs=$(ipvsadm -Ln | grep 192.168.137.${ip}|cut -d " " -f 4)
             if [ -n "${lvs}" ];
                then
                    ipvsadm -d -t ${VIP}:22 -r 192.168.137.${ip}
            fi
        fi
done
}

#增加负载均衡节点
add_client(){
for ip in {101..103}
do
status=$(nmap -n -sT -p 22 192.168.137.${ip} | grep open)
        if [ -n "${status}" ];
        then
            lvs=$(ipvsadm -Ln | grep 192.168.137.${ip}|cut -d " " -f 4)
            if [ -z "${lvs}" ];
                then
                    ipvsadm -a -t ${VIP}:22 -r 192.168.137.${ip}
            fi
        fi
done
}
while :;
do
judge_clientStatus
add_client
done
[root@slqxhhtt ~]#bash ipvsadm.sh &
```