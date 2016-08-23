title: 高效简单的分布式存储系统--weedfs
categories: 分布式存储
tags: [weedfs]
date: 2015-09-28 21:53:09
---
#### 目录
- 介绍
- 架构
- 部署
- 运维
- 测试
- 后记

-----
#### 一、 介绍
`weed-fs`，全名**Seaweed-fs**，是一种用golang实现的简单且高可用的分布式文件系统。该系统有以下两个目标：

- to store billions of files
- to serve the files fast

简单地讲，`weedfs`只是一个key/values存储，并不完全支持POSIX文件系统。它类似于“NOSQL"，可以简单把它看成是"NOFS".存储接口通过http 方式对向外提供服务。

<!--more-->
-----


#### 二、架构

整个weedfs包括三种服务：

>1. master
>2. volume
>3. filer

**master:** 维护整个集群状态的服务。master支持集群工作模式，通过多master选举work leader ,实现高可用服务，避免单点故障。

**volume:**提供数据存储服务。每个volume服务可以创建多个volume级别的存储实例

**filer:** 提供类似普通文件系统的目录结构(可选)

类比于常见的分布存储系统--ceph，我们可以清晰地找到以上服务的”兄弟“。
master即相当于ceph中的Monitor 组件，volumes就是OSD的克隆，filer与MDS异曲同工。

分布式存储系统最重要的特点就是保证数据可靠性。常见的可靠性方案有副本机制，有纠删码机制。当前，weedfs通过副本来保证数据的可靠性。相较于ceph完善的crush模型，weedfs也提供一个类似的集群拓扑结构。它(Topology)**DataCenter**(数据中心)**Rack**(机架)、**Machine**(或叫Node)组成。weed-fs可以通过配置文件来描述整个集群的拓扑结构，也可以在运行volumes服务的时候通过参数设置拓扑。配置文件以XML格式为承载。以下是官方的一个示例 ：

``` shell
<Configuration>
  <Topology>
    <DataCenter name="dc1">
      <Rack name="rack1">
        <Ip>192.168.1.1</Ip>
      </Rack>
    </DataCenter>
    <DataCenter name="dc2">
      <Rack name="rack1">
        <Ip>192.168.1.2</Ip>
      </Rack>
      <Rack name="rack2">
        <Ip>192.168.1.3</Ip>
        <Ip>192.168.1.4</Ip>
      </Rack>
    </DataCenter>
  </Topology>
</Configuration>

```

**Replication机制**
weed-fs提供的是volume级别的副本机制。在启动master时候的可以指定默认的`Replication`策略。

``` shell
[root@localhost]$ weed master -defaultReplication=001
```
此处的`001`表示在同rack级设置一个副本（表示在一个rack里，有两份数据镜像，类似于RAID1）
下面对`001`所代表的replication机制进行简要解析

| Value      |    Value |
| :--------  | :--------|
| 000        | 无副本，只有一份 |
| 001        | 在同一rack级有一个副本 |
| 010        | 在同一dataCenter不同rack的里有一个副本 |
| 100        | 在不同的dataCenter里有一个副本
| 200        | 在不同的dataCenter里有两个副本
| 110        | 在不同的dataCenter和同级的rack各有一个副本
| ....       | ....

如果replication的模式为`xyz`，**xyz**表示的含义如下

| Value      |    Value |
| :--------  | :--------|
| x          |  不同dataCenter级的副本数|
| y          | 同一dataCenter，不同rack级的副本数|
| z          | 同一rack,不同node级的副本数 |


-------

