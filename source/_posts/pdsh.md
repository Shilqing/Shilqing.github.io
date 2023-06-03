---
title: pdsh实现统一管理
date: '2022-12-14 00:37:00'
tags: pdsh
categories: Linux运维
cover: 'https://data-static.netdun.net/Fomalhaut/img/default_cover_73.webp'
abbrlink: 430
---
## 批处理工具pdsh实现统一管理

###  1.安装批处理工具pdsh

解压软件包

编译安装批处理工具，指定包含的模块。

查看安装pdsh的版本以及包含的模块。

```Shell
[root@slqxhhtt ~]# tar jxvf pdsh-2.29.tar.bz2
[root@slqxhhtt ~]# cd pdsh-2.29
[root@slqxhhtt pdsh-2.29]#./configure --with-ssh --without-rsh --with-exec --with-nodeupdown --with-rcmd-rank-list=ssh
[root@slqxhhtt pdsh-2.29]# make && make install
[root@slqxhhtt pdsh-2.29]# pdsh -V
pdsh-2.29
rcmd modules: ssh,rsh,exec (default: ssh)
misc modules: machines,dshgroup
[root@slqxhhtt pdsh-2.29]# pdsh -w Client[101-103] uptime
Client101:  14:23:19 up  3:57,  2 users,  load average: 0.01, 0.03, 0.05
Client102:  14:23:20 up  3:56,  3 users,  load average: 0.00, 0.01, 0.05
Client103:  14:23:20 up  3:56,  1 user,  load average: 0.03, 0.02, 0.05
```

### 2.批量设置客户端IP为静态

编辑脚本文件modifyIP.sh

脚本步骤：

- 通过循环登录不同需要配置静态IP的客户端。
- 用pdsh批处理工具为网卡路径添加软链接。
- 追加网卡信息至客户端网卡内。
- 根据登陆客户端的IP指定其静态的IP地址。
- 修改BOOTPROTO选项为static。
- 用pdsh批处理工具重启客户端的网络。

```Shell
[root@slqxhhtt ~]#vim modifyIP.sh
#!/bin/bash
#登录需要更改网卡信息的节点
loop_login_Client(){
for ip in {101..103};
do
modify_ip $ip
done
}

#更改网卡信息
modify_ip(){
expect <<-EOF
        set timeout 2
        spawn ssh Client${ip}
        expect "]#" {send "echo TYPE=Ethernet>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo PROXY_METHOD=none>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo BROWSER_ONLY=no>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo IPV4_FAILURE_FATAL=no>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo NAME=ens33>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo IPADDR=192.168.137.${ip}>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo NETMASK=255.255.255.0>>/root/ifcfg-ens33\r" }
        expect "]#" {send "echo GATEWAY=192.168.137.2>>/root/ifcfg-ens33\r" }
        expect "]#" {send "sed -i 's/dhcp/static/' /root/ifcfg-ens33\r" }
        expect "]#" {send "exit\r;exp_continue" }
EOF
}
pdsh -w Client[101-103] ln -n /etc/sysconfig/network-scripts/ifcfg-ens33 /root/ifcfg-ens33
loop_login_Client
pdsh -w Client[101-103] systemctl restart network
```