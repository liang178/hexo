---
title: "LAMP运行环境搭建及部署php程序软件"
date: 2016-08-11
categories: LAMP
tags: [ lamp,apache,php ]
---

#### 前言
&emsp;&emsp;Linux+Apache+MySQL+PHP是大多数web服务器的运行环境，能熟练配置此运行环境是每个运维的必备基本技能，本文以安装MySQL管理工具PHPmyadmin以及博客软件wordpress为例，演示编译安装LAMP运行环境的过程。
#### 看图
![](http://obbrvy8yl.bkt.clouddn.com/20160811lamp.png)
#### 安装步骤
##### 编译安装apr、apr-util
httpd2.4版本依赖apr、apr-util这两个包，且版本要求高于系统自带包，需要编译安装
```
# tar xf apr-1.5.1.tar.bz2
# cd apr-1.5.1
# ./configure --prefix=/usr/local/apr
# make && make install

# tar xf apr-util-1.5.3.tar.bz2
# cd apr-util-1.5.3
# ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr
# make && make install
```
##### 编译安装httpd
```
# tar xf httpd-2.4.10.tar.bz2
# cd httpd-2.4.10
# ./configure --prefix=/usr/local/apache --sysconfdir=/etc/httpd24 --enable-so \
--enable-ssl --enable-cgi --enable-rewrite --with-zlib --with-pcre \
--with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util \
--enable-modules=most --enable-mpms-shared=all --with-mpm=event
# make && make install
# vim /etc/profile.d/httpd24.sh
export PATH=/usr/local/apache/bin:$PATH
# source /etc/profile.d/httpd24.sh
```
提供SysV服务脚本/etc/init.d/httpd24，可从httpd的RPM包复制一份，修改一下即可
```
# cp /etc/init.d/httpd /etc/init.d/httpd24
# vim /etc/init.d/httpd24
    ……
apachectl=/usr/local/apache/bin/apachectl
httpd=${HTTPD-/usr/local/apache/bin/httpd}
pidfile=${PIDFILE-/usr/local/apache/logs/httpd.pid}
 
    ……
# chkconfig --add httpd24
# chkconfig httpd24 on
```
##### 编译安装php，以fpm方式运行
```
# tar xf php-5.4.26.tar.bz2
# cd php-5.4.26
# ./configure --prefix=/usr/local/php5 --with-mysql=mysqlnd --with-openssl \
--with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --enable-mbstring --with-freetype-dir \
--with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir=/usr --enable-xml  \
--enable-sockets --enable-fpm --with-mcrypt  --with-config-file-path=/etc \
--with-config-file-scan-dir=/etc/php.d --with-bz2
# make && make intall

# cp sapi/fpm/init.d.php-fpm  /etc/rc.d/init.d/php-fpm
# chmod +x /etc/rc.d/init.d/php-fpm
# chkconfig --add php-fpm
# chkconfig php-fpm on
# cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
# vim /usr/local/php/etc/php-fpm.conf
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 2
pm.max_spare_servers = 8
pid = /usr/local/php/var/run/php-fpm.pid
```
##### 编译安装xcache，为php加速
```
# tar xf xcache-3.2.0.tar.gz
# cd xcache-3.2.0
# /usr/local/php5/bin/phpize
# ./configure --enable-xcache --with-php-config=/usr/local/php5/bin/php-config
# make && make install
# mkdir /etc/php.d
# cp xcache.ini /etc/php.d
# vim /etc/php.d/xcache.ini
extension = /usr/local/php5/lib/php/extensions/no-debug-non-zts-20121212/xcache.so
#如果有多条extension，确保该条在第一位
```
##### 以通用二进制方式安装MySQL
```
# useradd -r -M -s /sbin/nologin mysql
# mkdir -p /mydata/data
# tar xf mysql-5.6.26-linux-glibc2.5-x86_64.tar.gz -C /usr/local/
# cd /usr/local/
# ln -sv mysql-5.6.26-linux-glibc2.5-x86_64/ mysql
# cd mysql 
# chown -R mysql:mysql  ./
# scripts/mysql_install_db --user=mysql --datadir=/mydata/data
# chown -R root  ./
# cd /usr/local/mysql
# cp support-files/mysql.server  /etc/rc.d/init.d/mysqld
# chmod +x /etc/rc.d/init.d/mysqld
```
修改MySQL主配置文件
```
# mkdir /etc/mysql
# vim /etc/mysql/my.cnf
[mysqld]
datadir = /mydata/data
thread_concurrency = 4
#CPU个数*2
port = 3306
server_id = 2
socket = /tmp/mysql.sock
character_set_server = utf8
innodb_buffer_pool_size = 3G
max_allowed_packet = 64M
```
创建MySQL用户
```
# mysql -uroot -p -e "grant all on *.* to root@'%' identified by '123456';"
# mysql -uroot -p -e "grant all on *.* to testuser@'%' identified by '123456';"
# mysql -uroot -p -e "drop user ''@localhost;"
# mysql -uroot -p -e "drop user ''@localhost.localdomain;"
```
##### 写个测试页面，验证环境是否部署成功
```php
# vim /usr/local/apache/htdocs/index.php
<?php
  phpinfo();
  $link = mysql_connect('192.168.5.11','testuser','123456');
  if ($link)
    echo "Success...";
  else
    echo "Failure...";
  mysql_close();
?>
```
#### 搭建CA证书服务器
```
# (umask 077;openssl genrsa -out /etc/pki/CA/private/cakey.pem 2048)
# openssl req -new -x509 -key /etc/pki/CA/private/cakey.pem -out /etc/pki/CA/cacert.pem -days 3650
# touch /etc/pki/CA/{index.txt,serial}
# echo 01 > /etc/pki/CA/serial
```
#### 配置httpd服务器
- 生成证书请求文件，并发送给CA签署
```
# openssl genrsa -out /etc/httpd24/httpd.key 2048
# openssl req -new -key /etc/httpd24/httpd.key -out /etc/httpd24/httpd.csr
# scp /etc/httpd24/httpd.csr root@192.168.5.12:/tmp
```
- 登录CA服务器，签署证书并发回给Apache服务器
```
# openssl ca -in /tmp/httpd.csr -out /tmp/httpd.crt
# scp /tmp/httpd.crt root@192.168.5.10:/etc/httpd24/
```
- 修改httpd主配置文件，提供虚拟主机
```bash
# vim /etc/httpd24/httpd.conf
 
#DocumentRoot "/usr/local/apache/htdocs"
#Listen 80
#<Directory />
#   AllowOverride none
#   Require all denied
#</Directory>
AddType application/x-httpd-php  .php
AddType application/x-httpd-php-source  .phps
DirectoryIndex  index.php  index.html
Include /etc/httpd24/extra/httpd-vhosts.conf
 
LoadModule ssl_module modules/mod_ssl.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
```
```bash
# vim /etc/httpd24/extra/httpd-vhosts.conf
Listen 80
Listen 443
<VirtualHost *:443>
    ServerName test.phpmyadmin.com
    DocumentRoot "/www/pma"
    ErrorLog "logs/pma-error_log"
    CustomLog "logs/pma-access_log" common
    ProxyRequests Off
    ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/www/pma/$1
    SSLEngine on
    SSLCertificateFile "/etc/httpd24/httpd.crt"
    SSLCertificateKeyFile "/etc/httpd24/httpd.key"
</VirtualHost>
 
<VirtualHost *:80>
    ServerName test.wordpress.com
    DocumentRoot "/www/wordpress"
    ErrorLog "logs/wordpress-error_log"
    CustomLog "logs/wordpress-access_log" common
    ProxyRequests Off
    ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/www/wordpress/$1
</VirtualHost>
```
#### 安装PHPmyadmin和WordPress
把程序包解压到指定目录
```
# mkdir /www
# cd /www
# unzip /tmp/phpMyAdmin-4.0.5-all-languages.zip
# mv phpMyAdmin-4.0.5-all-languages/ pma
# tar xf /tmp/wordpress-4.3.1-zh_CN.tar.gz
```
修改phpmyadmin配置文件
```
# cp pma/config.sample.inc.php pma/config.inc.php
$cfg['Servers'][$i]['host'] = '192.168.5.10';
#把主机改成IP即可
```
修改wordpress配置文件
```
# cp wordpress/wp-config-sample.php wordpress/wp-config.php 
# vim wordpress/wp-config.php
// ** MySQL 设置 - 具体信息来自您正在使用的主机 ** //
/** WordPress数据库的名称 */
define('DB_NAME', 'wordpress');
 
/** MySQL数据库用户名 */
define('DB_USER', 'testuser');
 
/** MySQL数据库密码 */
define('DB_PASSWORD', '123456');
 
/** MySQL主机 */
define('DB_HOST', '192.168.5.11');
```
为wordpress创建数据库
```
# mysql -uroot -p -e "create database wordpress;"
```
#### 重启服务，访问测试
```
# service php-fpm restart
# service httpd24 restart
```
访问wordpress
![](http://obbrvy8yl.bkt.clouddn.com/20160811wordpress.png)
访问phpmyadmin
![](http://obbrvy8yl.bkt.clouddn.com/20160811phpadmin.png)
至此，整个环境部署完成
