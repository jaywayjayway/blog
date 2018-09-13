title: 深入浅出BlueStore的OSD创建与启动
categories: ceph
tags: [ceph]
date: 2018-09-13 12:53:09
---

## 前言
之前[《记一次节点重启osd启动失败分析》](http://jaywaychou.com/2018/08/30/%E8%AE%B0%E4%B8%80%E6%AC%A1%E8%8A%82%E7%82%B9%E9%87%8D%E5%90%AFosd%E5%90%AF%E5%8A%A8%E5%A4%B1%E8%B4%A5%E5%88%86%E6%9E%90)有读者留言提到，现在都M版本了，已经是`Bluestore`的时代了，还在停留在H版本，有点落后。受此提醒，开始学习下Bluestore存储。本文主要对基于`ceph-volume`创建osd及osd的启动流程做下简单的梳理，希望对读者有所启示。

----
## 为什么需要`BlueStore`

Ceph作为软件定义存储(`SDS`)解决方案，其首要目标是保障存储数据的安全。为了达到数据安全的目的，Ceph使用了WAL的方式（`Write-Ahead-Log`)，这就是我们日常最熟悉的`journal`.

但是写前记录日志这种技术有一个主要缺陷就是它把你的硬盘性能降低到原来的二分之一（仅当日志和OSD数据共享同一个硬盘时），因为`filestore`在写数据前需要先写`journal`，所以有一倍的写放大。

`filestore`设计初衷就是就是为了充分发挥普通机械盘的性能，没有对SSD进行优化考虑。但随着SSD全面普及(主要性价比越来越实惠，也是新技术不断推陈出新的结果）,Ceph应用在`SSD`之上案例越来越多，对于性能的需求是更加迫切。基于以上现实，社区推出了`Bluestore`的存储引擎，剔除`journal`方案，缩减写放大，优化`SSD`写入，同时数据直写裸盘。

## `BlueStore`架构
![整体架构](http://www.sysnote.org/2016/08/19/ceph-bluestore/arch.png)

### 内部组件
- `RocksDB`:	存储预写式日志、数据对象元数据、Ceph的omap数据信息、以及分配器的元数据（分配器负责决定真正的数据应在什么地方存储）
- `BlueRocksEnv`:	与RocksDB交互的接口
- `BlueFS`:	迷你的文件系统（相对于`xfs，ext2/3/4`系列而言)，解决元数据、文件空间及磁盘空间的分配和管理。因为`rocksdb`一般是直接存储在POSIX兼容的文件系统（如ext3/xfs等）之上，但`BlueStore`引擎是直接面向裸盘管理，没有直接兼容POSIX的文件接口。但幸运的是，`rocksdb`的开发者充分考虑了适配性，只要实现了`rocksdb::Env` 接口，就能持久化`rocksdb`的数据存储(*包含RocksDB日志和sst文件*)。`BlueStore`就是为此而设计开发的，它只包含了最小的功能，用来承接`rocksdb`。在`osd`启动的时候，它会被`"mount"`起来，并完全载入内存
- `Allocator`: 用来从空闲空间分配`block`（block是可分配的最小单位)

> 说明：
> 1.对象数据存储部分即osd指定的`data`设备(可以是裸盘分区，或者lvm卷，下同)
> 2.RocksDB日志即osd指定的`wal`设备
> 3.RocksDB数据部分即osd指定的`db`设备
> 4.以上设备可以共用同一物理盘设备，也可以分开在不同的物理设备,这充分体现了ceph的灵活性

 以上只是本人粗糙的理解(未必完全正确或者跟实际有出入)，希望有大师出来指点一二。



---
##  部署实战


### 基础环境
```

[root@compute ~]# ceph -v
ceph version 13.2.0 (79a10589f1f80dfe21e8f9794365ed98143071c4) mimic (stable)
```


在创建osd之前，集群已经初始化完毕（即已经有Mon节点）。先熟悉下`ceph-volume`,可以看到当前支持`lvm`和`simple`两类子命令集。`lvm`用来创建osd，`simple`则是管理已经创建的osd。

