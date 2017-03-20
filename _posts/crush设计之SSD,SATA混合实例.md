title: crush设计之SSD,SATA混合实例
date: 2017-03-14 13:44:10
tags:
 - crush
categories: ceph


### 前言

随着固态硬盘的成本大幅度降低，SSD已经“飞入寻常百姓家”了。SSD的高性能已经让不少应用享受了高IO的福利，ceph也不例外。以下将对SSD与Ceph结合的几种常见应用场景进行叙述与实战。



### 场景一：快慢存储方案

存储节点上即有SATA盘也有SSD盘，把各节点的SSD和SATA分别整合组成独立的存储池，为不同的应用供给不同性能的存储。
比如说常见的云环境中，虚拟机实例,对于实时数据IO性能要求高，并且都是热数据，可以把这部分的存储需求放入SSD的存储池里;而对于备份、快照等冷数据应用，相对IO性能需求比较低，因此可以放在普通的SATA盘组成的存储池里。




- 硬件环境

| 存储节点主机名      | Osd编号  |  对应的设备 |存储类型|
| --------   | :-----:  | :----:|----|
| storage-101      | osd.0 |    sdb  |SATA|
| storage-101         |  osd.1   |   sdc   |SSD|
| storage-102       |    osd.2    |  sdb  |SATA|
|storage-102 | osd.3 |　sdc |SSD|
| storage-103      | osd.4 |    sdb  |SATA
| storage-103         |   osd.5   |   sdc  |SSD|
| storage-104       |    osd.6    |  sdb  |SATA|
|storage-104 | osd.7 |sdc　 |SSD|
|storage-105 | osd.8 |sdb　 |SATA|
|storage-105 | osd.9 |sdc　 |SSD|
|storage-106 | osd.10 |sdc　 |SATA|
|storage-106 | osd.11 |sdc　 |SSD|

依据以上的硬件条件，我们把含有`SSD`的OSD聚合，并创建一个新的 `root` 层级（假如命名为ssd）并保留默认的层级关系，逻辑设计图如下所示

