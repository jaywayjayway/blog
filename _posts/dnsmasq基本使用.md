title: dnsmasq基本使用
date: 2017-01-24 09:44:10
tags:
categories: devops
---
内网需要做dns解析，但同时希望用最简单的办法，那么`dnsmasq`是最佳的方案。另外，它还可以用来做屏蔽或者dns缓存。

`dnsmasq`安装在此忽略，以下是最简单配置 (Centos 6 )
``` powershell
## /etc/dnsmasq.conf 的配置文件##
resolv-file=/etc/resolv.dnsmasq.conf ## 指定上层的dns服务器，可以从resolv.conf复制即可
# resolv.conf内的DNS寻址严格按照从上到下顺序执行，直到成功为止
strict-order
# DNS解析hosts时对应的hosts文件，对应no-hosts
no-hosts
cache-size=1024
addn-hosts=/etc/addion_hosts
# 多个IP用逗号分隔，192.168.x.x表示本机的ip地址，只有127.0.0.1的时候表示只有本机可以访问。
# 通过这个设置就可以实现同一局域网内的设备，通过把网络DNS设置为本机IP从而实现局域网范围内的DNS泛解析(注：无效IP有可能导至服务无法启动）
listen-address=127.0.0.1

```
`Centos7`配置如下

```powershell
resolv-file=/etc/resolv.dnsmasq.conf ## 指定上层的dns服务器，可以从resolv.conf复制即可
domain-needed
bogus-priv
strict-order
# line 55: add (query the specific domain name to the specific DNS server)
server=/google.com/8.8.8.8
# line 123: uncomment (add domain name automatically)
expand-hosts
# line 133: add (define domain name)
domain=s3.com    ##指定本地搜索的域 ##
addn-hosts=/etc/addion_hosts
address=/hello.me/127.0.0.1 ## 泛域名解析 ##
```
设置本地的dns服务器
```powershell
echo "nameserver 127.0.0.1"  > /etc/resolv.conf
```

添加需要解析的内网域名,格式跟本地的Host一样
```powershell

cat >> /etc/addion_hosts << EOF
100.100.100.1 node1 node2 node3
2.2.2.2  jayway
EOF

```
启动服务，验证结果,先验证我们内网的域名

```powershell
[root@localhost]# dig jayway

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6_8.2 <<>> jayway
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 308
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;jayway.                IN  A

;; ANSWER SECTION:
jayway.         0   IN  A   2.2.2.2 ## jayway的记录地址为2.2.2.2

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)    
;; WHEN: Thu Oct 27 18:41:18 2016
;; MSG SIZE  rcvd: 40

```

再来测试下外网解析的情况
```powershell

[root@localhost]# dig g.cn

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6_8.2 <<>> g.cn
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15358
;; flags: qr rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;g.cn.              IN  A

;; ANSWER SECTION:
g.cn.           209 IN  A   203.208.43.83   ##外网域名也正常解析了 ##
g.cn.           209 IN  A   203.208.43.82
g.cn.           209 IN  A   203.208.43.81
g.cn.           209 IN  A   203.208.43.80
g.cn.           209 IN  A   203.208.43.84

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Oct 27 18:42:33 2016
;; MSG SIZE  rcvd: 102
```
完美！！！！
