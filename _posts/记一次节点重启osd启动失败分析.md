title: 记一次节点重启osd启动失败分析
date: 2018-08-30 12:44:10
tags:
- ceph

categories: ceph
---



### 前言
因机房要断电维护，通知需要把所有机器都关机。等重新上电之后，发现有部分osd未能正常启动，本文对此次排障过程进行简单梳理。如果本文对读者起到一定的正面作用，不胜荣幸。若是相反，概不负责（^_^）

--- 

> 说明

- 本次集群的环境的`Hammer`

收到报警有部分osd已经`down`了，进入系统查看

```c
[root@openstack4 ~]# ceph -s
    cluster ebff3534-6293-4740-9259-11bfe4cca421
     health HEALTH_WARN
            452 pgs degraded
            447 pgs stuck unclean
            452 pgs undersized
            recovery 451824/4176546 objects degraded (10.818%)
            pool .rgw.buckets has too few pgs
            1/15 in osds are down
            noout,noscrub,nodeep-scrub flag(s) set
     monmap e1: 3 mons at {openstack4=192.168.10.168:6789/0,openstack5=192.168.10.169:6789/0,openstack6=192.168.10.212:6789/0}
            election epoch 310, quorum 0,1,2 openstack4,openstack5,openstack6
     osdmap e2346: 15 osds: 14 up, 15 in
            flags noout,noscrub,nodeep-scrub
      pgmap v3027288: 1536 pgs, 13 pools, 1208 GB data, 1359 kobjects
            2112 GB used, 10319 GB / 12431 GB avail
            451824/4176546 objects degraded (10.818%)
                1084 active+clean
                 452 active+undersized+degraded
  client io 0 B/s rd, 1655 B/s wr, 1 op/s
  ```
  
确认故障`osd`的位置
```c
[root@openstack4 ~]# ceph osd tree | grep down
ID WEIGHT  TYPE NAME               UP/DOWN REWEIGHT PRIMARY-AFFINITY
 2 1.00000         osd.2              down  1.00000          1.00000
 
 [root@openstack4 ~]# ceph osd find 2
{
    "osd": 2,
    "ip": "192.168.10.168:6806\/219136",
    "crush_location": {
        "host": "openstack4",
        "root": "sata"
    }
}
```
查看对应的osd的日志，没有发现异常。查看对应osd的挂载情况。
```c
[root@openstack4 ~]# df -Th
文件系统       类型      容量  已用  可用 已用% 挂载点
/dev/sdd2      ext4      1.1T  526G  518G   51% /
devtmpfs       devtmpfs   63G     0   63G    0% /dev
tmpfs          tmpfs      63G   12K   63G    1% /dev/shm
tmpfs          tmpfs      63G  2.3M   63G    1% /run
tmpfs          tmpfs      63G     0   63G    0% /sys/fs/cgroup
/dev/sdc1      xfs       427G   11G  417G    3% /var/lib/ceph/osd/ceph-14
/dev/sdb1      xfs       427G   14G  414G    4% /var/lib/ceph/osd/ceph-13
tmpfs          tmpfs      13G     0   13G    0% /run/user/0
/dev/sdf1      xfs       1.1T  205G  893G   19% /var/lib/ceph/osd/ceph-0
/dev/sdg1      xfs       1.1T  430G  668G   40% /var/lib/ceph/osd/ceph-1
```
故障的osd并没有正常挂载。如果尝试手动挂载能够正常，其实osd就能正常启动了。为了找到为什么不能自动挂载，那就得刨得深一点。

现在复盘下osd的启动流程
- 手动启动osd.2
```c
[root@openstack4 ~]# /etc/init.d/ceph start osd.2
/etc/init.d/ceph: osd.2 not found (/etc/ceph/ceph.conf defines mon.openstack4 osd.14 osd.1 osd.0 osd.13 , /var/lib/ceph defines mon.openstack4 osd.14 osd.1 osd.0 osd.13)
```
未发现`osd.2`的踪迹(原因不言自明，因为osd.2没有挂载)

为什么启动脚本没有找到osd.2? 带着疑问，一步一步深入下去

> 注意:
 H版本的ceph还是使用的`sysV`脚本来启动，后续版本已经都是systemd来管理