![crush设计图](http://i.imgur.com/oBDwgx2.png)

- 部署

利用`ceph-deploy`工具直接部署添加OSD。

    [root@ceph-deploy ceph-deploy]#ceph-deploy  osd  --zap-disk create  \
    storage-101:sdb  storage-101:sdc \
    storage-102:sdb  storage-102:sdc \
    storage-103:sdb  storage-103:sdc \
    storage-104:sdb  storage-104:sdc \
    storage-105:sdb  storage-105:sdc    
    [ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
    [ceph_deploy.cli][INFO  ] Invoked (1.5.24): /usr/bin/ceph-deploy osd --zap-disk create storage-102:sdb
    [ceph_deploy.osd][DEBUG ] Preparing cluster ceph disks storage-102:/dev/sdb:
    [ceph-deploy][DEBUG ] connected to host: storage-101
    [ceph-deploy][DEBUG ] detect platform information from remote host
    [ceph-deploy][DEBUG ] detect machine type
    [ceph_deploy.osd][INFO  ] Distro info: CentOS Linux 7.1.1503 Core
    [ceph_deploy.osd][DEBUG ] Deploying osd to storage-101
    ....
    以下忽略！>

- 获取当前的`crush map`

利用`ceph`工具获取当前`crushmap`

    [root@ceph-deploy ~]#ceph osd getcrushmap -o /tmp/mycrushmap
    got crush map from osdmap epoch 20

- 反编译`crushmap`

`/tmp/mycrushmap`是一个二制文件，需要通过`crushtool`反编译为文本文件

    [root@ceph-deploy ceph-deploy] crushtool -d /tmp/mycrushmap > /tmp/mycrushmap.txt

- 编辑`crushmap`文本文件

    1. 设置`SSD pool`的bucket入口，新建一个root层级，命名为`ssd`,并且把`SSD`设备的`OSD`移到里面（保留OSD所属的`host`层级）
    
              host storage-101 {
                id -9       # do not change unnecessarily  ## 设置唯一ID ## 
                # weight 0.180    ## 权重可以忽略,会依据osd自动计算##
                alg straw
                hash 0  # rjenkins1
                item osd.1 weight 0.090
              }


             host storage-102 {
                id -10      # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.3 weight 0.090
              }


             host storage-103 {
                id -11      # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.5 weight 0.090
              }

             host storage-104 {
                id -12      # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.7 weight 0.090
              }

             host storage-105 {
                id -13      # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.9 weight 0.090
              }

             host storage-106 {
                id -14      # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.11 weight 0.090
              }

              root ssd  {    ## 新建名为 ssd 的 root bucket , 作为后续SSD pool的入口 ## 
                id -8       # do not change unnecessarily
                # weight 0.720
                alg straw
                hash 0  # rjenkins1
                item storage-101 weight 0.090
                item storage-102 weight 0.090
                item storage-103 weight 0.090
                item storage-104 weight 0.090
                item storage-105 weight 0.090

             } 

   2. 把默认的default root bucket重命名为`sata`，并把`SATA `设备的`OSD`转移到`sata`里
   
    
              host storage-101 {
                id -2       # do not change unnecessarily  
                # weight 0.180    
                alg straw
                hash 0  # rjenkins1
                item osd.0 weight 0.090
              }


             host storage-102 {
                id -3       # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.2 weight 0.090
              }


             host storage-103 {
                id -4       # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.4 weight 0.090
              }

             host storage-104 {
                id -5       # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.6 weight 0.090
              }

             host storage-105 {
                id -6       # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.8 weight 0.090
              }

             host storage-106 {
                id -7       # do not change unnecessarily
                # weight 0.180
                alg straw
                hash 0  # rjenkins1
                item osd.10 weight 0.090
              }

              root sata  {    ## default 重命名为 sata  , 作为SATA pool的入口 ## 
                id -1       # do not change unnecessarily
                # weight 0.720
                alg straw
                hash 0  # rjenkins1
                item storage-101 weight 0.090
                item storage-102 weight 0.090
                item storage-103 weight 0.090
                item storage-104 weight 0.090
                item storage-105 weight 0.090

             }  


- 设置`rule`规则

    1. 为`SSD pool`添加`ssd rule`
    
            rule ssd {
            ruleset 1
            type replicated
            min_size 1
            max_size 10
            step take ssd  ## 指定入口为 ssd bucket 
            step choose firstn 0 type host
            step emit
            }

    2. 为`SATA pool`添加 `sata rule`
        
            rule sata {
            ruleset 0
            type replicated
            min_size 1
            max_size 10
            step take sata  ## 指定入口为 sata bucket 
            step choose firstn 0 type host
            step emit
            }


- 编译crushmap文本文件为二进制文件


        crushtool -c /tmp/mycrushmap.txt -o /tmp/mycrushmap.new
    

- 把新的`crushmap`应用于集群使用之生效


        ceph osd setcrushmap -i /tmp/mycrushmap.new 

- 查看`crush`结构

        [root@ceph-deploy ceph-deploy]# ceph osd tree 
          ID WEIGHT  TYPE NAME  UP/DOWN REWEIGHT PRIMARY-AFFINITY 
         -1 0.54000 root default 
         -2 0.18000  host storage-101   
          0 0.09000      osd.0   up  1.00000  1.00000 
          -3 0.18000  host storage-102  
           2 0.09000      osd.2   up  1.00000  1.00000 
          -4 0.18000  host storage-103   
           4 0.09000      osd.4   up  1.00000  1.00000 
          -5 0.18000  host storage-104   
           6 0.09000      osd.6   up  1.00000  1.00000 
          -6 0.18000  host storage-105  
           8 0.09000      osd.8   up  1.00000  1.00000 
          -7 0.18000  host storage-106  
          10 0.09000      osd.10   up  1.00000  1.00000 

           -8 0.54000 root ssd
           -9 0.09000  host storage-101  
            1 0.09000      osd.1   up  1.00000  1.00000   
          -10 0.09000  host storage-102  
            3 0.09000      osd.3   up  1.00000  1.00000    
          -11 0.09000  host storage-103   
            5 0.09000      osd.5   up  1.00000  1.00000   
          -12 0.09000  host storage-104  
            7 0.09000      osd.7   up  1.00000  1.00000   
          -13 0.09000  host storage-105  
            9 0.09000      osd.9   up  1.00000  1.00000   
          -14 0.08000  host storage-106  
           11 0.09000      osd.11   up  1.00000  1.00000  

- 创建 `SSD` 和 `SATA`存储池

        [root@ceph-deploy ceph-deploy]# ceph osd  pool   create   SSD 128 128 
        pool 'SSD' created
        [root@ceph-deploy ceph-deploy]# ceph osd  pool   create   SATA 128 128 
        pool 'SATA' created

- 为存储池指定何时的`rule`
        
        [root@ceph-deploy ceph-deploy]#ceph osd pool set SSD crush_ruleset 1
        [root@ceph-deploy ceph-deploy]#ceph osd pool set SATA crush_ruleset 0
        

- 查看`rule`是否生效

        [root@ceph-deploy ceph-deploy]# ceph osd dump | grep -Ei "ssd|sata" 
        pool 1 'SSD' replicated size 3 min_size 2 crush_ruleset 1 object_hash rjenkins pg_num 128 pgp_num 128 last_change 20 flags hashpspool stripe_width 0
        pool 2 'SATA' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 128 pgp_num 128 last_change 22 flags hashpspool stripe_width 0

- 查看`pg`分布是否正确

        [root@ceph-deploy ceph-deploy ]# ceph pg dump  | grep '^1\.' |  awk 'BEGIN{print "PG_id","\t","copy_set"}{print $1,"\t",$15}' | less
        dumped all in format plain
        PG_id    copy_set
        1.7e     [1,7,9]        ## SSD pool的PG都分配到了ssd rule所指向的OSD ##
        1.7f     [1,7,11]
        1.7c     [5,9,11]
        1.7d     [1,3,11]
        1.7a     [5,7,9]
        1.7b     [11,7,5]
        1.78     [1,3,11]
        1.79     [5,9,11]
        ......
    `SATA pool`的验证也是一样，在此不再一一展示。


以上是`SSD`和`SATA`设备分别组存储池的流程，结合云环境（比如说OpenStack),可以把 `SSD pool`给虚拟机实例使用，而`SATA pool`分拔给冷数据(如备份、快照等)。 

---

### 场景二：主备存储方案

设备配置同`场景一`，也是`SSD`和`SATA`设备混合，但不同于`场景一`组建独立的存储池分别为不不同的应用提供存储服务，而是依据`ceph`读写流程（如下图一所示），把主副本放在`SSD`组成的`bucket`里，其他副本放置在`SATA`设备上。如此，即可以在性能上得到一定的提升，也可以充分利用现有设备。

![pgIO](http://i.imgur.com/5VmM7io.png)

图一(来自官网)



主备存储方案的`crush`设计跟场景一（快慢存储方案)一样（逻辑结构见图二），只是在`rule`上面做些修改即可.
<center>![crush](http://i.imgur.com/oM1lZDL.png)</center>
<center>图二</center>

在此，我们直接沿用场景一中的`crush`（不再展示，回上详看），创建新的`crush rule`.

                
            rule pg {

            ruleset 3
            type replicated
            min_size 1
            max_size 10
            step take ssd  ## 指定入口为 ssd bucket 
            step choose firstn 1 type host  ## 从ssd bucket 搜索选一个合适的 osd 存储主副本
            step emit

            step take sata  ## 指定入口为 sata bucket 
            step choose firstn -1 type host  ## 从sata bucket 搜索其他副本所需要的 osd 来存储，隔离域依然为 host
            step emit
            }

编译、应用新的`crushmap`到集群里,并创建一个名为`pg`的存储池且指定 `pg rule`(步骤详见场景一所示）,验证`pg pool`的PG分布

        [root@ceph-deploy ceph-deploy ]# ceph pg dump  | grep '^3\.' |  awk 'BEGIN{print "PG_id","\t","copy_set"}{print $1,"\t",$15}' | less
        dumped all in format plain
        PG_id    copy_set
        3.2e     [1,2,8]      ## 主副都落在[1,3,5,7,9,11]的SSD的OSD上，其他副本都在SATA设备上 ##
        3.2f     [1,4,6]
        3.2c     [3,8,10]
        3.2d     [3,4,8]
        3.2a     [5,6,8]
        3.2b     [11,2,4]
        3.28     [7,8,10]
        3.29     [5,4,8]
        ......

---

### 外篇   
在ceph集群环境里，并不是所有的硬盘设备在性能和容量是完全一致的（如我们场景中的环境）。在高版本ceph中（0.80以后），可以通过调整 osd的 `primary affinity` (osd的亲和性）的值(0~1之间)，实现减小`OSD`设备承载客户端读写的负载。（从场景一中图1所示，客户端的读写主要发生在`primay osd`上）。亲和性的调整不会引起数据的变动。

因此，实现主备存储方案便有更简便的方法，即利用 osd的`primary affinity`值，只要把`SATA`设备对应OSD的 `primary affinity` 设置为`0`,那这些`osd`就不会成为主副本。那读写都只落到`SSD`设备对应的OSD上。

- 调整  `primary affinity`的值，首先要打开 `mon osd allow primary affinity`的开关，默认是关闭的。


        [root@storage-101 ~]#  ceph --admin-daemon /var/run/ceph/ceph-mon.*.asok config show | grep 'primary_affinity'
        "mon_osd_allow_primary_affinity": "false",  ## 默认值 ##

- 通过` ceph tell`实现参数地在线调整

        [root@ceph-deploy ceph-deploy]# ceph tell  mon.*  injectargs "--mon_osd_allow_primary_affinity=1"
        mon.storage-101: injectargs:mon_osd_allow_primary_affinity = 'true' 
        mon.storage-102: injectargs:mon_osd_allow_primary_affinity = 'true' 
        mon.storage-103: injectargs:mon_osd_allow_primary_affinity = 'true' 

- 在`ceph.conf`的`[mon]`作用域添加配置.

        mon osd allow primary affinity = true

- 把`SATA`设备对应的`osd`的`primary affinity`调整为`0` 

        [root@ceph-deploy ceph-deploy ]# for i  in 0 2 4 6 8 10;do ceph osd primary-affinity osd.$i 0 ;done
        set osd.0 primary-affinity to 0 (802)
        set osd.2 primary-affinity to 0 (802)
        set osd.4 primary-affinity to 0 (802)
        set osd.6 primary-affinity to 0 (802)
        set osd.8 primary-affinity to 0 (802)
        set osd.10 primary-affinity to 0 (802)

- 验证`SATA`设备的osd是否还在担任`primary osd`

        [root@ceph-deploy ceph-deploy ]# for i  in 0 2 4 6 8 10;do ceph pg dump | grep active+clean | egrep "\[$i," | wc -l ;done
        dumped all in format plain
        0
        dumped all in format plain
        0
        dumped all in format plain
        0
        dumped all in format plain
        0
        dumped all in format plain
        0
        dumped all in format plain
        0
从检验结果看，`SATA`设备的osd,已经没有再担任`primay osd`,设置有效。


**注意**
> 1. 在调整之前做好crushmap的备份，以防crush设置不当时能够及时复原
 
> 2. crush的设计应该在业务系统上线之前已经敲定，在线调整将面临极大的风险，必须慎之又慎

> 3. 有时候`reblance`未能达到完全收敛，可能需要设置下`tunable`值为`optimal`.即使用 `ceph osd crush tunable optimal`命令进行调整

> 4. 设置 `osd crush update on start` 为`false`，防止`osd`重启更新`crushmap`


        


