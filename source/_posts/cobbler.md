---
title: "基于Cobbler实现多版本操作系统自动部署"
date: 2016-08-11
categories: Cobbler
tags: [ cobbler,kickstart ]
---
**前言**
&emsp;&emsp;在生产环境中，当需要批量部署几十甚至上百台服务器时，实现自动化安装操作系统尤为重要，按照传统的光盘引导安装是不可想象的；此前我们通过pxe+kickstart简单实现了自动化安装，但只能实现单一版本安装，当需要部署不同版本或不同引导模式（BIOS、EFI）时，此种方式就不够灵活。而Cobbler正是为了解决此问题而设计的，本文以CentOS为例简单介绍Cobbler的安装配置及使用。
**简介**
&emsp;&emsp;Cobbler是一个免费开源系统安装部署软件，用于自动化网络安装操作系统。Cobbler可以集成DNS, DHCP, 软件包更新以及配置管理，方便操作系统安装自动化。Cobbler 可以支持PXE启动, 操作系统重新安装,以及虚拟化客户机创建，包括Xen, KVM or VMware. Cobbler透过koan程序以支持虚拟化客户机安装。Cobbler可以支持管理复杂网路环境，如建立在链路聚合以太网的桥接环境。
**Cobbler组件结构图**

<div align=center>
  <img src="http://obbrvy8yl.bkt.clouddn.com/20160809cobbler.png"/>
</div>

#### 安装cobbler及相关软件

```
# yum -y install cobbler dhcp rsync
# chkconfig tftp on
# chkconfig rsync on
```

#### 配置cobbler

##### 启动相关服务tftp、cobbler

```
# service xinetd start
# service cobblerd start
```

##### 检查cobbler配置

```sh
# cobbler check
The following are potential configuration items that you may want to fix:
 
1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
4 : debmirror package is not installed, it will be required to manage debian deployments and repositories
5 : ksvalidator was not found, install pykickstart
6 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
7 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
 
Restart cobblerd and then run 'cobbler sync' to apply changes.
```

##### 根据提示修正cobbler配置
- 指定cobbler服务器及dhcp服务器地址

```
# vim /etc/cobbler/settings 
server: 192.168.18.45
next_server: 192.168.18.45
```

- 获取cobbler引导程序 
 
```
# cobbler get-loaders
```

- 安装依赖包pykickstart  

```
# yum -y install pykickstart
```

- 生成一个密码串，替换默认密码  

```
# openssl passwd -1 -salt 345223 123456
$1$345223$/jb8Mdzzy3SRfwM5RbG3D.
# vim /etc/cobbler/settings
default_password_crypted: "$1$345223$/jb8Mdzzy3SRfwM5RbG3D."
```

- 第4、7项提示可忽略

##### 同步cobbler配置，再检查配置是否报错

```sh
# cobbler sync
# service cobblerd restart
# cobbler check
The following are potential configuration items that you may want to fix:
 
1 : debmirror package is not installed, it will be required to manage debian deployments and repositories
2 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
 
Restart cobblerd and then run 'cobbler sync' to apply changes.
```

##### 准备kickstart文件