```python
ceph-volume -h

... <中间省略>
Available subcommands:

lvm                      Use LVM and LVM-based technologies like dmcache to deploy OSDs
simple                   Manage already deployed OSDs with ceph-volume
```

预先创建好需要的lvm卷,如果没有独立的盘来单独存放wal和db,不推荐再在设备上面创建lvm卷分配给db和wal。默认只需要指定数据路径(--data)即可。Bluestore会自动管理所有的空间（包括data、db、wal)。

> 注意
> 如果有独立的盘来存放db，官方推荐db的空间不应小于数据空间的4%,以1T的数据空间为例，db的大小不应该小于40G.[官方连接](http://docs.ceph.com/docs/master/rados/configuration/bluestore-config-ref/)


### 创建lvm卷
以下创建的lvm卷只做演示用，data、db、wal分别创建独立的卷
```
[root@compute osd]# vgcreate   osd.3  /dev/sde
  Volume group "osd.3" successfully created

[root@compute osd]# lvcreate  -L 1G  -n  osd.3.db   osd.3
  Logical volume "osd.3.db" created.

[root@compute osd]# lvcreate  -L 1G  -n  osd.3.wal   osd.3
  Logical volume "osd.3.wal" created.

[root@compute osd]# lvcreate  -l 100%FREE -n osd.3.data osd.3
  Logical volume "osd.3.data" created.
```

### 创建osd
![image_1cn618mlp1eskvqdeov12o1vgj19.png-232.8kB][1]


从上面的截图其实可以把整个`create`流程分解为`prepare`和`activate`阶段，下面就开始庖丁解牛。

##### prepare 阶段
- 收集`keyring`
```
Running command: /bin/ceph-authtool --gen-print-key
```
- 创建新的`osd id`
```
Running command: /bin/ceph --cluster ceph --name client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring -i - osd new 9501a491-625f-4c2c-bf8e-a1a27cc0e6e5

```
- 挂载`tmpfs`目录，这不同于旧版本使用本地文件系统。`BlueStore`把这些依赖的文件信息都写入到了裸盘里（在osd启动之前，会重新生成）
```
Running command: /bin/mount -t tmpfs tmpfs
```
- data对应设备创建软链
```
Running command: /bin/ln -s /dev/osd.3/osd.3.data /var/lib/ceph/osd/ceph-3/block
```
- 获取当前集群的`monmap`
```
Running command: /bin/ceph --cluster ceph --name client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring mon getmap -o /var/lib/ceph/osd/ceph-3/activate.monmap
```
- 写入`keyring`文件并在集群注册
```
Running command: /bin/ceph-authtool /var/lib/ceph/osd/ceph-3/keyring --create-keyring --name osd.3 --add-key AQCy/Zdb4rTZIRAApGGe5QENqb/UPUHVRzI5Dw==
```
- `keyring`文件和工作目录以及设备的权限设置
```
Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-3/keyring
Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-3/
Running command: /bin/chown -R ceph:ceph /dev/dm-16
Running command: /bin/chown -R ceph:ceph /dev/dm-15
```
- `mkfs`初始化`bluestore`
```
Running command: /bin/ceph-osd --cluster ceph --osd-objectstore bluestore --mkfs -i 3 --monmap /var/lib/ceph/osd/ceph-3/activate.monmap --keyfile - --bluestore-block-wal-path /dev/osd.3/osd.3.wal --bluestore-block-db-path /dev/osd.3/osd.3.db --osd-data /var/lib/ceph/osd/ceph-3/ --osd-uuid 9501a491-625f-4c2c-bf8e-a1a27cc0e6e5 --setuser ceph --setgroup ceph
```
以上就是`prepare`的过程中涉及到的相关操作。


##### acitivate 阶段

-  从裸设备里直接获取启动`OSD`需要的相关元数据信息并写入到工作目录里(这些信息都存储在`BlueStore`的`label`里)
```
Running command: /bin/ceph-bluestore-tool --cluster=ceph prime-osd-dir --dev /dev/osd.3/osd.3.data --path /var/lib/ceph/osd/ceph-3
```
- 可以通过以下命令获取`label`信息
```
ceph-bluestore-tool  show-label --path /var/lib/ceph/osd/ceph-3
{
    "/var/lib/ceph/osd/ceph-3/block": {
        "osd_uuid": "9501a491-625f-4c2c-bf8e-a1a27cc0e6e5",
        "size": 2997882978304,
        "btime": "2018-09-11 17:39:00.380028",
        "description": "main",
        "bluefs": "1",
        "ceph_fsid": "a02fbd16-0db7-477d-8f16-a7d3cbfd8d73",
        "kv_backend": "rocksdb",
        "magic": "ceph osd volume v026",
        "mkfs_done": "yes",
        "osd_key": "AQCy/Zdb4rTZIRAApGGe5QENqb/UPUHVRzI5Dw==",
        "path_block.db": "/dev/osd.3/osd.3.db",
        "path_block.wal": "/dev/osd.3/osd.3.wal",
        "ready": "ready",
        "whoami": "3"
    },
    "/var/lib/ceph/osd/ceph-3/block.wal": {
        "osd_uuid": "9501a491-625f-4c2c-bf8e-a1a27cc0e6e5",
        "size": 1073741824,
        "btime": "2018-09-11 17:39:00.381666",
        "description": "bluefs wal"
    },
    "/var/lib/ceph/osd/ceph-3/block.db": {
        "osd_uuid": "9501a491-625f-4c2c-bf8e-a1a27cc0e6e5",
        "size": 1073741824,
        "btime": "2018-09-11 17:39:00.380983",
        "description": "bluefs db"
    }
}
```
- 创建设备文件软链并变更设备的所有者和组
```
Running command: /bin/ln -snf /dev/osd.3/osd.3.data /var/lib/ceph/osd/ceph-3/block
Running command: /bin/chown -R ceph:ceph /dev/dm-17
Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-3
Running command: /bin/ln -snf /dev/osd.3/osd.3.db /var/lib/ceph/osd/ceph-3/block.db
Running command: /bin/chown -R ceph:ceph /dev/dm-15
Running command: /bin/ln -snf /dev/osd.3/osd.3.wal /var/lib/ceph/osd/ceph-3/block.wal
Running command: /bin/chown -R ceph:ceph /dev/dm-16
```
- 注册系统服务(*稍后分析，继续往下看*）
```
Running command: /bin/systemctl enable ceph-volume@lvm-3-9501a491-625f-4c2c-bf8e-a1a27cc0e6e5
 stderr: Created symlink from /etc/systemd/system/multi-user.target.wants/ceph-volume@lvm-3-9501a491-625f-4c2c-bf8e-a1a27cc0e6e5.service to /usr/lib/systemd/system/ceph-volume@.service.
```

- 启动osd
```
Running command: /bin/systemctl start ceph-osd@3
--> ceph-volume lvm activate successful for osd ID: 3
```
`acitivate`阶段结束，`osd`进程就起来了

> 小结：
> 以上分解的osd创建步骤，为后续使用`ansible`来自动化部署`Ceph`集群至关重要。后续打算写个简单部署ceph的`playbook`，在此先`mark`下。

## 启动分析

BlueStore的OSD启动不同于老版本基于`udev`规则触发`ceph-disk`相关命令来启动，它依赖于`ceph-volume`相关服务与命令。

### ceph-volume 系统服务
上面可以看到，在创建osd的过程中有注册一个系统服务`ceph-volume@lvm-3-9501a491-625f-4c2c-bf8e-a1a27cc0e6e5`,

```
#cat /usr/lib/systemd/system/ceph-volume@.service
[Unit]
Description=Ceph Volume activation: %i
After=local-fs.target
Wants=local-fs.target

[Service]
Type=oneshot
KillMode=none
Environment=CEPH_VOLUME_TIMEOUT=10000
ExecStart=/bin/sh -c 'timeout $CEPH_VOLUME_TIMEOUT /usr/sbin/ceph-volume-systemd %i'
TimeoutSec=0

[Install]
WantedBy=multi-user.target
```
这个系统服务是在`local-fs`之后就执行了，优先级别是比较高的。这个服务传入的参数是`lvm-3-9501a491-625f-4c2c-bf8e-a1a27cc0e6e5`。其时可以分解为`lvm`、`3`、`9501a491-625f-4c2c-bf8e-a1a27cc0e6e5`.见明知意，分别是是lvm标签，osd的id,osd的uuid。
系统服务只有一个简单的脚本。
```
# cat /usr/sbin/ceph-volume-systemd
#!/usr/bin/python2.7

from ceph_volume.systemd import main

if __name__ == '__main__':
    main.main()
```
从上面看，已经开始进入ceph-volume项目了。
项目路径是`ceph/src/ceph-volume`
```
## ceph/src/ceph-volume/systemd/main
def main(args=None):
    """
    ... < 中间省略> 
        ## 转换为ceph-volume lvm 命令 继续执行 ## 
        ceph-volume lvm trigger 0-8715BEB4-15C5-49DE-BA6F-401086EC7B41

    """
    log.setup(name='ceph-volume-systemd.log', log_path='/var/log/ceph/ceph-volume-systemd.log')
    logger = logging.getLogger('systemd')
     ... < 中间省略> 
     ## 参数解析
    sub_command = parse_subcommand(suffix)
    extra_data = parse_extra_data(suffix)
    ... < 中间省略> 
    ## 构造成为['ceph-volume','lvm','trigger','id-uuid']的格式
    command = ['ceph-volume', sub_command, 'trigger', extra_data]
    tries = os.environ.get('CEPH_VOLUME_SYSTEMD_TRIES', 30)
    interval = os.environ.get('CEPH_VOLUME_SYSTEMD_INTERVAL', 5)
    ... < 中间省略> 
            process.run(command, terminal_logging=False) ##下一步执行入口## 

```
这里只是对传入的参数进行处理，最后调用`ceph-volume lvm trigger` 

### `ceph-volume lvm trigger` 处理
```
#cat  /usr/sbin/ceph-volume
#!/usr/bin/python2.7

from ceph_volume import main

if __name__ == '__main__':
    main.Volume()
```

下面进入`ceph/src/ceph-volume/main`，找到入口函数`main()`
```
    def main(self, argv):
        # these need to be available for the help, which gets parsed super
        # early
        ## 一些预备处理，比如日志，ceph配置路径等
        self.load_ceph_conf_path()
        self.load_log_path()
        self.enable_plugins()
        main_args, subcommand_args = self._get_split_args()
        ...<中间省略>
        terminal.dispatch(self.mapper, subcommand_args)
```
最终进入的是`terminal.dispatch`，核心是传入的参数。
以下是`pdb`的结果，其实这是一个工厂模式的设计实现，根据`subcommand_args`里类型，创建对应的实例，通过实例方法实现功能处理。
![image_1cn6k8a3e1f754eu15armjhcgj9.png-25.6kB][2]

```
## ceph/src/ceph-volume/ceph_volume/terminal.py
def dispatch(mapper, argv=None):
    argv = argv or sys.argv
    for count, arg in enumerate(argv, 1):
        if arg in mapper.keys():
            ## 创建实例
            instance = mapper.get(arg)(argv[count:])
            if hasattr(instance, 'main'):
                ## 实例main方法
                instance.main()
                raise SystemExit(0)
                
```
显然根据传入的参数创建了`LVM`的实例，然后进入实例的`main`方法


```
  ## ceph/src/ceph-volume/ceph_volume/devices/lvm/main.py
 class LVM(object):   
        mapper = {
        'activate': activate.Activate,
        'batch': batch.Batch,
        'prepare': prepare.Prepare,
        'create': create.Create,
        'trigger': trigger.Trigger,  ### 关注这里### 
        'list': listing.List,
        'zap': zap.Zap,
    }
    def main(self):
        terminal.dispatch(self.mapper, self.argv)
        ...<中间省略>
```
跟前面如出一辙，根据传入的参数对应进入`trigger.Trigger`实例的`main`方法
```
## ceph/src/ceph-volume/ceph_volume/devices/lvm/trigger.py
class Trigger(object):

    help = 'systemd helper to activate an OSD'

    def __init__(self, argv):
        self.argv = argv

    @decorators.needs_root
    def main(self):
          ... <中间省略>
        ## 解析参数
        args = parser.parse_args(self.argv)
        ## 检验osd id
        osd_id = parse_osd_id(args.systemd_data)
        ## 检验uuid
        osd_uuid = parse_osd_uuid(args.systemd_data)
        ## 再次跳转到Activate实例main方法
        Activate(['--auto-detect-objectstore', osd_id, osd_uuid]).main()
```
以下是`Activate`类的`main`方法
```
    ##ceph/src/ceph-volume/ceph_volume/devices/lvm/activate.py
    def main(self):
        sub_command_help = dedent("""
        ...<中间省略>
        )
        ## 参数解析
        args = parser.parse_args(self.argv)
        ## 如果不指定是bluestore或者filestore,默认按bluestore来处理
        if not args.bluestore and not args.filestore:
            args.bluestore = True
        if args.activate_all:
            self.activate_all(args)
        else:
            self.activate(args)
```
最后通过实例的`activate`方法来激活

```
 ##ceph/src/ceph-volume/ceph_volume/devices/lvm/activate.py
 def activate(self, args, osd_id=None, osd_fsid=None):
         ...<中间省略>
        ## 获取本地所有的lvm卷信息
        lvs = api.Volumes()
        ...<中间省略>
            for lv in lvs:
                    ....
        ## 通过lvm的tags来判断传入的osd id跟uuid是否已经存在，这些tags是在创建osd的时候写入（请继续往下看^_^）             
            return activate_bluestore(lvs)
        if args.bluestore:
            activate_bluestore(lvs, no_systemd=args.no_systemd)
        elif args.filestore:
            activate_filestore(lvs, no_systemd=args.no_systemd)
            
```
BlueStore的osd则继续跳转到`activate_bluestore`

```
 ##ceph/src/ceph-volume/ceph_volume/devices/lvm/activate.py
def activate_bluestore(lvs, no_systemd=False):
    # 从tags里找osd相关id、fsid、cluster_name等信息
    osd_lv = lvs.get(lv_tags={'ceph.type': 'block'})
    if not osd_lv:
        raise RuntimeError('could not find a bluestore OSD to activate')
    is_encrypted = osd_lv.tags.get('ceph.encrypted', '0') == '1'
    dmcrypt_secret = None
    osd_id = osd_lv.tags['ceph.osd_id']
    conf.cluster = osd_lv.tags['ceph.cluster_name']
    osd_fsid = osd_lv.tags['ceph.osd_fsid']

    osd_path = '/var/lib/ceph/osd/%s-%s' % (conf.cluster, osd_id)
    ### 有没有很熟悉，其时就是前面分析的activate阶段的相关命令###
    if not system.path_is_mounted(osd_path):
        # ## 创建工作目录并挂载tmpfs 
        prepare_utils.create_osd_path(osd_id, tmpfs=True)
    ## 构建data、db、wal的软链路径
    for link_name in ['block', 'block.db', 'block.wal']:
        link_path = os.path.join(osd_path, link_name)
        if os.path.exists(link_path):
            os.unlink(os.path.join(osd_path, link_name))
    ## 获取设备路径 ##
    db_device_path = get_osd_device_path(osd_lv, lvs, 'db', dmcrypt_secret=dmcrypt_secret)
    wal_device_path = get_osd_device_path(osd_lv, lvs, 'wal', dmcrypt_secret=dmcrypt_secret)

    ### 看这里！看这里！看这里！重要的事情说三遍，就是调用ceph-bluestore-tool工具读出元数据并写入工作目录，一模一样，下面都一样了。很熟悉了吧  ^__^
    process.run([
        'ceph-bluestore-tool', '--cluster=%s' % conf.cluster,
        'prime-osd-dir', '--dev', osd_lv_path,
        '--path', osd_path])

    process.run(['ln', '-snf', osd_lv_path, os.path.join(osd_path, 'block')])
    ## 所有者修改以及创建软连
    system.chown(os.path.join(osd_path, 'block'))
    system.chown(osd_path)
    if db_device_path:
        destination = os.path.join(osd_path, 'block.db')
        process.run(['ln', '-snf', db_device_path, destination])
        system.chown(db_device_path)
    if wal_device_path:
        destination = os.path.join(osd_path, 'block.wal')
        process.run(['ln', '-snf', wal_device_path, destination])
        system.chown(wal_device_path)

    if no_systemd is False:
        # enable the ceph-volume unit for this OSD
        systemctl.enable_volume(osd_id, osd_fsid, 'lvm')

        # 启动osd进程
        systemctl.start_osd(osd_id)
    terminal.success("ceph-volume lvm activate successful for osd ID: %s" % osd_id)
```
到此，相信大部分读者已经豁然开朗了。`ceph-volume`系统服务会触发`ceph-volume lvm trigger`， 再进入`activate`阶段，最后一系列的系统命令调用，完成osd启动。

整个启动流程大致如下图所示：
![image_1cn8kcnnbh521niu16djegnqhb4d.png-40.8kB][3]


## 创建分析


osd的创建跟启动代码入口是一样的,都是从`ceph/src/ceph-volume/main`的`main()`开始

![image_1cn8melrej3a5glk0c1cpqeqv4q.png-21.5kB][4]
根据传入的`mapper`和`argv`，再次回看下`terminal.py`里的`dispatch`
```python
## ceph/src/ceph-volume/ceph_volume/terminal.py
def dispatch(mapper, argv=None):
    argv = argv or sys.argv
    for count, arg in enumerate(argv, 1):
        if arg in mapper.keys():
            ## 创建实例
            instance = mapper.get(arg)(argv[count:])
            if hasattr(instance, 'main'):
                ## 实例main方法
                instance.main()
                raise SystemExit(0)
                
```
如下图所示的LVM的类变量`mapper`,结合上面内容，实例`instacne`实际上就是`create.Create()`,然后执行实例的`main()`方法
![image_1cn8i7f3l12b61ap1i0a1u1h124440.png-65.5kB][5]


```python
   ## ceph/src/ceph-volume/ceph_volume/devices/lvm/create.py
    def main(self):
        ... <中间省略>
        ## 解析参数
        if len(self.argv) == 0:
            print(sub_command_help)
            return
        exclude_group_options(parser, groups=['filestore', 'bluestore'], argv=self.argv)
        args = parser.parse_args(self.argv)
        ## 判断是bluestore还是filestore,默认为bluestore
        if not args.bluestore and not args.filestore:
            args.bluestore = True
        ## 调用create()方法
        self.create(args)
```
进入`create()`方法，重点关注它里包含了创建`Prepare`实例，然后进入`Activate().activate()`,这在前面启动阶段已经分析了。下面重点分析下`Prepare`里具体做了哪些处理。
```python
     ## ceph/src/ceph-volume/ceph_volume/devices/lvm/create.py
    @decorators.needs_root
    def create(self, args):
        if not args.osd_fsid:
            args.osd_fsid = system.generate_uuid()
        ### 注意看这里!! 包含了prepare ###
        prepare_step = Prepare([])
        prepare_step.safe_prepare(args)
        osd_id = prepare_step.osd_id
        try:
            Activate([]).activate(args)
        except Exception:
            logger.error('lvm activate was unable to complete, while creating the OSD')
            logger.info('will rollback OSD ID creation')
            ## 创建失败，就回滚，删除osd，其实就是调用ceph osd purge 相关命令 ##
            rollback_osd(args, osd_id)
            raise
        terminal.success("ceph-volume lvm create successful for: %s" % args.data)
```

```
    ##ceph/src/ceph-volume/ceph_volume/devices/lvm/prepare.py
    @decorators.needs_root
    def prepare(self, args):

        ...<中间省略>

        ### 创建osd的一些准备工作，比如keyring，osd id，osd uuid等
        cluster_fsid = conf.ceph.get('global', 'fsid')
        osd_fsid = args.osd_fsid or system.generate_uuid()
        crush_device_class = args.crush_device_class
        if crush_device_class:
            secrets['crush_device_class'] = crush_device_class
        self.osd_id = prepare_utils.create_id(osd_fsid, json.dumps(secrets), osd_id=args.osd_id)
        ## 这些元数据打包到tags里
        tags = {
            'ceph.osd_fsid': osd_fsid,
            'ceph.osd_id': self.osd_id,
            'ceph.cluster_fsid': cluster_fsid,
            'ceph.cluster_name': conf.cluster,
            'ceph.crush_device_class': crush_device_class,
        }
         ...<中间省略>
            #### 看重点！！！根据传入的数据先去扫描本地所有lvm卷信息，是否已经存在，如果没有，就把这些tags写入lvm的tags里
            data_lv = self.get_lv(args.data)
            if not data_lv:
                 ### prepare_device方法里会把tags写入到lvm卷的tags
                data_lv = self.prepare_device(args.data, 'data', cluster_fsid, osd_fsid)

             ...<中间省略>
    
            #### bluestore 初始化准备，跳转进入preapre_bluestore继续处理
            prepare_bluestore(
                block_lv.lv_path,
                wal_device,
                db_device,
                secrets,
                tags,
                self.osd_id,
                osd_fsid,
            )
```
  
```
### ceph/src/ceph-volume/ceph_volume/devices/lvm/prepare.py
def prepare_bluestore(block, wal, db, secrets, tags, osd_id, fsid):
    ...<中间省略>
    ### 其中都是一些准备工作，核心就是这个初始化了,也就是前面出面过的初始化
    prepare_utils.osd_mkfs_bluestore(
        osd_id, fsid,
        keyring=cephx_secret,
        wal=wal,
        db=db
    )
```

至此，`bluestore`已经初始化完毕了，然后接着走启动流程就可以了。
![image_1cn8n9740i5nilf19jgjl1l1264.png-45.7kB][6]

## 小结
通过阅读学习`ceph-volume`的代码，可以清楚地理解整个`osd`的创建与启动流程。后续本人还会继续发布学习`Ceph`的相关文章，读者的支持是我写作的最大动力。
##参考学习

https://cloud.tencent.com/developer/article/1171493
http://xcodest.me/ceph-bluestore-and-ceph-volume.html
http://xiaqunfeng.cc/2017/02/23/Bluestore%E8%B0%83%E7%A0%94/

  [1]: http://static.zybuluo.com/jaywayjayway/e1lsji8v2mbs1c8vopieungm/image_1cn618mlp1eskvqdeov12o1vgj19.png
  [2]: http://static.zybuluo.com/jaywayjayway/p9omgojlxqkm12i4xi221l76/image_1cn6k8a3e1f754eu15armjhcgj9.png
  [3]: http://static.zybuluo.com/jaywayjayway/nkbx7azy24yiz3r7o8rjwfnl/image_1cn8kcnnbh521niu16djegnqhb4d.png
  [4]: http://static.zybuluo.com/jaywayjayway/eny0sggwt0x3o88uavd6h55e/image_1cn8melrej3a5glk0c1cpqeqv4q.png
  [5]: http://static.zybuluo.com/jaywayjayway/ogpylq2bbc2de76785xmqjik/image_1cn8i7f3l12b61ap1i0a1u1h124440.png
  [6]: http://static.zybuluo.com/jaywayjayway/xbkjxghxnzry59grc35iutyz/image_1cn8n9740i5nilf19jgjl1l1264.png
  [7]: http://static.zybuluo.com/jaywayjayway/sgwnfng1s6688o1q0frxv7ur/image_1cn8mhj0bghiq6al82btg1d825n.png
