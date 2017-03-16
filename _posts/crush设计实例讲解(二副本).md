title: crush设计实例讲解（二副本）
date: 2017-03-15 13:44:10
tags:
 - crush
categories: ceph
---

## 前言

在一般的生产环境中为了保证数据的安全性及可靠性，常用三副本的方案来设计与实施。但有时候基于成本考虑及业务数据的可靠性不是很高的情况下，会考虑用`二副本`的方案来实现。以下，我们就以`二副本` 来设计与实施crush的方案。

---

- 硬件环境



| 存储节点主机名      | Osd编号  |  对应的设备 |
| --------   | :-----:  | :----:|
| storage-101      | osd.0 |    sdb  |
| storage-101         |  osd.1   |   sdc   |
| storage-102       |    osd.2    |  sdb  |
|storage-102 | osd.3 |　sdc |
| storage-103      | osd.4 |    sdb  |
| storage-103         |   osd.5   |   sdc  |
| storage-104       |    osd.6    |  sdb  |
|storage-104 | osd.7 |sdc　 |


    
我们为此添加 `rack `级的 bucket，分别包含两个 存储节点 （以`host`的bucket),然后以`rack`为隔离域，保证两个副本分别落在不同的rack上。
CRUSH的逻辑架构如下图所示
![crush](http://i.imgur.com/65wLZ1R.png)


- 部署

利用`ceph-deploy`工具快速部署添加`osd`.(注：ceph集群已经创建）

    [root@ceph-deploy ceph-deploy]# ceph-deploy  osd  --zap-disk create   storage-101:sdb storage-101:sdc storage-102:sdb storage-102:sdc \
    storage-103:sdb storage-103:sdc storage-104:sdb storage-104:sdc
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

成功添加`osd`之后，我们查看下`crush`的默认拓扑结构

再次查看集群的拓扑结构
    
    [root@ceph-deploy ceph-deploy]# ceph osd tree 
    ID WEIGHT  TYPE NAME  UP/DOWN REWEIGHT PRIMARY-AFFINITY 
    -1 0.72000 root default 
    -2 0.18000  host storage-101   
     0 0.09000      osd.0   up  1.00000  1.00000 
     1 0.09000      osd.1   up  1.00000  1.00000 
    -3 0.18000  host storage-102  
     2 0.09000      osd.2   up  1.00000  1.00000 
     3 0.09000      osd.3   up  1.00000  1.00000 
    -4 0.18000  host storage-103   
     4 0.09000      osd.4   up  1.00000  1.00000 
     5 0.09000      osd.5   up  1.00000  1.00000 
    -5 0.18000  host storage-104   
     6 0.09000      osd.6   up  1.00000  1.00000 
     7 0.09000      osd.7   up  1.00000  1.00000 

- 获取当前的`crush map`


利用`ceph`工具获取当前`crushmap`

    [root@ceph-deploy ~]#ceph osd getcrushmap -o /tmp/mycrushmap
    got crush map from osdmap epoch 14

- 反编译`crushmap`

`/tmp/mycrushmap`是一个二制文件，需要通过`crushtool`反编译为文本文件

    [root@ceph-deploy ceph-deploy] crushtool -d /tmp/mycrushmap > /tmp/mycrushmap.txt

- 查看crush

直接通过文件编辑工具（如`vim`)查看

    # begin crush map
    tunable choose_local_tries 0
    tunable choose_local_fallback_tries 0
    tunable choose_total_tries 50
    tunable chooseleaf_descend_once 1
    tunable straw_calc_version 1
    
    # devices
    device 0 osd.0
    device 1 osd.1
    device 2 osd.2
    device 3 osd.3
    device 4 osd.4
    device 5 osd.5
    device 6 osd.6
    device 7 osd.7
    
    # types              ##默认设置的bucket层级 ##
    type 0 osd
    type 1 host
    type 2 chassis
    type 3 rack
    type 4 row
    type 5 pdu
    type 6 pod
    type 7 room
    type 8 datacenter
    type 9 region
    type 10 root
    
    # buckets
    host storage-101 {                 ##Host 层级 ##
        id -2       # do not change unnecessarily
        # weight 0.180
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 0.090
        item osd.1 weight 0.090
    }

    host storage-102 {
        id -3       # do not change unnecessarily
        # weight 0.180
        alg straw
        hash 0  # rjenkins1
        item osd.2 weight 0.090
        item osd.3 weight 0.090
    }


    host storage-103 {
        id -4       # do not change unnecessarily
        # weight 0.180
        alg straw
        hash 0  # rjenkins1
        item osd.4 weight 0.090
        item osd.5 weight 0.090
    }


    host storage-104 {
        id -4       # do not change unnecessarily
        # weight 0.180
        alg straw
        hash 0  # rjenkins1
        item osd.6 weight 0.090
        item osd.7 weight 0.090
    }

    root default {
        id -1       # do not change unnecessarily
        # weight 0.720
        alg straw
        hash 0  # rjenkins1
        item storage-101 weight 0.180
        item storage-102 weight 0.180
        item storage-103 weight 0.180
        item storage-104 weight 0.180

    }
    
    # rules    ##默认的crush rule ##
    rule replicated_ruleset {   
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
    }
    
    # end crush map
    
- 编辑crush

 1.添加我们自定义的`rack` 层级，依据我们的逻辑图设计，并把原来的`host`级层的`storage-101`和`storage-102`划分到`rack-01`,`storage-103`和`storage-104`划分到`rack-02`

``` python
    rack  rack-01  { ## rack 层级 ##
        id -5       # do not change unnecessarily
        # weight 0.180
        alg straw
        hash 0  # rjenkins1
        item storage-101  weight 0.180
        item storage-102  weight 0.180
    }



    rack  rack-02  { ## rack 层级 ##
        id -6       # do not change unnecessarily
        # weight 0.180
        alg straw
        hash 0  # rjenkins1
        item storage-103  weight 0.180
        item storage-104  weight 0.180
    }
```
2.修改`root`级层，把`rack-01`和`rack-01` 添加到里面


    root default {
        id -1       # do not change unnecessarily
        # weight 0.720
        alg straw
        hash 0  # rjenkins1
        item rack-01 weight 0.360
        item rack-02 weight 0.360
    
    }

3.把默认的`host`层级 故障域修改为 `rack`层级

    rule replicated_ruleset {   
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type rack  ## 就是这里 ##
        step emit
    }

- 编译crushmap文本文件为二进制文件


        crushtool -c /tmp/mycrushmap.txt -o /tmp/mycrushmap.new
    

- 把新的`crushmap`应用于集群使用之生效


        ceph osd setcrushmap -i /tmp/mycrushmap.new 

当前crushmap应用之后，`PG`就会出现重新分布，此时带宽和IO就会出现增长。
    
**注意**
> 1. 在调整之前做好crushmap的备份，以防crush设置不当时能够及时复原
 
> 2. crush的设计应该在业务系统上线之前已经敲定，在线调整将面临极大的风险，必须慎之又慎

> 3. 副本数为二时，我们创建的pool的max_size应该修改为2，min_size为1 

> 4. 有时候`reblance`未能达到完全收敛，可能需要设置下`tunable`值为`optimal`.即使用 `ceph osd crush tunable optimal`调整



----

##  外篇

这里我们也点下如何通过命令行，实现在线修改`crushmap`，达到设计的效果

1. 添加`rack`层级： '**rack-01**'和'**rack-02**'

        ceph osd crush add-bucket    rack-01   rack
        ceph osd crush add-bucket    rack-01   rack

2. 把`rack-01`和`rack-02`转移到`root`下面

        ceph osd crush move rack-01    root=default
        ceph osd crush move rack-02    root=default

3. 把`host`转移到`rack`层级里

        ceph osd crush move  storage-101 rack=rack-01   root=default
        ceph osd crush move  storage-102 rack=rack-01   root=default
        ceph osd crush move  storage-103 rack=rack-02   root=default
        ceph osd crush move  storage-104 rack=rack-02   root=default

4. 创建以`rack`为隔离域的新 `rule` 
        
        ceph osd crush rule create-simple   newrule   default  rack  firstn

5. 为`pool`(假设命令为newpool)指定选用`newrule` （`ceph osd crush rule dump`可获取newrule的id） 


        ceph osd pool set  newpool crush_ruleset  1 
        ####此处 1 是指在rule 里  rule_id 设置的值####

        
## 后记

此文选自《Ceph分布式存储实战》，笔者也是联合作者之一        
        





