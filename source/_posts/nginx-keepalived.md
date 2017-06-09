---
title: "Nginx+Keepalived实现web服务器的高可用"
date: 2016-08-11
categories: nginx
tags: [ nginx,keepalived,vrrp ]
---

#### 简介
**VRRP协议**
&emsp;&emsp;VRRP全称 Virtual Router Redundancy Protocol，即[虚拟路由冗余协议](http://baike.baidu.com/view/876125.htm)。 可以认为它是实现路由器高可用的容错协议，即将N台提供相同功能的路由器组成一个路由器组(Router Group)，这个组里面有一个master和多个backup，但在外界看来就像一台一样，构成虚拟路由器，拥有一个虚拟IP（vip，也就是路由器所 在局域网内其他机器的默认路由），占有这个IP的master实际负责ARP相应和转发IP数据包，组中的其它路由器作为备份的角色处于待命状态。 master会发组播消息，当backup在超时时间内收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个 backup当master，保证路由器的高可用。
&emsp;&emsp;在VRRP协议实现里，虚拟路由器使用 00-00-5E-00-01-XX 作为虚拟MAC地址，XX就是唯一的 VRID （Virtual Router IDentifier），这个地址同一时间只有一个物理路由器占用。在虚拟路由器里面的物理路由器组里面通过多播IP地址 224.0.0.18 来定时发送通告消息。每个Router都有一个 1-255 之间的优先级别，级别最高的（highest priority）将成为主控（master）路由器。通过降低master的优先权可以让处于backup状态的路由器抢占（proempt）主路由 器的状态，两个backup优先级相同的IP地址较大者为master，接管虚拟IP。
**Keepalived**
&emsp;&emsp;keepalived可以认为是VRRP协议在Linux上的实现，主要有三个模块，分别是core、check和vrrp。core模块为 keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。check负责健康检查，包括常见的各种检查方式。vrrp模块是来实现VRRP协议的。
**Keepalived配置项说明**
**global_defs{}**
- ***notification_email*** ： keepalived在发生诸如切换操作时需要发送email通知地址，此方式需要自建邮件服务。
- ***router_id*** ： 机器标识，通常可设为hostname。故障发生时，邮件通知会用到

**vrrp_instance{}**
- ***state*** ： 指定instance(Initial)的初始状态，就是说在配置好后，这台服务器的初始状态就是这里指定的，但这里指定的不算，还是得要通过竞选通过优 先级来确定。如果这里设置为MASTER，但如若他的优先级不及另外一台，那么这台在发送通告时，会发送自己的优先级，另外一台发现优先级不如自己的高， 那么他会就回抢占为MASTER
- ***interface*** ： 实例绑定的网卡，因为在配置虚拟IP的时候必须是在已有的网卡上添加的
- ***mcast_src_ip*** ： 发送多播数据包时的源IP地址，这里注意了，这里实际上就是在那个地址上发送VRRP通告，这个非常重要，一定要选择稳定的网卡端口来发送，这里相当于 heartbeat的心跳端口，如果没有设置那么就用默认的绑定的网卡的IP，也就是interface指定的IP地址
- ***virtual_router_id*** ： 虚拟路由ID，同组内的VRID必须相同，取值0~255
- ***priority*** ： 设置本节点的优先级，优先级高的为master
- ***advert_int*** ： 检查间隔，默认为1秒。这就是VRRP的定时器，MASTER每隔这样一个时间间隔，就会发送一个advertisement报文以通知组内其他路由器自己工作正常
- ***authentication*** ： 定义认证方式和密码，主从必须一样
- ***virtual_ipaddress*** ： 这里设置的就是VIP，也就是虚拟IP地址，他随着state的变化而增加删除，当state为master的时候就添加，当state为backup的时候删除，这里主要是有优先级来决定的，和state设置的值没有多大关系，这里可以设置多个IP地址
- ***track_script*** ： 引用VRRP脚本，即在 vrrp_script 部分指定的名字。定期运行它们来改变优先级，并最终引发主备切换。 
- ***notify_master***: 指定服务器状态切换为master时执行的脚本
- ***notify_backup***: 指定服务器状态切换为backup时执行的脚本
- ***notify_fault***: 指定服务器状态出错时执行的脚本
 
**vrrp_script{}**
- ***script*** ： 自己写的检测脚本。也可以是一行命令如killall -0 nginx
- ***interval*** 2 ： 每2s检测一次
- ***weight*** -5 ： 检测失败（脚本返回非0）则优先级 -5
- ***fall*** 2 ： 检测连续 2 次失败才算确定是真失败。会用weight减少优先级（1-255之间）
- ***rise*** 1 ： 检测 1 次成功就算成功。但不修改优先级

#### 系统架构图
![](http://obbrvy8yl.bkt.clouddn.com/20160810nginx-keepalived.png)
#### 系统安装步骤

##### 分别在两台主机上安装nginx+keepalived

```
# wget http://nginx.org/download/nginx-1.10.1.tar.gz
# tar xf nginx-1.10.1.tar.gz
# cd nginx-1.10.1
# ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_flv_module --sbin-path=/usr/sbin/nginx --with-http_stub_status_module --with-http_gzip_static_module --with-pcre
# make && make install
# yum install keepalived

```

##### 编辑Master的配置文件
```
# vim /etc/keepalived/keepalived.conf
global_defs {
   router_id 333
}
vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}
vrrp_instance nginx {
    state MASTER
    interface eth0
    virtual_router_id 88
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass abc123
    }
    virtual_ipaddress {
        192.168.5.18
    }
    track_script {
        chk_nginx
    }
    notify_master /etc/keepalived/master.sh
    notify_backup /etc/keepalived/backup.sh
}

```

##### 编辑Backup的配置文件
```
# vim /etc/keepalived/keepalived.conf
global_defs {
   router_id 333
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}

vrrp_instance nginx {
    state BACKUP
    interface eth0
    virtual_router_id 88
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass abc123
    }
    virtual_ipaddress {
        192.168.5.18
    }
    track_script {
        chk_nginx
    }
    notify_master /etc/keepalived/master.sh
    notify_backup /etc/keepalived/backup.sh
}

```
##### 调用的shell脚本
- ***check_nginx.sh：***检查nginx的健康状态，如发现nginx无法启动就关闭keepalived服务
```bash
#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    /usr/sbin/nginx
    sleep 2
    counter=$(ps -C nginx --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
        /etc/init.d/keepalived stop
    fi
fi
```

- ***master.sh:***服务器状态切换为master时执行的脚本，发邮件告诉管理员
```bash
#!/bin/bash
Date=$(date +%F" "%T)
IP=$(ifconfig eth0 |grep "inet addr" |cut -d":" -f2 |awk '{print $1}')
Mail="abc@123.com"
counter=$(ps -C nginx --no-heading|wc -l)

echo "$Date $IP change to master." |mail -s "Master-Backup Change Status" $Mail
if [ "${counter}" = "0" ];then
    /usr/sbin/nginx
else 
    /usr/sbin/nginx -s reload
fi
```
- ***backup.sh:***服务器状态切换为backup时执行的脚本
```bash
#!/bin/bash
Date=$(date +%F" "%T)
IP=$(ifconfig eth0 |grep "inet addr" |cut -d":" -f2 |awk '{print $1}')
Mail="abc@123.com"
echo "$Date $IP change to backup." |mail -s "Master-Backup Change Status" $Mail
```
#### 第三方邮件报警配置
&emsp;&emsp;由于keepalived发送邮件依赖于本地邮件服务，不太方便；建议使用第三方邮箱服务，可通过修改mail.rc文件实现。
```bash
# vim /etc/mail.rc
set from=abc@123.com
set smtp=smtp.exmail.qq.com
set smtp-auth-user=abc@123.com
set smtp-auth-password="xxx"
set smtp-auth=login
```
#### 定时同步nginx配置
&emsp;&emsp;为了保证nginx配置一致，需要定时从Master同步nginx配置到Backup，写个crontab任务即可
```bash
# crontab -e
30 03 * * *   /usr/bin/rsync -avzP --exclude=logs /usr/local/nginx 192.168.5.12:/usr/local &> /dev/null
```
#### 验证方法
##### 为nginx创建测试页面
Master
```
# echo '<h1>node1</h1>' > /usr/local/nginx/html/index.html
```
Backup
```
# echo '<h1>node2</h1>' > /usr/local/nginx/html/index.html
```
##### 浏览器访问
访问VIP的80端口，正常情况下会显示"node1"；手动关闭master上的nginx，并使之无法正常启动；再次访问VIP的80端口，如显示"node2"，验证成功。
可监控/var/log/messages日志文件，查看服务器状态变化。
