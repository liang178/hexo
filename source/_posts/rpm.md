---
title: "RPM包管理"
date: 2016-08-11
categories: Linux基础
tags: rpm
---
#### 简介
&emsp;&emsp;RPM（简称RPM，全称为The RPM Package Manager）是在Linux下广泛使用的软件包管理器。RPM此名词可能是指.rpm的文件格式的软件包，也可能是指其本身的软件包管理器(RPM Package Manager)。最早由Red Hat研制，现在也由开源社区开发。RPM通常随附于Linux发行版，但也有单独将RPM作为应用软件发行的发行版（例如Gentoo）。RPM仅适用于安装用RPM来打包的软件，目前是GNU/Linux下软件包资源最丰富的软件包类型之一。
#### RPM命名格式
-  **name-version-release.arch.rpm**
-  **name-version-release.noarch.rpm**
-  **name-version-release.arch.src.rpm**
	- *name:* 软件包名称
	- *version:* 软件的版本号，格式通常为“主版本号.次版本号.修正号”
	- *release:* rpm自身的发行号，与程序源码的发行号无关，仅用于标识对rpm包不同制作的修订；同时，release还包含此包适用的OS
	- *arch:* 适用的硬件平台，如：x86、x86_64、powerpc、noarch等
	- *noarch:* 不需要特定的硬件平台
	- *src:* 源代码包，不可直接安装
- 示例：bash-4.2.3-3.centos5.x86_64.rpm

#### 获取RPM包的途径
1. 发行的光盘或站点服务器，如
	- [http://mirrors.aliyun.com/](http://mirrors.aliyun.com/)
	- [http://mirrors.163.com/](http://mirrors.163.com/)
2. 项目的官网，很多开源项目都有自己的官网。
3. 第三方机构或个人制作并公开发布的rpm包，如
	- [http://rpmfind.net](http://rpmfind.net)
	- [http://rpm.pbone.net](http://rpm.pbone.net)
	- [https://fedoraproject.org/wiki/EPEL](https://fedoraproject.org/wiki/EPEL)

#### rpm的命令格式
- rpm [option…] PACKAGE_FILE

#### rpm包管理
- **打包**，制作rpm包较繁琐，此处略过。
- **安装**，选项如下
```
-i：安装指定包
-{v|vv|vvv}：显示详细安装信息，v字母越多显示越详细
-h：显示进度条
--nodeps：忽略依赖关系
--test：仅做测试，不执行安装
--replacepkgs：重新安装
-ivh：常用安装组合选项
```
- **卸载**，选项如下
```
-e：卸载包
--nodeps：忽略依赖关系
    如果被其它包所依赖：
    1、将依赖于此包的所有包一并卸载
    2、忽略依赖关系；能卸载，但依赖于此包程序包可能会运行不正常；
    3、如果包的配置文件安装后曾被改动过，卸载时，此文件将不会卸载，而是被重命名并保留
```

- **升级**，选项如下
```
-U：升级或安装
-F：纯升级，前提是得有老版本
--force：强制升级
```
- 已安装包**信息查看**
```
-q：查询包是否已安装
常用查询选项组合：
	-qa：列出所有已安装的包
	-qi：列出某已安装包的描述信息
	-ql：列出某已安装包生成了哪些文件
	-qc：列出某已安装包生成了哪些配置文件
	-qd：列出某已安装包生成了哪些帮助文件
	-qf：/path/to/some_file：查询某文件是由哪个包生成的
	-qs：查询某已安装包内文件的状态，有三种：normal、not installed、replaced
	-q --scripts package_name：列出某已安装包的相关脚本
		脚本有四类：
		preinstall：安装前脚本
		postinstall: 安装后脚本
		preuninstall: 卸载前脚本
		postuninstall: 卸载后脚本
```
- 未安装包**信息查看**
	- 方法与查看已安装包信息一样，只需要加上一个"p"选项即可，如：'-qpi'，即列出未安装包的描述信息

- **校验**，包制作者制作完成之后会附加数字签名于包上；包的制作者使用单向加密提取原始数据的特征码，而后使用自己的私钥加密这段特性码，附加原始数据后面。我们在安装不明来源的rpm包时需要验证包的来源合法性、完整性。
验证前提：必须有可靠机制获取到包制作者的公钥；
验证过程：
	- 使用制作者的公钥解密加密的特征码，能解密则意味着来源合法；
	- 使用与制作者同样的意向加密算法提取原始数据的特征码，并与解密出来的特征作比对，相同，则意味着完整性没问题；
```
rpm --import：导入公钥，导入后即可自动检查
rpm -qa gpg-pubkey：查看所有已导入的公钥
rpm -qi gpg-pubkey-NAME：显示密钥的详细信息
rpm --checksig package：手动检测包的合法性及完整性
rpm -K package：手动检测包的合法性及完整性
    rpm -K --nodigest：不检查包完整性
    rpm -K --nosignature：不检查来源合法性
rpm -V package_name
    S file Size differs
    M Mode differs (includes permissions and file type)
    5 digest (formerly MD5 sum) differs
    D Device major/minor number mismatch
    L readLink(2) path mismatch
    U User ownership differs
    G Group ownership differs
    T mTime differs
    P caPabilities differ
    某属性无变化，显示为"."
```

- **数据库管理**：路径为/var/lib/rpm，保存了所有包相关信息，如：包名、包组、依赖关系、公钥文件等
```
rpm –initdb：初始化，如果没有库，会新建一个，如果有则不重建
rpm –rebuiddb：重建数据库
```