- `debug`启动脚本
```python
[root@openstack4 ~]# bash -x   /etc/init.d/ceph start osd.2
+ '[' -e /lib/lsb/init-functions ']'
+ . /lib/lsb/init-functions

.... <中间省略>....
+ test -f /usr/lib64/ceph/ceph_common.sh
+ . /usr/lib64/ceph/ceph_common.sh
.... <中间省略>....

+ local=' mon.openstack4 osd.14 osd.1 osd.0'
+ for i in '`find -L /var/lib/ceph/$type -mindepth 1 -maxdepth 1 -type d -printf '\''%f\n'\''`'
+ '[' -e /var/lib/ceph/osd/ceph-13/sysvinit ']'
++ sed 's/[^-]*-//'
++ echo ceph-13
+ id=13
+ local=' mon.openstack4 osd.14 osd.1 osd.0 osd.13'  // 看这里，如果本地osd挂载目录里有这个文件，则在local 变量里添加

+ '[' -e /var/lib/ceph/osd/ceph-2/sysvinit ']'//由于osd.2没有挂载，所以local变量没有osd.2

+ get_name_list osd.2 //接着使用get_name_list进行下一步处理
+ orig=osd.2
++ /usr/bin/ceph-conf -c /etc/ceph/ceph.conf -l mon
++ egrep -v '^mon$'
++ true
++ /usr/bin/ceph-conf -c /etc/ceph/ceph.conf -l mds
++ egrep -v '^mds$'
++ true
++ /usr/bin/ceph-conf -c /etc/ceph/ceph.conf -l osd
++ egrep -v '^osd$'
++ true
+ allconf=' mon.openstack4 osd.14 osd.1 osd.0 osd.13 '
+ '[' -z osd.2 ']'
+ what=
+ for f in '$orig'
++ echo osd.2
++ cut -c 1-3
+ type=osd
++ echo osd.2
++ cut -c 4-
++ sed 's/\.//'
+ id=2
+ case $f in
+ echo ' ' mon.openstack4 osd.14 osd.1 osd.0 osd.13 mon.openstack4 osd.14 osd.1 osd.0 osd.13 ' '
+ egrep -q '( osd2 | osd.2 )'
+ echo '/etc/init.d/ceph: osd.2 not found (/etc/ceph/ceph.conf defines' mon.openstack4 osd.14 osd.1 osd.0 osd.13 ', /var/lib/ceph defines' mon.openstack4 osd.14 osd.1 osd.0 'osd.13)'
/etc/init.d/ceph: osd.2 not found (/etc/ceph/ceph.conf defines mon.openstack4 osd.14 osd.1 osd.0 osd.13 , /var/lib/ceph defines mon.openstack4 osd.14 osd.1 osd.0 osd.13)
+ exit 1
```

从脚本debug的信息看，由于osd.2没有挂载，自然就找不到` /var/lib/ceph/osd/ceph-2/sysvinit`。在启动脚本里会执行
`get_name_list osd.2`进行进下处理。最终启动脚本报错也是在这个函数里。

> 注意:
>这个`get_name_list`函数并没有直接出现在启动脚本里，而通过`/usr/lib64/ceph/ceph_common.sh`引用过来的

- 找到函数位置
```python
get_name_list() {
    orig="$*"

    # extract list of monitors, mdss, osds defined in startup.conf
    allconf="$local "`$CCONF -c $conf -l mon | egrep -v '^mon$' || true ; \
	$CCONF -c $conf -l mds | egrep -v '^mds$' || true ; \
	$CCONF -c $conf -l osd | egrep -v '^osd$' || true`

    if [ -z "$orig" ]; then
	what="$allconf"
	return
    fi

    what=""
    for f in $orig; do
	type=`echo $f | cut -c 1-3`   # e.g. 'mon', if $item is 'mon1'
	id=`echo $f | cut -c 4- | sed 's/\\.//'`
	case $f in
	    mon | osd | mds)
		for d in $allconf; do
		    if echo $d | grep -q ^$type; then
			what="$what $d"
		    fi
		done
		;;
	    *)
		if ! echo " " $allconf $local " " | egrep -q "( $type$id | $type.$id )"; then
		    echo "$0: $type.$id not found ($conf defines" $allconf", /var/lib/ceph defines" $local")"   //就是这里，最终启动脚本报错的地儿
		    exit 1
		fi
		what="$what $f"
		;;
	esac
    done
}

```
找到祖坟了，因为`allconf`变量里没有找到`osd.2`，所以osd.2退出启动流程。

