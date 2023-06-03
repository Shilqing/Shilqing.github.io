---
title: 配置Keepalived高可用
date: '2022-12-24 14:37:00'
tags: keepalived
categories: Linux运维
cover: 'https://data-static.netdun.net/Fomalhaut/img/default_cover_39.webp'
abbrlink: 47121
---

## 配置Keepalived高可用

 ## 1、编译安装

```
1.安装依赖
[root@slqxhhtt ~]# yum install -y gcc kernel kernel-devel openssl openssl-devel popt popt-devel
2.编译Keepalived
[root@slqxhhtt ~]# wget http://www.keepalived.org/software/keepalived-1.4.3.tar.gz
[root@slqxhhtt ~]# tar -xzvf keepalived-1.4.3.tar.gz
[root@slqxhhtt ~]# cd keepalived-1.4.3/
#RHEL7中的编译参数
[root@slqxhhtt ~]# ./configure --prefix=/ --with-kernel-dir=/usr/src/kernels/3.10.0-123.el7.x86_64/net/

[root@slqxhhtt ~]# make && make install

```

## 2、实验环境

```text
[实验环境]

[类型]            [IP地址]              [VIP/IO]

LVS1_Master     IP:192.168.137.100       VIP:192.168.137.90
LVS2_Slaves     IP:192.168.137.110                   

RealServer_1        IP:192.168.137.101
RealServer_2        IP:192.168.137.102
RealServer_3        IP:192.168.137.103
[配置说明]

1.首先我们需要配置一个LVS实现负载均衡
2.其次安装第二个LVS但不需要配置
3.在第一个LVS上配置轮询规则,并且安装开启Keepalived服务
4.在第二个LVS上安装Keepalived服务,启动后会自动同步数据
```

## 3、配置主节点

```shell
[root@slqxhhtt ~]# vim /etc/keepalived/keepalived.conf

 1 ! Configuration File for keepalived
 2 
 3 global_defs {
★    router_id kp_master                    #指定本机keepalaved名字(主从不能重复)
 5 }
 6 
 7 vrrp_instance VI_1 {
★     state MASTER                      #声明成主服务器(MASTER)/声明成从服务器(SLAVE)
★     interface ens33                        #定义相应网卡接口名称
★     virtual_router_id 100                 #虚拟路由ID（主从应同步）
★     priority 100                      #Keepalaved主从服务器优先级(主服务器必须大于从服务器)
12     advert_int 1                     #检查间隔,默认1秒
13     authentication {                     #定义主从验证
14         auth_type PASS                   #设置验证方式(PASS或HA)
15         auth_pass 1111                   #验证密码
16     }
17     virtual_ipaddress {                  #指定负载调度器（指定VIP的地址）
★         192.168.137.90
19     }
20 }
21 
★ virtual_server 192.168.137.90 22 {                #虚拟主机区域(指定VIP地址)
23     delay_loop 6                     #服务器轮询间隔时间
24     lb_algo rr                       #指定rr轮询算法
★     lb_kind DR                        #指定DR模式
★     net_mask 255.255.255.0                    #指定子网掩码
27     persistence_timeout 0                   #会话保持时间
28     protocol TCP                     #指定数据转发协议
29 
★     real_server 192.168.137.101 22 {               #RealServer1池，如有多台复制此区域
31         weight 1                     #设置服务器权重
★         TCP_CHECK {                       #对后端真实服务器TCP健康检查
33             connect_timeout 3                #链接超时时间
34             retry 3                      #重试次数
35             delay_before_retry 3             #重试时间间隔
36         }
37     }
38 
★     real_server 192.168.137.102 22 {               #RealServer2池，如有多台复制此区域
40         weight 1                     #设置服务器权重
★         TCP_CHECK {                       #对后端真实服务器TCP健康检查
42             connect_timeout 3                #连接超时时间
43             retry 3                      #重试次数
44             delay_before_retry 3             #重试时间间隔
45         }
46     }
47    real_server 192.168.137.103 22 {               #RealServer3池，如有多台复制此区域
48        weight 1                     #设置服务器权重
★         TCP_CHECK {                       #对后端真实服务器TCP健康检查
50             connect_timeout 3                #连接超时时间
51             retry 3                      #重试次数
52             delay_before_retry 3             #重试时间间隔
53         }
54     }
55 }
```

```
[root@slqxhhtt ~]# systemctl restart keepalived
```

## 4、配置备节点

1.修改配置文件

```shell
! Configuration File for keepalived

   global_defs {
    router_id kp_slave
   }

   vrrp_instance VI_1 {
     state SLAVE
     interface ens33
     virtual_router_id 100
     priority 50
     advert_int 1
     authentication {
         auth_type PASS
         auth_pass 1111
     }
     virtual_ipaddress {
        192.168.137.90
     }
 }

 virtual_server 192.168.137.90 22 {
     delay_loop 6
     lb_algo rr
       lb_kind DR
       net_mask 255.255.255.0
     persistence_timeout 0
     protocol TCP

    real_server 192.168.137.101 22 {
         weight 1
           TCP_CHECK {
             connect_timeout 3
             retry 3
             delay_before_retry 3
           }
     }
   real_server 192.168.137.102 22 {
         weight 1
           TCP_CHECK {
             connect_timeout 3
             retry 3
             delay_before_retry 3
         }
     }

    real_server 192.168.137.103 22 {
         weight 1
           TCP_CHECK {
             connect_timeout 3
             retry 3
             delay_before_retry 3
         }
     }
   }

```

2.修改内核参数.防止相同网络地址广播冲突

```
[root@slqxhhtt ~]# vim /etc/sysctl.conf

net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.eth0.send_redirects = 0

[root@slqxhhtt ~]# sysctl -p
```

3.启动Keepalived服务

```
[root@slqxhhtt ~]# systemctl restart keepalived
```