title: cgroup的使用及在KVM中的应用
date: 2015-12-24 02:44:10
tags:
- cgroup

categories: KVM
---
### 1.cgroup简介 

  **[Cgroups](http://https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html-single/Resource_Management_Guide/index.html)**是control groups的缩写，是Linux内核提供的一种可以限制、记录、隔离进程组（process groups）所使用的物理资源（如：cpu,memory,IO等等）的机制.cgroup所属的RPM为libcgroup这个是被cgconfig服务所控制的。如果此服务没有启动，在根目录下的cgroup文件夹里就不会存在内容。默认情况下cgroup挂载目录为*/cgroup*,新的版本为`/dev/cgroup`. 

<!--more-->
    [root@localhost]#/etc/init.d/cgconfig status
    [root@localhost]#/etc/init.d/cgconfig stop
    [root@localhost]#ls /cgroup

启动服务之后，在`/cgroup`就会出现以下内容：

    [root@localhost cgroup]# /etc/init.d/cgconfig restart
    Stopping cgconfig service:                                 [  OK  ]
    Starting cgconfig service:                                 [  OK  ]
    [root@localhost cgroup]# pwd /cgroup
    [root@localhost cgroup]# ls
    blkio  cpu  cpuacct  cpuset  devices  freezer  memory  net_cls

### 2.cgroup模块


- 概念

*1.控制组（control group）*。控制组就是一组按照某种标准划分的进程。Cgroups中的资源控制都是以控制组为单位实现。一个进程可以加入到某个控制组，也从一个进程组迁移到另一个控制组。一个进程组的进程可以使用cgroups以控制组为单位分配的资源，同时受到cgroups以控制组为单位设定的限制。

*2.层级（hierarchy）*。控制组可以组织成hierarchical的形式，既一颗控制组树。控制组树上的子节点控制组是父节点控制组的孩子，继承父控制组的特定的属性。

*3.子系统（subsytem）*。一个子系统就是一个资源控制器，比如cpu子系统就是控制cpu时间分配的一个控制器。子系统必须附加（attach）到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。

Cgroup模型和Linux进程模型类似，子Cgroup可以继承父Cgroup的属性。两者区别就是Cgroup可以同时存在多个树状模型，而Linux进程只有一个树状模型。在RHEL6里一共有9个子系统
    
    Blkio – 物理块设备（磁盘、固态磁盘、USB等等）I/O访问控制
    Cpu –  CPU访问控制
    Cpuacct – CPU资源使用情况统计
    Cpuset – NUMA相关配置（那些CPU和那些内存节点被cgroup里面的任务执行）
    Devices – 设备访问控制
    Freezer – 挂起或者回复cgroup的任务
    Memory – cgroup任务内存使用限制赫尔内存使用情况统计报告
    Net_cls – 网络数据包标记分类，用于QOS管理
    Net_prio – 动态设备网络数据包优先级
    Ns – 名字服务

- 配置

在/etc/cgconfig.conf文件中，主要包含了两个主要类型：mount和group。mount是指创建以及挂载哪些层次为虚拟文件系统，并附上子系统的层次结构。

    mount {
    cpuset  = /cgroup/cpuset;
    cpu = /cgroup/cpu;
    cpuacct = /cgroup/cpuacct;
    memory  = /cgroup/memory;
    devices = /cgroup/devices;
    freezer = /cgroup/freezer;
    net_cls = /cgroup/net_cls;
    blkio   = /cgroup/blkio;
    }
其可以通过命令来实现，例如挂载cpuset子系统

    [root@susir /]# mount -t cgroup -o cpuset cpuset /cgroup/cpuset

- 常用查看命令

*lssubsys*：显示已经存在的子系统

    [root@localhost cgroup]# lssubsys -am
    ns
    perf_event
    net_prio
    cpuset /cgroup/cpuset
    cpu /cgroup/cpu
    cpuacct /cgroup/cpuacct
    memory /cgroup/memory
    devices /cgroup/devices
    freezer /cgroup/freezer
    net_cls /cgroup/net_cls
    blkio /cgroup/blkio

*lscgroup*:显示所有的cgroup

    [root@localhost cgroup]# lscgroup 
    cpuset:/libvirt
    cpuset:/libvirt/lxc
    cpuset:/libvirt/qemu
    cpuset:/libvirt/qemu/10.10.10.209
    cpuset:/libvirt/qemu/10.10.10.209/emulator
    ....
    cpu:/libvirt
    cpu:/libvirt/lxc
    cpu:/libvirt/qemu
    cpu:/libvirt/qemu/10.10.10.209
    cpu:/libvirt/qemu/10.10.10.209/emulator
    cpu:/libvirt/qemu/10.10.10.209/vcpu15
    ...
    memory:/
    memory:/libvirt
    memory:/libvirt/lxc
    memory:/libvirt/qemu
    memory:/libvirt/qemu/10.10.10.209
    memory:/libvirt/qemu/10.10.10.208
    ....


*cgsnapshot*:查前当前所有cgroup的配置
    
    root@localhost cgroup]#cgsnapshot -s 
    mount {
    cpuset = /cgroup/cpuset;
    cpu = /cgroup/cpu;
    cpuacct = /cgroup/cpuacct;
    memory = /cgroup/memory;
    devices = /cgroup/devices;
    freezer = /cgroup/freezer;
    net_cls = /cgroup/net_cls;
    blkio = /cgroup/blkio;
    }

    group libvirt {
    cpuset {
        cpuset.memory_spread_slab="0";
        cpuset.memory_spread_page="0";
        cpuset.memory_migrate="0";
        cpuset.sched_relax_domain_level="-1";
        cpuset.sched_load_balance="1";
        cpuset.mem_hardwall="0";
        cpuset.mem_exclusive="0";
        cpuset.cpu_exclusive="0";
        cpuset.mems="0-1";
        cpuset.cpus="0-31";
      }
    }
    .......