>小结：
>启动脚本对本次排查关系不是很大，主要还是梳理。

- 找到osd.2对应的物理盘
```python
[root@openstack4 ~]# ceph-disk list
/dev/sda other, unknown
/dev/sdb :
 /dev/sdb1 ceph data, active, cluster ceph, osd.13, journal /dev/sdb2
 /dev/sdb2 ceph journal, for /dev/sdb1
/dev/sdc :
 /dev/sdc1 ceph data, active, cluster ceph, osd.14, journal /dev/sdc2
 /dev/sdc2 ceph journal, for /dev/sdc1
/dev/sdd :
 /dev/sdd1 swap, swap
 /dev/sdd2 other, ext4, mounted on /
/dev/sde other, unknown
/dev/sdf :
 /dev/sdf1 ceph data, active, cluster ceph, osd.0, journal /dev/sdf2
 /dev/sdf2 ceph journal, for /dev/sdf1
/dev/sdg :
 /dev/sdg1 ceph data, active, cluster ceph, osd.1, journal /dev/sdg2
 /dev/sdg2 ceph journal, for /dev/sdg1
/dev/sdh :
 /dev/sdh1 ceph data, prepared, cluster ceph, osd.2, journal /dev/sdh2 ###对应osd.2的盘符在这里###
 /dev/sdh2 ceph journal, for /dev/sdh1
/dev/sdi other, unknown
/dev/sdj other, LVM2_member
/dev/sdk other, unknown
```
osd.2的对应盘符是`sdh`.强制激活试试
```c
[root@openstack4 ~]# /usr/sbin/ceph-disk-activate /dev/sdh1
ERROR:ceph-disk:Failed to activate
ceph-disk: Error: another ceph osd.2 already mounted in position (old/different cluster instance?); unmounting ours.
```
出现新的提示错误。查看`ceph-disk-activate`是什么类型的工具。
```python
[root@openstack4 ~]# file /usr/sbin/ceph-disk-activate
/usr/sbin/ceph-disk-activate: POSIX shell script, ASCII text executable
```
原来是个脚本，那就可以继续刨了。
```shell
[root@openstack4 ~]# cat  /usr/sbin/ceph-disk-activate
#!/bin/sh
dir=`dirname $0`
$dir/ceph-disk activate $*
```
好简洁，继续刨`ceph-disk`.`ceph-disk`依然是个脚本，利好消息。

```python
1953     try:
1954         (osd_id, cluster) = activate(path, activate_key_template, init)
1955
1956         # check if the disk is already active, or if something else is already
1957         # mounted there
1958         active = False
1959         other = False
1960         import pdb  ##插入pdb
1961         pdb.set_trace()  ##开始debug
1962         src_dev = os.stat(path).st_dev
1963         try:
1964             dst_dev = os.stat((STATEDIR + '/osd/{cluster}-{osd_id}').format(
1965                 cluster=cluster,
1966                 osd_id=osd_id)).st_dev
1967             if src_dev == dst_dev:
1968                 active = True
1969             else:
1970                 parent_dev = os.stat(STATEDIR + '/osd').st_dev
1971                 if dst_dev != parent_dev:
1972                     other = True
1973                 elif os.listdir(get_mount_point(cluster, osd_id)):
1974                     LOG.info(get_mount_point(cluster, osd_id) + " is not empty, won't override")
1975                     other = True
1976
1977         except OSError:
1978             pass
1979
1980         if active:
1981             LOG.info('%s osd.%s already mounted in position; unmounting ours.' % (cluster, osd_id))
1982             unmount(path)
1983         elif other:
1984             raise Error('another %s osd.%s already mounted in position (old/different cluster instance?); unmounting ours.' % (cluster, osd_id))    ## 找到报错的位置，可以抛祖坟了 ##
1985         else:
....<省略>
```
找到出错的位置，就开始debug了，找到出错根本原因。
```
[root@openstack4 ~]# ceph-disk activate /dev/sdh1
> /usr/sbin/ceph-disk(1962)mount_activate()
-> src_dev = os.stat(path).st_dev
(Pdb) l
1957 	        # mounted there
1958 	        active = False
1959 	        other = False
1960 	        import pdb
1961 	        pdb.set_trace()
1962 ->	        src_dev = os.stat(path).st_dev
1963 	        try:
1964 	            dst_dev = os.stat((STATEDIR + '/osd/{cluster}-{osd_id}').format(
1965 	                cluster=cluster,
1966 	                osd_id=osd_id)).st_dev
1967 	            if src_dev == dst_dev:
(Pdb) n
> /usr/sbin/ceph-disk(1963)mount_activate()
> (Pdb) a
dev = /dev/sdh1
activate_key_template = {statedir}/bootstrap-osd/{cluster}.keyring
init = auto
(Pdb) src_dev  ## 注意这个值  ##
2161L
(Pdb) path
'/var/lib/ceph/tmp/mnt.2G_Ku_'

```
>注意：
>关于pdb的使用，请自动google，在此不再一一说明

