---
title: "yum软件包管理器"
date: 2016-08-11
categories: Linux基础
tags: [ yum,rpm ]
---

#### 简介
&emsp;&emsp;Yum (Yellow dog Updater, Modified) 由Duke University团队，修改Yellow Dog Linux的Yellow Dog Updater开发而成，是一个基于 RPM 包管理的字符前端软件包管理器。能够从指定的服务器自动下载 RPM 包并且安装，可以处理依赖性关系，并且一次安装所有依赖的软件包，无须繁琐地一次次下载、安装。被Yellow Dog Linux本身，以及Fedora、Red Hat Enterprise Linux采用。
**Yum工作机制**
- C/S架构
- Server端：保存rpm包以及元数据，如：包名、版本信息、各包所包含的文件列表、依赖关系、包分组信息等。
- Client端：以安装过程为例
	- 第一步：获取仓库元数据，缓存于本地，缓存目录/var/cache/yum
    - 第二步：yum客户端读取并分析本地缓存的元数据文件，结合本地系统环境做出要安装的程序包的决策，如依赖关系
    - 第三步：根据决策联系yum仓库，下载各程序包缓存到本地，一并进行安装

#### Yum仓库
可以理解为rpm包的文件服务器，repodata目录所在的父目录就是一个可用仓库。
自建一个yum仓库：
- 把rpm包放到一个目录
- 通过createrepo命令分析该目录的rpm包，然后生成元数据目录repodata
- 配置yum客户端使用此仓库即可，就是这么简单。

#### Yum客户端
&emsp;&emsp;要想通过yum管理rpm包，需要读取yum仓库的配置文件，指明仓库访问路径及各种属性信息。主配置文件：'/etc/yum.conf'；一个或多个yum仓库的配置信息可保存为一个文件，文件名以.repo结尾放到'/etc/yum.repos.d'目录下，文件格式如下：
```bash
[REPOID]
name=Some name for this repository    #设置该yum仓库的名称
baseurl=url://server1/path/to/repository/           #方式1：指定yum仓库访问路径，支持多种url，如：ftp、nfs、http等，可指定多个url
http://mirror.centos.org/centos/$releasever/os/$basearch/     #URL中也支持$releasever 、$basearch这样的变量，指定系统版本号
mirrorlist=/path/to/urlfile        #方式2：可以将多个url写入一个文件，然后读取这个文件    
enabled={0|1}        #是否启用该yum仓库，0为不启用，1为启用，默认为1
gpgcheck={0|1}       #是否校验rpm包，0为不启用，1为启用
gpgkey=URL           #如果校验，指定公钥文件路径
```
yum配置文件中可用的四个变量：
- $releasever: 程序的版本，对Yum而言指的是redhat-release版本；只替换为主版本号，如RedHat 6.5，则替换为6; 
- $arch: 系统架构
- $basearch: 系统基本架构，如i686，i586等的基本架构为i386；
- $uuid: 
- $YUM0-9: 在系统中定义的环境变量，可以在yum中使用；
一个配置文件支持多个yum仓库
#### yum命令用法
- 列出所有的repo
```
# yum repolist  {enabled|disabled|all}  #enabled列出可用的，disabled列出禁用的，all列出所有(默认)
```
- 列出rpm包
```
# yum list {all|installed|available}    #all列出所有包(默认)，installed列出已安装的包，available列出可安装的包
# yum list KEYWORD*    #列出指定前缀的所有包，支持文件名通配
```
- 列出包的描述信息
```
# yum info package_name
```
- 列出包组
```
# yum grouplist
```
- 列出包组的详细信息
```
# yum groupinfo "GROUP NAME"
```
- 清理缓存
```
# yum clean {all|packages|metadata|expire-cache|rpmdb|plugins}
```
- 安装程序包
```
# yum install package_name
# yum reinstall package_name    #重新安装
```
- 升级或降级程序包
```
# yum check-update    #检查可用的升级包
# yum update package_name    #可以指定升级包的版本号升级；如果不指定包名，就升级所有可用的升级包
# yum downgrade package_name    #包降级
```

- 卸载程序包
```
# yum {remove|erase} package_name
```
- 查询某文件是由哪一个包安装生成的
```
# yum {whatprovides|provides} /path/to/somefile
```
- 安装包组
```
# yum groupinstall "GROUP NAME"
```
- 卸载包组
```
# yum groupremove "GROUP NAME"
```
- 只下载包，不安装
```
# yumdownloader package_name    #此命令由yum-utils包提供
```
- 另外一些常用选项介绍
```
--nogpgcheck    #不对包做校验
--disablerepo=[repo]    #安装时禁用某些yum仓库
-y：    #对所有交互式操作返回yes
[root@qin ~]# yum history    #显示yum的历史记录
[root@qin ~]# yum history list X    #X是历史事务ID，显示指定事务明细操作
[root@qin ~]# yum history redo X    #重新执行指定事务
[root@qin ~]# yum history undo X    #回滚指定事务
[root@qin ~]# yum makecache         #生成元数据
[root@qin ~]# yum search  STRING    #查找包含指定关键字的包
```
