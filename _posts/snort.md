title: Snort学习笔记
date: 2016-05-24 09:44:10
tags:
- network
- security

categories: 网络
---
**简介**

> 在1998年，Martin Roesch先生用C语言开发了开放源代码(Open Source)的入侵检测系统Snort.直至今天，Snort已发展成为一个多平台(Multi-Platform),实时(Real-Time)流量分析，网络IP数据包(Pocket)记录等特性的强大的网络入侵检测/防御系统(Network Intrusion Detection/Prevention System),即NIDS/NIPS.Snort符合通用公共许可(GPL——GNU General Pubic License),在网上可以通过免费下载获得Snor

> snort有三种工作模式：嗅探器、数据包记录器、网络入侵检测系统。嗅探器模式仅仅是从网络上读取数据包并作为连续不断的流显示在终端上。数据包记录器 模式把数据包记录到硬盘上。网路入侵检测模式是最复杂的，而且是可配置的。我们可以让snort分析网络数据流以匹配用户定义的一些规则，并根据检测结果 采取一定的动作。

<!--more-->
**工作原理**

* 1: snort通过将网卡设置为混杂模式, 抓取TCP/IP 5层(数据链路层)数据包. 
* 2: 将捕获的数据包送到解码器进行解码; 
    `网络中的数据包有可能是各种包格式(以太网包, 令牌环包, TCP/IP包, 802.11包等), 这里解码器将其解码为Snort认识的统一格式`
* 3: 将解码后的数据包送到预处理器进行处理, 将分片的数据包进行重装
    `预处理主要通过插件来完成`
* 4: 根据规则对包进行检测
* 5: 将检测结果输出到文件,数据库或者Socket