对`src_dev`和`path`进行关注。
`path`是个临时目录，其时此时osd对应盘已经持载了(间接也说明盘没有问题，可以暂时放心了）
```
/dev/sdh1      1149987184 426703244 723283940   38% /var/lib/ceph/tmp/mnt.2G_Ku_
```
重点看下这段代码
```
1964 ->	            dst_dev = os.stat((STATEDIR + '/osd/{cluster}-{osd_id}').format(
1965 	                cluster=cluster,
1966 	                osd_id=osd_id)).st_dev
1967 	            if src_dev == dst_dev:
1968 	                active = True
```
这里主要是用来对比`src_dev`和`dst_dev`.这个字段是取自`stat`的调用，表示的是设备编号
- dst_dev = 2161L
可以通过`stat`命令来获取
```
[root@openstack4 ~]# stat /var/lib/ceph/tmp/mnt.2G_Ku_
  文件："/var/lib/ceph/tmp/mnt.2G_Ku_"
  大小：287       	块：0          IO 块：4096   目录
设备：871h/2161d ##看这里##	Inode：16          硬链接：7
权限：(0755/drwxr-xr-x)  Uid：(    0/    root)   Gid：(    0/    root)
最近访问：1970-01-01 08:00:00.000000000 +0800
最近更改：2018-08-29 16:05:45.358453692 +0800
最近改动：2018-08-29 16:05:45.358453692 +0800
创建时间：-
```
从这里看，对上了吧。

继续debug，获取`src_dev`的值 
```python
-> osd_id=osd_id)).st_dev
(Pdb) n
> /usr/sbin/ceph-disk(1967)mount_activate()
-> if src_dev == dst_dev:
(Pdb) l
1962 	        src_dev = os.stat(path).st_dev
1963 	        try:
1964 	            dst_dev = os.stat((STATEDIR + '/osd/{cluster}-{osd_id}').format(
1965 	                cluster=cluster,
1966 	                osd_id=osd_id)).st_dev
1967 ->	            if src_dev == dst_dev:
1968 	                active = True
1969 	            else:
1970 	                parent_dev = os.stat(STATEDIR + '/osd').st_dev
1971 	                if dst_dev != parent_dev:
1972 	                    other = True
(Pdb) dst_dev  ## 获取到了 ##
2098L
```
这个`dst_dev`其实就是跟命令 `stat /var/lib/ceph/osd/ceph-2`获取是一样的
```
[root@openstack4 ~]# stat  /var/lib/ceph/osd/ceph-2/
  文件："/var/lib/ceph/osd/ceph-2/"
  大小：4096      	块：8          IO 块：4096   目录
设备：832h/2098d	Inode：38928733    硬链接：3
权限：(0755/drwxr-xr-x)  Uid：(    0/    root)   Gid：(    0/    root)
最近访问：2018-08-30 10:19:17.245489803 +0800
最近更改：2018-08-30 10:19:16.182492378 +0800
最近改动：2018-08-30 10:19:16.182492378 +0800
创建时间：-
```
这个设备`2098L`其实就是根设备而已。接着刨
```
1966 	                osd_id=osd_id)).st_dev
1967 	            if src_dev == dst_dev:
1968 	                active = True
1969 	            else:
1970 	                parent_dev = os.stat(STATEDIR + '/osd').st_dev
1971 ->	                if dst_dev != parent_dev:
1972 	                    other = True
(Pdb) dst_dev
2098L
(Pdb) parent_dev
2098L
```
代码里又获取了`parent_dev`的设备号，其实就是`/var/lib/ceph/osd`，这也在根下面。那就接下走

```python
> /usr/sbin/ceph-disk(1975)mount_activate()
-> other = True
(Pdb) l
1970 	                parent_dev = os.stat(STATEDIR + '/osd').st_dev
1971 	                if dst_dev != parent_dev:
1972 	                    other = True
1973 	                elif os.listdir(get_mount_point(cluster, osd_id)):
1974 	                    LOG.info(get_mount_point(cluster, osd_id) + " is not empty, won't override")
1975 ->	                    other = True
```
注意这个`other`变量，已经变成`true`.说明`os.listdir(get_mount_point(cluster, osd_id))`这个判断逻辑为真。来看这个做了什么。

```shell
(Pdb) get_mount_point(cluster, osd_id)
'/var/lib/ceph/osd/ceph-2'
(Pdb) os.listdir(get_mount_point(cluster, osd_id))
['s3_data']
(Pdb)
```
真相大白了，`os.listdir`会读取`/var/lib/ceph/osd/ceph-2/`下面有什么内容。如果有内容则`other`为真。接下看代码

```
-> raise Error('another %s osd.%s already mounted in position (old/different cluster instance?); unmounting ours.' % (cluster, osd_id))
(Pdb) l
1979
1980 	        if active:
1981 	            LOG.info('%s osd.%s already mounted in position; unmounting ours.' % (cluster, osd_id))
1982 	            unmount(path)
1983 	        elif other:
1984 ->	            raise Error('another %s osd.%s already mounted in position (old/different cluster instance?); unmounting ours.' % (cluster, osd_id))
1985 	        else:
1986 	            move_mount(
1987 	                dev=dev,
1988 	                path=path,
1989 	                cluster=cluster,
(Pdb) n
Error: Error('a... ours.',)
> /usr/sbin/ceph-disk(1984)mount_activate()
-> raise Error('another %s osd.%s already mounted in position (old/different cluster instance?); unmounting ours.' % (cluster, osd_id))
```
报错终于一览无余了。就是这为挂载目录下面有内容，导致挂载失败了.
```
[root@openstack4 ceph-2]# ll -l /var/lib/ceph/osd/ceph-2
总用量 4
drwxr-xr-x 2 root root 4096 8月  30 10:19 s3_data // 就是这个坑货
```
>复盘下挂载流程：
>- 1.把osd对应的盘挂载到临时目录
>- 2.先检测被挂载点是否已经被其他设备挂载
>- 3.再检测挂载点下面是否为空
>- 4.如果以上两个条件都没有满足，则报错

### 总结
坑是无处不在，但是只要细心一步一步跟踪，总到找到病灶。这就是开源的魅力。`Free SoftWare`不止是免费，更是自由。只要你愿意扎进去。 

###  延伸
关于osd是如何实现自动挂载的，可以参看下[磨磨的这篇blog](http://www.zphj1987.com/2016/12/22/Ceph%E6%95%B0%E6%8D%AE%E7%9B%98%E6%80%8E%E6%A0%B7%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%8A%A8%E6%8C%82%E8%BD%BD/)

我再简单梳理下：
- 1.系统启动时，内核加载设备
- 2.udev相关的规则执行
- 3.ceph-osd相关的规则会被引入(`/usr/lib/udev/rules.d/95-ceph-osd.rules`)
- 4.osd对应的磁盘的`add`事件会被触发
- 5.根据osd创建的时候打入磁盘分区特殊的code进行匹配
```
ACTION=="add" SUBSYSTEM=="block", \`     ENV{DEVTYPE}=="partition", \
  ENV{ID_PART_ENTRY_TYPE}=="4fbd7e29-9d25-41b8-afd0-35865ceff05d", \
  RUN+="/sbin/cryptsetup --key-file /etc/ceph/dmcrypt-keys/$env{ID_PART_ENTRY_UUID}.luks.key luksOpen /dev/$name $env{ID_PART_ENTRY_UUID}", \
  RUN+="/bin/bash -c 'while [ ! -e /dev/mapper/$env{ID_PART_ENTRY_UUID} ];do sleep 1; done'", \
  RUN+="/usr/sbin/ceph-disk-activate /dev/mapper/$env{ID_PART_ENTRY_UUID}"  ##匹配上就执行激活
  ```
  - 6.激活脚本包含挂载（上面的分析中），还包括启动osd
  - 7.流程顺畅的话，`ceph-osd`进程就可以常驻了

