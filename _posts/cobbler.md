title: Cobbler的简单使用
date: 2016-05-24 12:44:10
tags:
- cobbler

categories: devops
---
###  一、 cobbler介绍
Cobbler是一个快速网络安装linux的服务，而且在经过调整也可以支持网络安装windows。该工具使用python开发，小巧轻便（才15k行python代码），使用简单的命令即可完成PXE网络安装环境的配置，同时还可以管理`DHCP`、`DNS`、以及`yum`仓库、构造系统ISO镜像。
Cobbler支持命令行管理，web界面管理，还提供了API接口，可以方便二次开发使用。
Cobbler客户端 **Koan** 支持虚拟机安装和操作系统重新安装，使重装系统更便捷。
 Cobbler提供以下服务集成： 
 
* PXE服务支持
* DHCP服务管理
* DNS服务管理
* 电源管理
* Kickstart服务支持
* yum仓库管理

<!--more-->
 cobbler模型
![Alt text](http://occl0vo6g.bkt.clouddn.com/cobbler.png)


### 二、cobbler安装说明

#### 2.1 Cobbler命令说明

``` python
命令                 用途
cobbler check       检查当前设置是否有问题
cobbler list        列出所有的cobbler元素
cobbler report      详细的列出各元素
cobbler sync        同步配置到dhcp/pxe和数据目录
cobbler reposync    同步yum仓库到本地
```


#### 2.2 Cobbler配置文件说明

Cobbler配置文件存放在   `/etc/cobbler` 下
``` python
目录名称                   用途
/etc/cobbler/settings     cobbler主配置文件
/etc/cobbler/             dhcp、dns、pxe、dnsmasq的模板配置文件

#如果cobbler不开启管理dhcp，cobbler check 检测时，不再对dhcp服务进行检测。
/etc/cobbler/users.digest     用于web创建配置的用户名密码文件
/etc/cobbler/modules.conf     模块配置文件
/etc/cobbler/users.conf       Cobbler WebUI/Web service授权配置文件
/var/www/cobbler             Cobbler安装镜像存储目录
```

安装系统数据目录`/var/www/cobbler`
导入的系统盘镜像和kickstart文件都放置在`/var/www/cobbler`
*确保 `/var` 目录有足够的空间来存储这些文件*


``` python
目录名称                                用途
/var/www/cobbler/images/               存储所有导入发行版的Kernel和initrd镜像用于远程网络启动
/var/www/cobbler/ks_mirror/            存储导入的发行版(安装系统的软件包)
/var/www/cobbler/repo_mirror/          yum repos存储目录(本次实验未使用)
/var/www/cobbler//var/log/cobbler      存放日志文件/var/log/cobbler/cobbler.log

```
  安装系统数据目录/var/lib/cobbler
Cobbler数据目录/var/lib/cobbler，此目录存储和Cobbler profiles、systems、distros相关的配置。
``` python
目录名称                       用途
/var/lib/cobbler/configs/     存储distros（镜像）、profiles ( 角色组 )、
/systems (角色中的具体主机) 和repos
/var/lib/cobblerbackup/       备份目录
/var/lib/cobblersnippets/     放置一些可以在kickstarts导入的脚本小片段
/var/lib/cobblertriggers/     放置一些可执行脚本
/var/lib/cobblerloaders/      放置同步到tftp目录下的引导文件
``` 

#### 2.3  cobbler安装

1. 添加EPEL源
``` python
wget http://mirrors.sohu.com/fedora-epel/5/i386/epel-release-5-4.noarch.rpm
rpm -ivh epel-release-5-4.noarch.rpm
```
2. 安装依赖库
``` python
yum -y install cobbler httpdxinetdtftp-server yum-util srsyn cdhcp cman PyYAML debmirror    cobbler-web(可选WEB_UI)  python-ctypes
``` 
3. 关闭Selinux及iptables防火墙否则Pxe会停滞在引导界面
``` python
#临时关闭
selinux:Setenfoce 0
#永久关闭selinux:
sed–i ‘/^SELINUX=/ s/=.*$/=disabled/g’ /etc/selinux/config
#关闭iptalbes
/etc/init.d/iptalbes stop   or   iptables –F
``` 
4. 配置`dhcpd.conf `文件
``` python
cat > /etc/dhcpd.conf<< EOF
ddns-update-style interim;
allow booting;
allow bootp;
ignore client-updates;
set vendorclass = option vendor-class-identifier;

subnet 192.168.2.0 netmask 255.255.255.0 {
#    option routers             192.168.11.2;
#    option domain-name-servers 8.8.8.8,8.8.4.4;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.2.100 192.168.2.254;
     filename                   "/pxelinux.0";
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                192.168.2.2;     #指定cobbler服务端
}
EOF 
``` 
5. 设置相关服务开机自启动
``` python
chkconfig --level 345 dhcpd on
chkconfig --level 345 httpd on
chkconfig --level 345 xinetd on
chkconfig --level 345 cobblerd on
/etc/init.d/httpdrestart
/etc/init.d/dhcpd restart
/etc/init.d/xinetd restart
/etc/init.d/cobblerd restart

[root@localhost ~]# /etc/init.d/httpd start
Starting httpd: Syntax error on line 10 of /etc/httpd/conf.d/cobbler.conf:
Invalid command 'WSGIScriptAliasMatch', perhaps misspelled or defined by a module not included in the server configuration
                                                           [FAILED]

#解决如下：
sed -i 's/#LoadModule/LoadModule/g ' /etc/httpd/conf.d/wsgi.conf  #启动apache wsgi动态模块
```
6.检查当前环境和配置文件是否有问题.运行会有如下提示
``` python
cobbler check 

The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
4 : change 'disable' to 'no' in /etc/xinetd.d/tftp
5 : change 'disable' to 'no' in /etc/xinetd.d/rsync
6 : comment 'dists' on /etc/debmirror.conf for proper debian support
7 : comment 'arches' on /etc/debmirror.conf for proper debian support
8 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "opensslpasswd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
Restart cobblerd and then run 'cobbler sync' to apply changes.
``` 
根据提示做出相应修改，例如：
1.  把server地址改为网卡IP ,否则kickstart功能无法实现
2.  把next_server地址设置为cobbler服务网卡地址
3.  使用cobbler get-loaders下载PXE启动组件
4.  启动xinetd管理tftp
5.  启动xinetd管理rsync
6.  注释’dists’,实现对debian系统安装的支持（debmirror包如果不安装debian系统，可以不装）
7.  注释’arches’,实现对debian系统安装的支持（debmirror包如果不安装debian系统，可以不装）
8.  修改获取tempble模板的密码
具体实现如下：
``` python
IP=$(ifconfig eth1|awk 'NR==2{print $2}'|cut -d: -f2)
PASS=$(opensslpasswd -1 -salt '' 'yourpasswd')
sed -i '/^server/ s/:.*$/: '$IP'/g' /etc/cobbler/settings
sed -i '/^next_server/ s/:.*$/: '$IP'/g' /etc/cobbler/settings
sed -i '/^default_password_crypted/ s/".*$/"'$PASS'"/g' /etc/cobbler/settings
sed -i '/disable.*$/ s/yes/no/g' /etc/xinetd.d/tftp
sed -i '/disable.*$/ s/yes/no/g' /etc/xinetd.d/rsync
sed -i 's!\(@dists\)!#\1!g;s!\(@arches\)!#\1!g' /etc/debmirror.conf
```

重启cobbler 服务，同步cobbler配置
``` python
/etc /init.d/cobblerd restart 
Cobbler sync
``` 

### 三、管理cobbler


#### 3.1 管理常用命令
``` python 
命令名称               命令用途
cobbler check      检查cobbler配置
cobbler list       列出所有的cobbler元素
cobbler report     列出元素的详细信息
cobbler distro     查看导入的发行版系统信息
cobbler system     查看添加的系统信息
cobbler profile    查看配置信息
cobbler sync       同步Cobbler配置，更改配置最好都要执行下
cobbler reposync   同步yum仓库
``` 
 
#### 3.2 导入安装源
- 光盘镜像
``` python
mkdir /sharkshow
mount -o loop /usr/local/src/GamewaveOS-0.4-x86_64.iso /sharkshow
cobbler import --path=/sharkshow --name=Gm4 --arch=x86_64
```
- 挂载光驱
``` python
mkdir /data/cdrom/
mount /dev/cdrom /data/cdrom/
cobbler import --path=/data/cdrom/ --name=Gm4 --arch=x86_64
``` 

- 网络下载系统镜像
``` python
mkdir /falcon
wget http://XXXX.com/CentOS-5.5-x86_64-bin-DVD-1of2.iso  #推荐到163.com镜像站下载
mount -o loop /root/CentOS-5.5-x86_64-bin-DVD-1of2.iso /falcon/
cobbler import --path=/falcon --name=centos5.5 --arch=x86_64
``` 
- 查看已有的元素情况
``` python 
cobbler distro report 
``` 


#### 3.3生成kickstat文件
**来源一**：利用`system-config-kickstart`工具生成（需要图形界面支持）
**来源二**：根本`anaconda-ks.cfg`稍候修改

``` python 
PASSWD='$1$$Is9qBBnXgI7v7pfSo4LxQ0'
cat > /usr/local/src/SKS.ks<<SKS
# GAMEWAVE Kickstart file
# This file is for a standard linux
# The OS is based on CentOS5.6_x86_64


install
text
url --url=\$tree
    #tree为distro指定的ksmeta路径，

\$SNIPPET('network_config')
lang en_US.UTF-8
keyboard us
skipx

timezone --utc Asia/Shanghai
rootpw --iscrypted $PASSWD
firewall --disable
firstboot --disable
authconfig --enableshadow --enablemd5
selinux --disable
bootloader --location=mbr
zerombr yes

# autopart
clearpart --all --initlabel
part /boot --fstype ext3 --size=100 --asprimary
part / --fstype ext3 --size=40960
part swap --size=4096
part /data --fstype ext3 --size=1 --growreboot




%pre
\$SNIPPET('pre_install_network_config')

%post
\$SNIPPET('post_install_network_config')

cat > /etc/redhat-release << EOF
CentOS release 5.6 (Final)
EOF


cat> /etc/issue << EOF



*******************************************************
  jayway’s PXE install  4.0 (Release) (Centos5.6)

*******************************************************

EOF




rm–rf /etc/yum.repo.d/*

cat> /etc/yum.repo.d/media.repo<<EOF
[media]
name=CentOS-$releasever - Media
baseurl=ftp://192.168.2.1/YUM/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-5
EOF

%packages
@base
@editors
ntp
dstat
sysstat
epel-release
rpmforge-release
lrzsz

SKS
```

pxe需要的文件同步到tftp目录中
``` python 
cobbler sync
```
此命令会将pxe需要的引导文件同步到`/tftpboot/`目录下
也会将cobbler的情况更新同步到各个配置文件中


#### 3.4定制客户端
``` python 
  cobbler system add \
--name=Gm4THEA \
--profile=Centos-x86_64 \    (cobbler import 时自动创建的profile)也可以手动添加
--mac-address=00:25:90:4A:47:65  \
--interface=eth1 \
--ip-address=192.168.11.241 \
--subnet=255.255.255.0 \
--hostname=AutoGamewave \
--name-servers=8.8.8.8 \
--kickstart=/usr/local/src/SKS.ks \
--static=1

cobbler system edit \
--name=Gm4THEA \
--mac-address=00:25:90:4A:47:64 \
--interface=eth0 \
--ip-address=208.81.165.241 \
--static=1


也可以根据此主机需求单独配置此ks文件 ( 推荐 )
#但是需要在WEBA.ks配置参数里增加cobbler调用的函数. 才能实现cobbler system命令操作具体主机
$SNIPPET('network_config')
%pre
\$SNIPPET('pre_install_network_config')
%post
\$SNIPPET('post_install_network_config')
```
也可以修改默认使用的模板` sample.ks.` 具体根据情况因个人改变
如果不想更改配文件置则需使用cobbler自带模板` /var/lib/cobbler/kickstart/sample.ks` 

#### 3.5 配置cobbler-web
Cobbler web界面是一个很好的前端，非常容易管理Cobbler
可以添加和删除 system distro profile 
可以查看、编辑distros, profiles, subprofiles, systems, repos 、kickstart文件

- 安装cobbler_web
``` python 
yum -y install cobbler-web 
```
- 设置用户名密码
``` python
#为已存在的用户cobbler重置密码
htdigest /etc/cobbler/users.digest "Cobbler" cobbler   
```
- 添加新用户
``` python
htdigest /etc/cobbler/users.digest "Cobbler" your_newname
``` 
- 配置cobbler web可以登录
``` python 
sed -i 's/authn_denyall/authn_configfile/g' /etc/cobbler/modules.conf
```
- 重启Cobbler与http
``` python 
/etc/init.d/cobblerd restart  
/etc/init.d/httpd restart 
```

- 访问Cobbler Web页面
 
浏览器访问登录页面**https://localhost/cobbler_web**

---- 
###  后记

本文档相对比较陈旧，可能与新版本`cobber`有出入，看客们还是以官网为主.