###1: 部署
> 在[snort](https://www.snort.org/)的官网上既有各种系统部署文档, 这里以CentOS为例

#####1.1 安装包
<pre>
yum install https://www.snort.org/downloads/snort/daq-2.0.6-1.centos7.x86_64.rpm
yum install https://www.snort.org/downloads/snort/snort-2.9.7.6-1.centos7.x86_64.rpm
</pre>



###2: 基本使用
>部署完成后, 系统会多一个Snort的命令; 通过snort -h 可以看到详细的使用帮助
>
>后面介绍几个基本的用法
<pre>
[root@localhost ~]# snort -h
snort: option requires an argument -- 'h'
\
   ,,_     -\*> Snort! <*-
  o"  )~   Version 2.9.7.6 GRE (Build 285)
   ''''    By Martin Roesch & The Snort Team: http://www.snort.org/contact#team
           Copyright (C) 2014-2015 Cisco and/or its affiliates. All rights reserved.
           Copyright (C) 1998-2013 Sourcefire, Inc., et al.
           Using libpcap version 1.5.3
           Using PCRE version: 8.32 2012-11-30
           Using ZLIB version: 1.2.7
\
USAGE: snort [-options] <filter options>
...省略若干行
</pre>

####2.1 嗅探模式
> snort从网卡上读出数据包, 最终显示在屏幕上
<pre>
[root@localhost ~]# snort -v
Running in <font color="red">packet dump</font> mode
</pre>

* 通过加上-d可以看到应用层数据
* 通过加上-e可以显示数据链路层信息

####2.2 数据包记录模式
> 顾名思义, 改模式回把监听到的数据包记录下来, 通过加上-l指定日志目录进入数据包记录模式
> 在/var/log/snort/目录下会生成一个单独的文件来记录相关内容
<pre>
[root@localhost ~]# snort -v -l /var/log/snort/
Running in <font color="red">packet logging</font> mode
</pre>
> 会再指定的目录里生产一份单独的文件(文件名以时间戳结尾)把获取的内容记录下来
<pre>
[root@localhost snort]# cd /var/log/snort/
[root@localhost snort]# ls
snort.log.1444446978
</pre>

**其他实用选项**

* -h 监听网络 如: -h 192.168.1.0/24
* -b 以二进制格式(tcpdump使用的格式)记录内容(在网络很快是很有用)


**其他**

> snort在所有模式下都可以处理tcpdump格式的文件; 

如:
 
* 在嗅探器模式下: 
    snrot -dv -r packet.log
* 在日志包和入侵模式下:
   snort -dv -r packet.log icmp




####2.3 入侵检测模式
> snort的真正精华就在这里, 提供可以可自定义规则的网络报处理工具
> 使用中 首先要配置规则, 然后基于规则启动程序

#####2.3.1 添加规则
> 社区默认提供了一些默认规则
<pre>
wget https://www.snort.org/rules/community
tar -zxvf community -C /etc/snort/rules
</pre>

#####2.3.2 修改配置文件
> snort的默认配置文件为 <strong>/etc/snort/snort.conf<strong>

``` shell
diff --git a/snort.conf b/snort.conf_back
index 1d7c74e..66ce730 100644
--- a/snort.conf
+++ b/snort.conf_back
@@ -101,7 +101,7 @@ ipvar AIM_SERVERS [64.12.24.0/23,64.12.28.0/23,64.12.161.0/24,64.12.163.0/24,64.
 # Path to your rules files (this can be a relative path)
 # Note for Windows users:  You are advised to make this an absolute path,
 # such as:  c:\snort\rules
-var RULE_PATH rules
+var RULE_PATH /etc/snort/rules
 var SO_RULE_PATH ../so_rules
 var PREPROC_RULE_PATH ../preproc_rules

@@ -110,8 +110,8 @@ var PREPROC_RULE_PATH ../preproc_rules
 # not relative to snort.conf like the above variables
 # This is completely inconsistent with how other vars work, BUG 89986
 # Set the absolute path appropriately
-var WHITE_LIST_PATH rules
-var BLACK_LIST_PATH rules
+var WHITE_LIST_PATH ../rules
+var BLACK_LIST_PATH ../rules

 ###################################################
 # Step #2: Configure the decoder.  For more information, see README.decode
@@ -542,113 +542,112 @@ include reference.config
 ###################################################

 # site specific rules
-include $RULE_PATH/community-rules
-#include $RULE_PATH/local.rules
-
-#include $RULE_PATH/app-detect.rules
-#include $RULE_PATH/attack-responses.rules
-#include $RULE_PATH/backdoor.rules
-#include $RULE_PATH/bad-traffic.rules
-#include $RULE_PATH/blacklist.rules
-#include $RULE_PATH/botnet-cnc.rules
-#include $RULE_PATH/browser-chrome.rules
-#include $RULE_PATH/browser-firefox.rules
-#include $RULE_PATH/browser-ie.rules
-#include $RULE_PATH/browser-other.rules
-#include $RULE_PATH/browser-plugins.rules
-#include $RULE_PATH/browser-webkit.rules
-#include $RULE_PATH/chat.rules
-#include $RULE_PATH/content-replace.rules
-#include $RULE_PATH/ddos.rules
-#include $RULE_PATH/dns.rules
-#include $RULE_PATH/dos.rules
-#include $RULE_PATH/experimental.rules
-#include $RULE_PATH/exploit-kit.rules
-#include $RULE_PATH/exploit.rules
-#include $RULE_PATH/file-executable.rules
-#include $RULE_PATH/file-flash.rules
-#include $RULE_PATH/file-identify.rules
-#include $RULE_PATH/file-image.rules
-#include $RULE_PATH/file-multimedia.rules
-#include $RULE_PATH/file-office.rules
-#include $RULE_PATH/file-other.rules
-#include $RULE_PATH/file-pdf.rules
-#include $RULE_PATH/finger.rules
-#include $RULE_PATH/ftp.rules
-#include $RULE_PATH/icmp-info.rules
-#include $RULE_PATH/icmp.rules
-#include $RULE_PATH/imap.rules
-#include $RULE_PATH/indicator-compromise.rules
-#include $RULE_PATH/indicator-obfuscation.rules
-#include $RULE_PATH/indicator-shellcode.rules
-#include $RULE_PATH/info.rules
-#include $RULE_PATH/malware-backdoor.rules
-#include $RULE_PATH/malware-cnc.rules
-#include $RULE_PATH/malware-other.rules
-#include $RULE_PATH/malware-tools.rules
-#include $RULE_PATH/misc.rules
-#include $RULE_PATH/multimedia.rules
-#include $RULE_PATH/mysql.rules
-#include $RULE_PATH/netbios.rules
-#include $RULE_PATH/nntp.rules
-#include $RULE_PATH/oracle.rules
-#include $RULE_PATH/os-linux.rules
-#include $RULE_PATH/os-other.rules
-#include $RULE_PATH/os-solaris.rules
-#include $RULE_PATH/os-windows.rules
-#include $RULE_PATH/other-ids.rules
-#include $RULE_PATH/p2p.rules
-#include $RULE_PATH/phishing-spam.rules
-#include $RULE_PATH/policy-multimedia.rules
-#include $RULE_PATH/policy-other.rules
-#include $RULE_PATH/policy.rules
-#include $RULE_PATH/policy-social.rules
-#include $RULE_PATH/policy-spam.rules
-#include $RULE_PATH/pop2.rules
-#include $RULE_PATH/pop3.rules
-#include $RULE_PATH/protocol-finger.rules
-#include $RULE_PATH/protocol-ftp.rules
-#include $RULE_PATH/protocol-icmp.rules
-#include $RULE_PATH/protocol-imap.rules
-#include $RULE_PATH/protocol-pop.rules
-#include $RULE_PATH/protocol-services.rules
-#include $RULE_PATH/protocol-voip.rules
-#include $RULE_PATH/pua-adware.rules
-#include $RULE_PATH/pua-other.rules
-#include $RULE_PATH/pua-p2p.rules
-#include $RULE_PATH/pua-toolbars.rules
-#include $RULE_PATH/rpc.rules
-#include $RULE_PATH/rservices.rules
-#include $RULE_PATH/scada.rules
-#include $RULE_PATH/scan.rules
-#include $RULE_PATH/server-apache.rules
-#include $RULE_PATH/server-iis.rules
-#include $RULE_PATH/server-mail.rules
-#include $RULE_PATH/server-mssql.rules
-#include $RULE_PATH/server-mysql.rules
-#include $RULE_PATH/server-oracle.rules
-#include $RULE_PATH/server-other.rules
-#include $RULE_PATH/server-webapp.rules
-#include $RULE_PATH/shellcode.rules
-#include $RULE_PATH/smtp.rules
-#include $RULE_PATH/snmp.rules
-#include $RULE_PATH/specific-threats.rules
-#include $RULE_PATH/spyware-put.rules
-#include $RULE_PATH/sql.rules
-#include $RULE_PATH/telnet.rules
-#include $RULE_PATH/tftp.rules
-#include $RULE_PATH/virus.rules
-#include $RULE_PATH/voip.rules
-#include $RULE_PATH/web-activex.rules
-#include $RULE_PATH/web-attacks.rules
-#include $RULE_PATH/web-cgi.rules
-#include $RULE_PATH/web-client.rules
-#include $RULE_PATH/web-coldfusion.rules
-#include $RULE_PATH/web-frontpage.rules
-#include $RULE_PATH/web-iis.rules
-#include $RULE_PATH/web-misc.rules
-#include $RULE_PATH/web-php.rules
-#include $RULE_PATH/x11.rules
+include $RULE_PATH/local.rules
+
+include $RULE_PATH/app-detect.rules
+include $RULE_PATH/attack-responses.rules
+include $RULE_PATH/backdoor.rules
+include $RULE_PATH/bad-traffic.rules
+include $RULE_PATH/blacklist.rules
+include $RULE_PATH/botnet-cnc.rules
+include $RULE_PATH/browser-chrome.rules
+include $RULE_PATH/browser-firefox.rules
+include $RULE_PATH/browser-ie.rules
+include $RULE_PATH/browser-other.rules
+include $RULE_PATH/browser-plugins.rules
+include $RULE_PATH/browser-webkit.rules
+include $RULE_PATH/chat.rules
+include $RULE_PATH/content-replace.rules
+include $RULE_PATH/ddos.rules
+include $RULE_PATH/dns.rules
+include $RULE_PATH/dos.rules
+include $RULE_PATH/experimental.rules
+include $RULE_PATH/exploit-kit.rules
+include $RULE_PATH/exploit.rules
+include $RULE_PATH/file-executable.rules
+include $RULE_PATH/file-flash.rules
+include $RULE_PATH/file-identify.rules
+include $RULE_PATH/file-image.rules
+include $RULE_PATH/file-multimedia.rules
+include $RULE_PATH/file-office.rules
+include $RULE_PATH/file-other.rules
+include $RULE_PATH/file-pdf.rules
+include $RULE_PATH/finger.rules
+include $RULE_PATH/ftp.rules
+include $RULE_PATH/icmp-info.rules
+include $RULE_PATH/icmp.rules
+include $RULE_PATH/imap.rules
+include $RULE_PATH/indicator-compromise.rules
+include $RULE_PATH/indicator-obfuscation.rules
+include $RULE_PATH/indicator-shellcode.rules
+include $RULE_PATH/info.rules
+include $RULE_PATH/malware-backdoor.rules
+include $RULE_PATH/malware-cnc.rules
+include $RULE_PATH/malware-other.rules
+include $RULE_PATH/malware-tools.rules
+include $RULE_PATH/misc.rules
+include $RULE_PATH/multimedia.rules
+include $RULE_PATH/mysql.rules
+include $RULE_PATH/netbios.rules
+include $RULE_PATH/nntp.rules
+include $RULE_PATH/oracle.rules
+include $RULE_PATH/os-linux.rules
+include $RULE_PATH/os-other.rules
+include $RULE_PATH/os-solaris.rules
+include $RULE_PATH/os-windows.rules
+include $RULE_PATH/other-ids.rules
+include $RULE_PATH/p2p.rules
+include $RULE_PATH/phishing-spam.rules
+include $RULE_PATH/policy-multimedia.rules
+include $RULE_PATH/policy-other.rules
+include $RULE_PATH/policy.rules
+include $RULE_PATH/policy-social.rules
+include $RULE_PATH/policy-spam.rules
+include $RULE_PATH/pop2.rules
+include $RULE_PATH/pop3.rules
+include $RULE_PATH/protocol-finger.rules
+include $RULE_PATH/protocol-ftp.rules
+include $RULE_PATH/protocol-icmp.rules
+include $RULE_PATH/protocol-imap.rules
+include $RULE_PATH/protocol-pop.rules
+include $RULE_PATH/protocol-services.rules
+include $RULE_PATH/protocol-voip.rules
+include $RULE_PATH/pua-adware.rules
+include $RULE_PATH/pua-other.rules
+include $RULE_PATH/pua-p2p.rules
+include $RULE_PATH/pua-toolbars.rules
+include $RULE_PATH/rpc.rules
+include $RULE_PATH/rservices.rules
+include $RULE_PATH/scada.rules
+include $RULE_PATH/scan.rules
+include $RULE_PATH/server-apache.rules
+include $RULE_PATH/server-iis.rules
+include $RULE_PATH/server-mail.rules
+include $RULE_PATH/server-mssql.rules
+include $RULE_PATH/server-mysql.rules
+include $RULE_PATH/server-oracle.rules
+include $RULE_PATH/server-other.rules
+include $RULE_PATH/server-webapp.rules
+include $RULE_PATH/shellcode.rules
+include $RULE_PATH/smtp.rules
+include $RULE_PATH/snmp.rules
+include $RULE_PATH/specific-threats.rules
+include $RULE_PATH/spyware-put.rules
+include $RULE_PATH/sql.rules
+include $RULE_PATH/telnet.rules
+include $RULE_PATH/tftp.rules
+include $RULE_PATH/virus.rules
+include $RULE_PATH/voip.rules
+include $RULE_PATH/web-activex.rules
+include $RULE_PATH/web-attacks.rules
+include $RULE_PATH/web-cgi.rules
+include $RULE_PATH/web-client.rules
+include $RULE_PATH/web-coldfusion.rules
+include $RULE_PATH/web-frontpage.rules
+include $RULE_PATH/web-iis.rules
+include $RULE_PATH/web-misc.rules
+include $RULE_PATH/web-php.rules
+include $RULE_PATH/x11.rules

 ###################################################
 # Step #8: Customize your preprocessor and decoder alerts
``` 

>创建规则中制定的黑白名单(可以为空文件)
<pre>
[root@localhost snort]# cd /etc/snort/rules/
[root@localhost snort]# touch white\_list.rules
[root@localhost snort]# touch black_list.rules
[root@localhost snort]# chown -R snort:snort /etc/snort/rules
[root@localhost snort]# chmod -R 700 /etc/snort/rules
</pre>


> 创建配置文件中指定的动态规则目录
<pre>
[root@localhost ~]# mkdir -p /usr/local/lib/snort\_dynamicrules
[root@localhost ~]# chown -R snort:snort /usr/local/lib/snort\_dynamicrules
[root@localhost ~]# chmod -R 700 /usr/local/lib/snort_dynamicrules
</pre>

####2.4 启动服务
> snort默认即以入侵检测模式启动

<pre>
[root@localhost ~]# /etc/init.d/snortd start
Starting snortd (via systemctl):                           [  <font color="green">OK</font>  ]
[root@localhost ~]# ps aux | grep snort
snort    21316  0.1  0.4 422264 78132 ?        Ssl  10:11   0:00 /usr/sbin/snort -A fast -b -d -D -i eth0 -u snort -g snort -c /etc/snort/snort.conf -l /var/log/snort
</pre>

> 也可以自己在终端启动
<pre>
snort -d -l /var/log/snort/  -c /etc/snort/snort.conf
</pre>

###3: 规则编写
> 前面说过snort的真正意义就在于基于规则对包进行处理, 所以规则的定义与编写是在使用snort中最重要的([官方](https://www.snort.org/products)也是靠这个收钱,还挺贵); 下面我们看规则都能实现什么功能

*规则示例:*

`alert tcp any any -> 192.168.1.0/24 111 (content:"|00 01 86 a5|"; msg: "mountd access";)`

> `alert tcp any any -> 192.168.1.0/24 111` 
> 
> 为规则头,定义了一个包的who,where,what信息;以及瞒住规则定义时采取的动作

-
> `(content:"|00 01 86 a5|"; msg: "mountd access";)` 为规则选项(非必须,为了更加严格的限制触发条件)
> 
* content为关键字



_规则头由以下几部分组成:_

* 1: 规则满足时执行的动作
    * 1: Activation: 报警并启动另外一条Dynamic规则
    * 2: Dynamic: 等待被Activation调用, 被激活后作为一条Log规则
    * 3: Alert: 报警并记录
    * 4: Pass: 忽略
    * 5: Log: 仅记录
    * 6: Drop: 屏蔽并记录
    * 7: Reject: 屏蔽并记录; 如果是TCP在发送一个TCP重置的消息, 如果是UDP发送一个ICMP端口不可达
    * 8: Sdrop: 仅仅屏蔽
    
    *注意*
    在将来的版本中Activation和Dynamic规则将被功能增强的tagging所代替,
    
* 2: 协议
    * TCP
    * UDP
    * ICMP
* 3: IP地址
    * CIDR
    
    *注意*
    
        1: 可用any代表任意地址
        2: 可用!代表非该地址
        3: 多个IP地址可用`[]`包含, 并用`,`隔开 如[192.168.1.0/24, 10.0.0.0/24]
* 4: 来源端口号
    *注意*
    
        1: 可用any代表任意端口
        2: 用冒号代表范围, 如, 80到10000端口的 80:10000, 小于5000端口的 :500, 大于20000端口的 20000:
        3: 可用!代表非改端口
    
* 5: 方向操作符
    * -> 从左到有
    * <> 相互   

    *注意*
    
        1: 左边的IP地址和端口号为来源诸暨, 右边的为目标主机

_规则选项_
> 规则选项组成了snort入侵检测引擎的核心, 各个规则通过`;`隔开; 规则选项关键字与其对应的参数用`:`分开, 规则选项可又多个关键字组成

* msg - 在报警和包日志中打印一个消息。
* logto - 把包记录到用户指定的文件中而不是记录到标准输出。
* ttl - 检查ip头的ttl的值。
* tos 检查IP头中TOS字段的值。
* id - 检查ip头的分片id值。
* ipoption 查看IP选项字段的特定编码。
* fragbits 检查IP头的分段位。
* dsize - 检查包的净荷尺寸的值 。
* flags -检查tcp flags的值。
* seq - 检查tcp顺序号的值。
* ack - 检查tcp应答（acknowledgement）的值。
* window 测试TCP窗口域的特殊值。
* itype - 检查icmp type的值。
* icode - 检查icmp code的值。
* icmp_id - 检查ICMP ECHO ID的值。
* icmp_seq - 检查ICMP ECHO 顺序号的值。
* content - 在包的净荷中搜索指定的样式。
* content-list 在数据包载荷中搜索一个模式集合。
* offset - content选项的修饰符，设定开始搜索的位置 。
* depth - content选项的修饰符，设定搜索的最大深度。
* ocase - 指定对content字符串大小写不敏感。
* session - 记录指定会话的应用层信息的内容。
* rpc - 监视特定应用/进程调用的RPC服务。
* resp - 主动反应（切断连接等）。
* react - 响应动作（阻塞web站点）。
* reference - 外部攻击参考ids。
* sid - snort规则id。
* rev - 规则版本号。
* classtype - 规则类别标识。
* priority - 规则优先级标识号。
* uricontent - 在数据包的URI部分搜索一个内容。
* tag - 规则的高级记录行为。
* ip_proto - IP头的协议字段值。
* sameip - 判定源IP和目的IP是否相等。
* stateless - 忽略刘状态的有效性。
* regex - 通配符模式匹配。
* within - 强迫关系模式匹配所在的范围。
* byte_test - 数字模式匹配。
* byte_jump - 数字模式测试和偏移量调整。
        
**规则规范**

* Includes: 包含其他规则文件
* Variables: 定义变量
    * `var MY_NET 192.168.1.0/24`
    * 引用方式 `$MY_NET`

###4: 实际应用
1: 针对到本机22端口的数据报警

<pre>
[root@localhost rules]# cat /etc/snort/rules/local.rules
alert tcp any any -> any 22
</pre>

> 重启snort
<pre>
[root@localhost rules]# /etc/init.d/snortd restart
Restarting snortd (via systemctl):                         [  OK  ]
</pre>

-
> 效果

<pre>
[root@localhost rules]# tailf /var/log/snort/alert
10/12-10:59:31.447864  [**] [1:0:0] [**] [Priority: 0] {TCP} 124.127.138.35:50419 -> 42.51.161.228:22
10/12-10:59:31.544649  [**] [1:0:0] [**] [Priority: 0] {TCP} 124.127.138.35:50419 -> 42.51.161.228:22
10/12-10:59:31.549193  [**] [1:0:0] [**] [Priority: 0] {TCP} 124.127.138.35:50419 -> 42.51.161.228:22
</pre>

2: 针对到本机80端口的内容包含shell的包报警
> 规则
<pre>
alert tcp any any -> any 80 (content: "shell"; sid: 3; msg: "I am alert")
</pre>
> 然后重启snort

效果演示:
> 服务器节点执行如下命令, 开启80端口, 等待连接
<pre>
[root@localhost ~]# nc -l 80

</pre>
-
> 通过客户端连接到服务器
<pre>
telnet 192.168.161.228  80
pwd
shell
</pre>

-

> 可以看到, 只有在客户端输入shell的适合, log里才会多出一行, 输入其他命令是不会多的
<pre>
[root@localhost rules]# tailf /var/log/snort/alert
10/12-11:13:44.245573  [**] [1:3:0] I am alert [**] [Priority: 0] {TCP} 124.127.138.35:51934 -> 192.168.161.228:80
</pre>

*注意*
> 默认的报警方式是记录到log里, 这里无论是通过邮件报警, 或者通过这个文件最终实时通知到相关人员都是可行的