#### 三、部署
由于weedfs把所有服务都集成在一个二制文里，相对其他的分布式存储系统，weedfs的部署就简单的多了。
> 1. 二进制下载地址，[点击进入](https://bintray.com/chrislusf/seaweedfs/seaweedfs)
> 2. github源代码，[点击进入](https://github.com/chrislusf/seaweedfs)
> 3. docker 镜像，`docker search  seaweedfs`

以下是部署weedfs的脚本，内容包括：
> 1. 三个master
> 2. 三个volume
> 3. 每个volume分布在不同的rack
> 4. defaultReplication=010
> 5. 一个filer

``` shell
#!/bin/bash

MASTER_v1="/data/master/v1"
MASTER_v2="/data/master/v2"
MASTER_v3="/data/master/v3"
VOLUME_v1="/data/volume/v1"
VOLUME_v2="/data/volume/v2"
VOLUME_v3="/data/volume/v3"
FILER="/data/filer"
BASE_PORT=9000
PATH=/usr/local/sbin:/usr/local/bin:/usr/bin:/sbin:/usr/sbin:/bin
export LANG=cn_US.UTF-8


##以下是私有yum源检测
rpm -qa | grep opstack-source > /dev/null 2>&1  || rpm -hvi http://xx.xx.xx.xx:/opstack-source-1.0-noarch.rpm
rpm -qa | grep weed-fs >  /dev/null 2>&1 || yum install weed-fs  -y
which screen > /dev/null 2>&1 || yum install screen -y
###
mkdir -p {$MASTER_v1,$MASTER_v2,$MASTER_v3,$VOLUME_v1,$VOLUME_v2,$VOLUME_v3,$FILER}

##
while  true
do
lsof  -i:${BASE_PORT} > /dev/null 2>&1

if  [ "X$?" == "X0" ];then

    BASE_PORT=`expr $BASE_PORT + 1 `
    continue
fi

   break

done


MASTER_v1_PORT=${BASE_PORT}
MASTER_v2_PORT=`expr $MASTER_v1_PORT + 1 `
MASTER_v3_PORT=`expr $MASTER_v2_PORT + 1 `
BASE_PORT=$MASTER_v3_PORT
screen -d -m -S weedfs-master-v1  su - root -c " weed -v=3 master  -port=${MASTER_v1_PORT}  -mdir=${MASTER_v1}  -peers=localhost:${MASTER_v1_PORT},localhost:${MASTER_v2_PORT},localhost:${MASTER_v3_PORT} -defaultReplication=010 > ${MASTER_v1}/master.log 2>&1"
screen -d -m -S weedfs-master-v2  su - root -c " weed -v=3 master  -port=${MASTER_v2_PORT}  -mdir=${MASTER_v2}  -peers=localhost:${MASTER_v1_PORT},localhost:${MASTER_v2_PORT},localhost:${MASTER_v3_PORT} -defaultReplication=010 >  ${MASTER_v2}/master.log 2>&1"
screen -d -m -S weedfs-master-v3  su - root -c " weed -v=3 master  -port=${MASTER_v3_PORT}  -mdir=${MASTER_v3}  -peers=localhost:${MASTER_v1_PORT},localhost:${MASTER_v2_PORT},localhost:${MASTER_v3_PORT} -defaultReplication=010 >  ${MASTER_v3}/master.log 2>&1"


BASE_PORT=`expr $BASE_PORT + 1 `
VOLUME_v1_PORT=${BASE_PORT}
VOLUME_v2_PORT=`expr $VOLUME_v1_PORT + 1 `
VOLUME_v3_PORT=`expr $VOLUME_v2_PORT + 1 `

BASE_PORT=$VOLUME_v3_PORT
screen -d -m -S weedfs-volume-v1  su - root -c "weed -v=3 volume -port=${VOLUME_v1_PORT} -dir=${VOLUME_v1} -mserver=localhost:${MASTER_v1_PORT}  -mserver=localhost:${MASTER_v2_PORT}  -mserver=localhost:${MASTER_v3_PORT}  -rack=vol-v1 -max=100 >  ${VOLUME_v1}/vol.log 2>&1"
screen -d -m -S weedfs-volume-v2  su - root -c "weed -v=3 volume -port=${VOLUME_v2_PORT} -dir=${VOLUME_v2} -mserver=localhost:${MASTER_v1_PORT}  -mserver=localhost:${MASTER_v2_PORT}  -mserver=localhost:${MASTER_v3_PORT}  -rack=vol-v2 -max=100 >  ${VOLUME_v2}/vol.log 2>&1"
screen -d -m -S weedfs-volume-v3  su - root -c "weed -v=3 volume -port=${VOLUME_v3_PORT} -dir=${VOLUME_v3} -mserver=localhost:${MASTER_v1_PORT}  -mserver=localhost:${MASTER_v2_PORT}  -mserver=localhost:${MASTER_v3_PORT}  -rack=vol-v3 -max=100 >  ${VOLUME_v3}/vol.log 2>&1"

BASE_PORT=`expr $BASE_PORT + 1 `
FILER_PORT=$BASE_PORT
screen -d -m -S weedfs-filer  su - root -c "weed  -v=3  filer  -port=${FILER_PORT} -dir=${FILER} -master=localhost:${MASTER_v1_PORT}  > ${FILER}/filer.log 2>&1"


cat << EOF
 ----------------------------------
weedfs-maser-v1: localhost:${MASTER_v1_PORT}, log is ${MASTER_v1}/master.log
weedfs-maser-v2: localhost:${MASTER_v2_PORT}, log is ${MASTER_v2}/master.log
weedfs-maser-v3: localhost:${MASTER_v3_PORT}, log is ${MASTER_v3}/master.log
weedfs-vol-v1  : localhost:${VOLUME_v1_PORT}, log is ${MASTER_v1}/vol.log
weedfs-vol-v2  : localhost:${VOLUME_v2_PORT}, log is ${MASTER_v1}/vol.log
weedfs-vol-v3  : localhost:${VOLUME_v3_PORT}, log is ${MASTER_v1}/vol.log
weedfs-filer   : localhost:${FILER_PORT}, log is ${FILER}/filer.log
EOF


```
------

#### 四、运维

**1.基本操作：存储、获取和删除文件**

所有的 HTTP API 都可以通过添加 `&pretty=y` 参数来格式化 json 输出.首先我们在本地创建 一个测试文件。

``` shell

$ echo "hello weed-fs" > test.txt

```
**存储**一个文件，把`test.txt`上传到集群里
``` shell
$ curl -F file=@test.txt   http://localhost:9000/submit
{"fid":"2,0225cc5c5b","fileName":"test.txt","fileUrl":"127.0.0.1:9004/2,0225cc5c5b","size":38}
```
从返回的结果我们可以获取以下信息：
    > **2,01b126cf55**  这个字符串应该由volume id, key uint64和cookie code构成。其中逗号前面的2就是volume id, `01b126cf55`则是key和cookie组成的串。fid是文件`text.txt`在集群中的唯一ID。后续查看、获取以及删除该文件数据都需要使 用这个fid。
    >**fileUrl** 是该文件在weed-fs中的一个访问地址(多副本策略下会有多个)

有了fileUrl我们就可以**获取**文件了
``` shell
$ curl http://127.0.0.1:9004/2,0225cc5c5b
hello weed-fs  ##即我们本地文件内容，存储有效##

```
如果我们通过master地址的接口来查看上传的文件会发生什么呢？

``` shell
$ curl http://127.0.0.1:9000/2,0225cc5c5b
<a href="http://127.0.0.1:9005/2,0225cc5c5b">Moved Permanently</a>.

$ curl http://127.0.0.1:9000/2,0225cc5c5b
<a href="http://127.0.0.1:9005/2,0225cc5c5b">Moved Permanently</a>.

$ curl http://127.0.0.1:9000/2,0225cc5c5b
<a href="http://127.0.0.1:9004/2,0225cc5c5b">Moved Permanently</a>.
```
如上所示，master会提示我们文件真正的存储地址，而且在多副本的情况会轮训给不同的url.

`submit`是最简单上传接口，一般情况下我们可以向集群提交fid请求，然后根据fid再上传文件。

``` shell
$ curl http://localhost:9000/dir/assign
{"fid":"4,0384460ad9","url":"127.0.0.1:9003","publicUrl":"127.0.0.1:9003","count":1}

# 分配文件 id 并且指定副本类型
$ curl http://localhost:9000/dir/assign?replication=100
{"error":"Cannot grow volume group! Not enough data node found!"}#因为我们的环境只有一个datacenter所以请求失败


# 指定需要分配多少个文件 id
$ curl "http://localhost:9000/dir/assign?count=5"
{"fid":"6,06c06d6065","url":"127.0.0.1:9004","publicUrl":"127.0.0.1:9004","count":5}

# 指定拓扑分配（示例为rack)
$ curl "http://localhost:9000/dir/assign?rack=vol-v3"
{"fid":"4,056f1d9555","url":"127.0.0.1:9003","publicUrl":"127.0.0.1:9003","count":1}
```
通过HTTP 的DELETE方法 即可删除存储端的文件
``` shell
$ curl  -X DELETE  "http://127.0.0.1:9005/2,0225cc5c5b"
{"size":38}

$ curl  -I     http://127.0.0.1:9005/2,0225cc5c5b
HTTP/1.1 404 Not Found
Date: Mon, 21 Sep 2015 09:27:41 GMT
Content-Type: text/plain; charset=utf-8

$ curl  -I     http://127.0.0.1:9004/2,0225cc5c5b
HTTP/1.1 404 Not Found
Date: Mon, 21 Sep 2015 09:28:02 GMT
Content-Type: text/plain; charset=utf-8

```
如上所示，无论是数据本身还是副本已经都被删除了。

默认情况下删除的文件空间未不会被释放，通过`vacuum `接口进行强制回收
``` shell
$ curl "http://localhost:9000/vol/vacuum"
```

**2.状态查询**

master集群状态
``` shell
$ curl "http://localhost:9000/cluster/status?pretty=y"
{
  "IsLeader": true,
  "Leader": "localhost:9000",  ## work leader
  "Peers": [
    "localhost:9001",    #集群成员
    "localhost:9002"     #集群成员
  ]
}
```

拓扑信息查看
```shell
$ curl  "http://localhost:9000/dir/status?pretty=y"
{
  "Topology": {
    "DataCenters": [
      {
        "Free": 288,   #未分配的vollume卷数量
        "Id": "DefaultDataCenter",
        "Max": 300,   #总共可用volume卷数量
        "Racks": [
          {
            "DataNodes": [
              {
                "Free": 97,
                "Max": 100,
                "PublicUrl": "127.0.0.1:9003",
                "Url": "127.0.0.1:9003",
                "Volumes": 3  #当前节点分配的volume
              }
            ],
            "Free": 97,
            "Id": "vol-v1",  #rack id
            "Max": 100
          },
          {
            "DataNodes": [
              {
                "Free": 96,
                "Max": 100,
                "PublicUrl": "127.0.0.1:9004",
                "Url": "127.0.0.1:9004",
                "Volumes": 4
              }
            ],
            "Free": 96,
            "Id": "vol-v2",
            "Max": 100
          },
          {
            "DataNodes": [
              {
                "Free": 95,
                "Max": 100,
                "PublicUrl": "127.0.0.1:9005",
                "Url": "127.0.0.1:9005",
                "Volumes": 5
              }
            ],
            "Free": 95,
            "Id": "vol-v3",
            "Max": 100
          }
        ]
      }
    ],
    "Free": 288,
    "Max": 300,
    "layouts": [
      {
        "collection": "",
        "replication": "010",
        "ttl": "",
        "writables": [
          4,
          1,
          2,
          3,
          5,
          6
        ]
      },
      {
        "collection": "",
        "replication": "100",
        "ttl": "",
        "writables": null
      },
      {
        "collection": "",
        "replication": "001",
        "ttl": "",
        "writables": null
      }
    ]
  },
  "Version": "0.70 beta"
```

volume卷的状态

``` shell
$ curl "http://localhost:9000/vol/status?pretty=y"
{
  "Version": "0.70 beta",
  "Volumes": {
    "DataCenters": {
      "DefaultDataCenter": {
        "vol-v1": {
          "127.0.0.1:9003": [
            {
              "Id": 4,  ## volume id
              "Size": 8,
              "ReplicaPlacement": {  ##replication策略
                "SameRackCount": 0,
                "DiffRackCount": 1,
                "DiffDataCenterCount": 0
              },
              "Ttl": {},
              "Collection": "",
              "Version": 2,
              "FileCount": 0,
              "DeleteCount": 0,
              "DeletedByteCount": 0,
              "ReadOnly": false
            },
            {
              "Id": 5,
              "Size": 8,
              "ReplicaPlacement": {
                "SameRackCount": 0,
                "DiffRackCount": 1,
                "DiffDataCenterCount": 0
              },
              "Ttl": {},
              "Collection": "",
              "Version": 2,
              "FileCount": 0,
              "DeleteCount": 0,
              "DeletedByteCount": 0,
              "ReadOnly": false
            },
            {
              "Id": 3,
              "Size": 96,
              "ReplicaPlacement": {
                "SameRackCount": 0,
                "DiffRackCount": 1,
                "DiffDataCenterCount": 0
              },
              "Ttl": {},
              "Collection": "",
              "Version": 2,
              "FileCount": 1,
              "DeleteCount": 0,
              "DeletedByteCount": 0,
              "ReadOnly": false
            }
          ]
        },
...........
```

状态不佳，先就此挂笔了。记于2015.09.28