- 实例

***创建层级*:**有两种办法可以创建层次结构。第一种方法是，首先为层次结构创建一个挂载点，然后使用 mount 命令附加适当的子系统；第二种方法是，使用 cgconfig 服务。


  在第一种方法中，我使用以下命令创建一个名为 cpu-n-ram 的层次结构并附加 cpu、cpuset 和 memory 子系统：
    
    [root@localhost]mkdir /cgroup/cpu-n-ram
    [root@localhost]# mount -t cgroup -o cpu,cpuset,memory - /cgroup/cpu-n-ram

第二种方法，让 cgconfig 服务读取配置文件 /etc/cgconfig.conf。此文件中该挂载的对等项如下所示：

    mount {
    cpuset  = /cgroup/cpu-n-ram;
    cpu     = /cgroup/cpu-n-ram;
    memory  = /cgroup/cpu-n-ram;
    }

重新启动 cgconfig 服务将读取该配置文件并创建 /cgroup/cpu-n-ram 层次结构。

查看层级是否正确挂载

    [root@localhost cgroup]# lscgroup  
    cpuacct:/
    devices:/
    freezer:/
    net_cls:/
    blkio:/
    cpuset,cpu,memory:/

*创建cgroup控制组（cgcreate)*

 以下 cgcreate 命令将创建名为 group1-web 和 group2-db 的 cgroup：

    [root@localhost cgroup]# cgcreate -g cpuset:/group1-web
    [root@localhost cgroup]# cgcreate -g cpuset:/group2-db
    ################偶然间发现更快创建方法，直接在挂载点下面创建一个新目录就可以了################


*设置cgroup子系统参数（cgset)*