```sh
# cp /var/lib/cobbler/kickstarts/sample_end.ks /var/lib/cobbler/kickstarts/basic.ks
# vim /var/lib/cobbler/kickstarts/basic.ks 


# kickstart template for Fedora 8 and later.
# (includes %end blocks)
# do not use with earlier distros
 
#platform=x86, AMD64, or Intel EM64T
# System authorization information
auth  --useshadow  --enablemd5
# System bootloader configuration
bootloader --location=mbr
# Partition clearing information
clearpart --all --initlabel
# Use text mode install
text
# Firewall configuration
firewall --disable
# Run the Setup Agent on first boot
firstboot --disable
# System keyboard
keyboard us
# System language
lang en_US
# Use network installation
url --url=$tree
# If any cobbler repo definitions were referenced in the kickstart profile, include them here.
$yum_repo_stanza
# Network information
$SNIPPET('network_config')
# Reboot after installation
reboot
 
#Root password
rootpw --iscrypted $default_password_crypted   #md5加密的密码串，默认保存在/etc/cobbler/settings
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx
# System timezone
timezone  Asia/Chongqing
# Install OS instead of upgrade
install
# Clear the Master Boot Record
zerombr
# Allow anaconda to partition the system as needed
part pv.01 --grow --size=1            
part /boot --fstype=ext4 --size=400
part /boot/efi --fstype=efi --size 200   #EFI模式需要创建该分区
volgroup VolGroup --pesize=4096 pv.01
logvol / --fstype=ext4 --name=lv_root --vgname=VolGroup --grow --size=1024 --maxsize=51200
logvol swap --name=lv_swap --vgname=VolGroup --grow --size=1984 --maxsize=4096
 
 
%pre
$SNIPPET('log_ks_pre')
$SNIPPET('kickstart_start')
$SNIPPET('pre_install_network_config')
# Enable installation monitoring
$SNIPPET('pre_anamon')
%end
 
%packages
@base
@development
@server-platform-devel
samba-client
lftp
openssh-clients
epel-release
#$SNIPPET('func_install_if_enabled')
%end
 
%post --nochroot
$SNIPPET('log_ks_post_nochroot')
%end
 
%post
echo -e "alias net-pf-10 off\noptions ipv6 disable=1" >/etc/modprobe.d/ipv6.conf
echo "NETWORKING_IPV6=no" >> /etc/sysconfig/network
chkconfig ip6tables off
cat >> /etc/security/limits.conf <<EOF
*           soft    nproc           65535
*           hard    nproc           65535
*           soft    nofile          102400
*           hard    nofile          204800
EOF
$SNIPPET('log_ks_post')
# Start yum configuration
$yum_config_stanza
# End yum configuration
$SNIPPET('post_install_kernel_options')
$SNIPPET('post_install_network_config')
$SNIPPET('func_register_if_enabled')
$SNIPPET('download_config_files')
$SNIPPET('koan_environment')
$SNIPPET('redhat_register')
$SNIPPET('cobbler_register')
# Enable post-install boot notification
$SNIPPET('post_anamon')
# Start final steps
$SNIPPET('kickstart_done')
# End final steps
%end
```

##### 导入操作系统到cobbler并添加profile

```
# mount /dev/sr0 /mnt  #挂载光驱
# cobbler import --name=CentOS-6.6-x86_64 --path=/mnt  #导入操作系统
# ll /var/www/cobbler/ks_mirror/  #验证一下是否导成功
# cobbler profile add --name=CentOS-6.6-basic --distro=CentOS-6.6-x86_64 --kickstart=/var/lib/cobbler/kickstarts/basic.ks
# cobbler profile list
# cobbler sync  
```

#### 配置yum源
配置本地yum仓库

```
# mkdir /tmp/rpms
# createrepo /tmp/rpms  #放入rpm包,执行此步骤
# cobbler repo add --mirror=/tmp/rpms --name=local
# cobbler reposync
```

配置epel源

```
# cobbler repo add --mirror=http://mirrors.aliyun.com/epel/6/x86_64/ --name=epel
```

同步epel仓库到本地,需要较长时间

```
# cobbler reposync --tries=3 --no-fail 
```

查看已添加的repo

```
# cobbler repo list
```

添加repo到profile

```
# cobbler profile edit --name=CentOS-6.6-basic --repos="epel local"
```

最后执行同步

```
# cobbler sync
```

#### 配置dhcp服务器
##### 修改配置文件dhcpd.conf

```sh
# vim /etc/dhcp/dhcpd.conf
option arch code 93 = unsigned integer 16; # RFC4578
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
subnet 192.168.18.0 netmask 255.255.255.0 {
    range 192.168.18.150 192.168.18.230;
    option domain-name "example.org";
    option domain-name-servers 114.114.114.114,8.8.8.8;
    option routers 192.168.18.1;
          class "pxeclients" {
                  match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
                  next-server 192.168.18.45;
 
                  if option arch = 00:06 {
                          filename "grub/grub-x86.efi";
                  } else if option arch = 00:07 {
                          filename "grub/grub-x86_64.efi";
                  } else {
                          filename "pxelinux.0";
                  }
        }
}
```

#### 部署完毕，重新启动相关服务

```
# service dhcpd restart
# service xinetd restart
# service httpd restart
# service cobblerd restart
```

部署完毕后可以通过web界面进行管理 http://cobbler_server/cobbler_web
#### client安装操作系统测试
网络启动EFI模式，某些服务器可能不兼容
![](http://obbrvy8yl.bkt.clouddn.com/20160809cobbler1.png)
网络启动BIOS模式
![](http://obbrvy8yl.bkt.clouddn.com/20160809cobbler2.png)
开始安装
![](http://obbrvy8yl.bkt.clouddn.com/20160809cobbler3.png)
**结语**
&emsp;&emsp;一个简单的cobbler服务器到此部署完毕，本文只是简要地介绍了cobbler服务器的安装及配置；更多详情请参考官方文档[http://cobbler.github.io/manuals/](http://cobbler.github.io/manuals/)