根据[NUMA](http://www.cnblogs.com/yubo/archive/2010/04/23/1718810.html)架构，分配CPU和内存节点。

    [root@localhost cgroup]# numactl  --hardware
    available: 2 nodes (0-1)
    node 0 cpus: 0 2 4 6 8 10 12 14 16 18 20 22 24 26 28 30
    node 0 size: 98258 MB          
    node 0 free: 86542 MB
    node 1 cpus: 1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31
    node 1 size: 98304 MB
    node 1 free: 90684 MB
    node distances:
    node   0   1 
     0:  10  20 
     1:  20  10 

    [root@localhost cgroup]# cgset -r cpuset.cpus='0 2 4 6 8 10 12 14 16 18 20 22 24 26 28 30' group1-web
    [root@localhost cgroup]# cgset -r cpuset.cpus='1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31' group2-db
    [root@localhost cgroup]# cgset -r cpuset.mems='0' group1-web
    [root@localhost cgroup]# cgset -r cpuset.mems='1' group2-db

*为 cgroup分配进程*

分配进程的方法以下几种：
    


cgclassify命令 

    [root@localhost cgroup]# cgclassify -g cpuset:group1-web 1683
    

把进程的PID写入到tasks文件里
        
    [root@localhost cgroup]# echo 1683 > /cgroup/cpu-n-ram/group1-web/tasks
        


根据cgroup的继承属性来实现,把当前的shell放入cgroup，后续由它生成的子进程会自动进入cgroup

    [root@localhost cgroup]#echo $$　>　/cgroup/cpu-n-ram/group1-web/tasks


在cgroup中启动被控制的进程

    [root@localhost cgroup]#cgexec -g cpuset:group1-web httpd



对于sysV服务来而言，可以编辑配置文件来实现。例如，在/etc/sysconfig/httpd中添加以下配置：

    [root@localhost cgroup]echo  'CGROUP_DAEMON="cpuset:group1-web"' >> /etc/sysconfig/httpd


- 小样测试

利用cgroup的devices子系统，限制对磁盘的访问。

挂载devices子系统到/cgroup/blkio,然后创创建bash Cgroup控制组

    [root@htuidc cgroup]# cgcreate  -g devices:/bash

确认磁盘的设备号，由以下可知，/dev/sda 的主设备号为 8，次设备号为 0

    [root@htuidc devices]# ll /dev/sda
    brw-rw----. 1 root disk 8, 0 Aug  2  2013 /dev/sda

设置对/dev/sda拒绝访问

    [root@htuidc devices]# cgset -r devices.deny='b 8:0 mrw' bash
    
为cgroup分配进程

    [root@htuidc devices]# echo $$ > /cgroup/devices/bash/tasks 

访问sda磁盘测试

    [root@htuidc devices]# dd if=/dev/sda of=/dev/null bs=512 count=1 
    dd: opening `/dev/sda': Operation not permitted

显然，devices子系统限制了对sda的访问

删除devices子系统

    [root@htuidc ~]# cgdelete   -r devices:/bash
    [root@htuidc ~]# dd if=/dev/sda of=/dev/null bs=512 count=1 
    1+0 records in
    1+0 records out
    512 bytes (512 B) copied, 2.643e-05 s, 19.4 MB/s

**更多参数的调节详见[cgroup](https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html-single/Resource_Management_Guide/index.html)的官方文档，在此不一一列举**

-------------

### 3.cgroup在KVM中的应用 ###


默认情况下，libvirt把会把KVM创建的虚拟机挂载上`cpuset,cpu,cpuacct,memroy,devices,freezer,blkior`这几个子系统
 
    [root@htuidc ~]# lscgroup 
    cpuset:/
    cpuset:/libvirt
    cpuset:/libvirt/lxc
    cpuset:/libvirt/qemu
    cpuset:/libvirt/qemu/10.10.10.209
    cpuset:/libvirt/qemu/10.10.10.209/emulator  #qemu-kvm主进程
    cpuset:/libvirt/qemu/10.10.10.209/vcpu15    #每个vcpu都有自己对应的cgroup控制组
    ....

    cpu:/
    cpu:/libvirt
    cpu:/libvirt/lxc
    cpu:/libvirt/qemu
    cpu:/libvirt/qemu/10.10.10.209
    cpu:/libvirt/qemu/10.10.10.209/emulator
    cpu:/libvirt/qemu/10.10.10.209/vcpu15
    ....
    cpuacct:/
    cpuacct:/libvirt
    cpuacct:/libvirt/lxc
    cpuacct:/libvirt/qemu
    cpuacct:/libvirt/qemu/10.10.10.209
    cpuacct:/libvirt/qemu/10.10.10.209/emulator
    cpuacct:/libvirt/qemu/10.10.10.209/vcpu15
    ...
    
    memory:/
    memory:/libvirt
    memory:/libvirt/lxc
    memory:/libvirt/qemu
    memory:/libvirt/qemu/10.10.10.209

    devices:/
    devices:/libvirt
    devices:/libvirt/lxc
    devices:/libvirt/qemu
    devices:/libvirt/qemu/10.10.10.209

    freezer:/
    freezer:/libvirt
    freezer:/libvirt/lxc
    freezer:/libvirt/qemu
    freezer:/libvirt/qemu/10.10.10.209

    net_cls:/
    blkio:/
    blkio:/libvirt
    blkio:/libvirt/lxc
    blkio:/libvirt/qemu
    blkio:/libvirt/qemu/10.10.10.209


**CPU的优化**
 



针对CPU的使用优化主要是通过vcpu的绑定来实现的，基于cache共享的原则。NUMA同区域的CPU共郭二级缓存，所以一般把虚拟机的vcpu绑定在同一区域。区域内的内存为本地，因此对内存进行绑定。

    [root@localhost cgroup]# for i in `seq 0 16 `; do cgset -r cpuset.cpus='1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31'  /libvirt/qemu/10.10.10.209/vcpu$i;done 
    [root@localhost cgroup]# for i in `seq 0 16 `;do cgset -r cpuset.mems='1'  /libvirt/qemu/10.10.10.209/vcpu$i;done 

查看设置情况下(以vcpu0为例,下同）

    [root@localhost ~]# cgget  -g cpuset  /libvirt/qemu/10.10.10.209/vcpu0 
    /libvirt/qemu/10.10.10.209/vcpu0:
    cpuset.memory_spread_slab: 0
    cpuset.memory_spread_page: 0
    cpuset.memory_pressure: 0
    cpuset.memory_migrate: 0
    cpuset.sched_relax_domain_level: -1
    cpuset.sched_load_balance: 1
    cpuset.mem_hardwall: 0
    cpuset.mem_exclusive: 0
    cpuset.cpu_exclusive: 0
    cpuset.mems: 1        #NUMA节点1 
    cpuset.cpus: 1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31 #NUMA节点所在CPU

以上是通过Cgroup手动来实现的，也可以通过libvirt的virsh命令来实现

    virsh # vcpupin  10.10.10.209  --vcpu 0  1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31
    virsh #  numatune  10.10.10.209 strict  1

以上可以通过在XML文件中定义实现

    <vcpu placement='static' cpuset='1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31'>16</vcpu>
    <numatune>
    <memory mode='strict' nodeset='1'/>
    </numatune>

**CPU资源限制**

对于虚拟机CPU资源的限制可以有从两方面入。1.设置权重值；2.限制CPU分配时间

    #以下是设置前的数值
    [root@htuidc ~]# cgget  -g cpu  /libvirt/qemu/10.10.10.204/vcpu0 
    /libvirt/qemu/10.10.10.204/vcpu0:
    cpu.rt_period_us: 1000000
    cpu.rt_runtime_us: 0
    cpu.stat: nr_periods 0
    nr_throttled 0             #达到限制的次数统计
    throttled_time 0                  
    cpu.cfs_period_us: 1000000 #CFS调度策略下，CPU分配的周期时间为1000000微秒 
    cpu.cfs_quota_us: -1      #此外表示CPU分配置时间不受限制
    cpu.shares: 1024   #默认都1024 
    
    #设置权值为2444，分配时间为50%
    [root@htuidc ~]# cgset   -r cpu.shares=2444  /libvirt/qemu/10.10.10.204/vcpu0   
    [root@htuidc ~]# cgset   -r cpu.cfs_quota_us=500000  /libvirt/qemu/10.10.10.204/vcpu0 

    #设置后的结果
    [root@htuidc ~]# cgget  -g cpu  /libvirt/qemu/10.10.10.204/vcpu0 
    /libvirt/qemu/10.10.10.204/vcpu0:
    cpu.rt_period_us: 1000000
    cpu.rt_runtime_us: 0
    cpu.stat: nr_periods 2
    nr_throttled 0
    throttled_time 0
    cpu.cfs_period_us: 1000000
    cpu.cfs_quota_us: 500000
    cpu.shares: 2444

在XML中的定义如下：
    
    <shares>2444</shares>
    <period>1000000</period>
    <quota>500000</quota>

**IO**

在IO方面,cgroup除了可以设置对磁盘的访问的权限之外，还可以设置权重值，IO读写速度的限制及IOPS的限制。
    
    #默认下blkio子系统的参数值
    [root@htuidc ~]#cgget -g blkio /libvirt/qemu/10.10.10.204
    ....
    blkio.throttle.write_iops_device: 
    blkio.throttle.read_iops_device: 
    blkio.throttle.write_bps_device: 
    blkio.throttle.read_bps_device: 
    ....
    blkio.weight: 500     #默认权重值，范围100~1000
    blkio.weight_device: 

    #通过cgset命令设置，当然也可以通过echo的方式直接写入
    [root@htuidc ~]#cgset -r  blkio.weight_device='8:0 999'  /libvirt/qemu/10.10.10.204
    [root@htuidc ~]#cgset -r blkio.throttle.write_iops_device="8:0 999" /libvirt/qemu/10.10.10.204

    [root@htuidc ~]# cgget -g blkio /libvirt/qemu/10.10.10.204/
    ...
    blkio.throttle.write_iops_device: 8:0   999
    blkio.throttle.read_iops_device: 
    blkio.throttle.write_bps_device: 
    blkio.throttle.read_bps_device: 
    ...
    blkio.weight: 500
    blkio.weight_device: 8:0    999


以上设置也可以在XML文件中定义（截取片段，详细见[libvirt](http://libvirt.org/formatdomain.html#elementsNUMATuning)文档）
        
      #IO限速设置
      <iotune>
        <total_bytes_sec>10000000</total_bytes_sec>
        <read_iops_sec>400000</read_iops_sec>
        <write_iops_sec>100000</write_iops_sec>
      </iotune>
      #权重值设置
      <blkiotune>
    <weight>500</weight>
    <device>
      <path>/dev/sda</path>
      <weight>999</weight>
    </device>
    </blkiotune>

**网络**

当前Cgroup没有直接提供可以调节的参数（就我目前所知，如果有人了解请告诉^~^),不过在libvirt里可以实现网络带宽的限制的，还能进行Qos(目前只进行了带宽限制的测试)。底层都是通过调用Tc来实现的。


    #查看虚拟机的网络配置
    virsh # domiflist  10.10.10.204 
    Interface  Type       Source     Model       MAC
    -------------------------------------------------------
    vnet2      bridge     public     virtio      52:54:00:07:1c:09
    vnet3      bridge     private    virtio      52:54:00:4d:a1:32
    
    #查看具体网卡的限制情况
    
    virsh # domiftune  10.10.10.204 --interface vnet2 
    inbound.average: 0 #入口流量平均值
    inbound.peak   : 0 #入口流量峰值
    inbound.burst  : 0 #入口流量爆发值s
    outbound.average: 0 #出口流量平均值
    outbound.peak  : 0  #出口流量峰值
    outbound.burst : 0  #出口流量爆值

    #设置进口限1MB/s,visrh的限制单位为kB/s
    virsh # domiftune  10.10.10.204 --interface vnet2 --inbound 1024 

**内存**

内存管理方面，cgroup可以控制内存的使用总量的限制，实现OOM。但推荐不要使用OOM。具体在此不做具体举例。优化方面，主要还是结合NUMA来控制分配。具体可参见**[Cgroups](http://https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html-single/Resource_Management_Guide/index.html)**官方文档。



**KVM优化总结**

![虚拟化设置](http://7xj51m.com1.z0.glb.clouddn.com/虚拟化设置-new.jpg)

**参考**

1.[Cgroup分析与应用](http://blog.csdn.net/haitaoliang/article/details/22092715)

2.[如何使用 CGroup 管理系统资源](http://www.oracle.com/technetwork/cn/articles/servers-storage-admin/resource-controllers-linux-1506602-zhs.html)

3.[Cgroup用法解析](http://bbs.chinaunix.net/thread-3574612-1-1.html)

