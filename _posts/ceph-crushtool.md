title: crushtool模拟测试CRUSH分布情况
categories: ceph
tags: [ceph,crush]
date: 2016-05-20 20:10:04
---
####  前言
Ceph通过crush实现数据的伪随机分布。在ceph里，一但是你的crush创建成功(无变更)，你创建的所有object对应于OSD的映射关系是已经确认了。这就是本人所理解的伪随机分布，先决条件已经确认，可以推算数据的具体分落

-------
#### 创建crush map
在ceph的工具链中有一款强大的工具---`crushtool`。可以用来创建、编辑、测试.

```python 
# crushtool --outfn  crushmap --build --num_osds 10 \
   host straw 2 rack straw 2 default straw 0
2015-10-09 15:00:15.194369 7f20e30fc780  1 
ID  WEIGHT  TYPE NAME
-9  10.00000    default default
-6  4.00000     rack rack0
-1  2.00000         host host0
0   1.00000             osd.0
1   1.00000             osd.1
-2  2.00000         host host1
2   1.00000             osd.2
3   1.00000             osd.3
-7  4.00000     rack rack1
-3  2.00000         host host2
4   1.00000             osd.4
5   1.00000             osd.5
-4  2.00000         host host3
6   1.00000             osd.6
7   1.00000             osd.7
-8  2.00000     rack rack2
-5  2.00000         host host4
8   1.00000             osd.8
9   1.00000             osd.9
 
```
其中 `--outfn crushmap` 表示导出的mapy的文件名是crushmap ，`--build` 表示创建一个crushmap , `--num_osds` 表示此map 包含10 个 osd ,  `host straw 2` 每个host 里包含两个 osd, `rack straw 2`   每个rack里 包含两个host, `default straw 0` 表示所有的rack 都包含在一个root 里。

> 注：crush 的层级可以自定义添加，比如在rack 层级上可以添加 `dc`  级，整个结构为  ` --num_osds 10    host straw 2 rack straw 2   dc straw 1 default straw 0 `,生成的crush map 如下所示
``` shell 
ID  WEIGHT  TYPE NAME
-12 10.00000    default default
-9  4.00000     dc dc0
-6  4.00000         rack rack0
-1  2.00000             host host0
0   1.00000                 osd.0
1   1.00000                 osd.1
-2  2.00000             host host1
2   1.00000                 osd.2
3   1.00000                 osd.3
-10 4.00000     dc dc1
-7  4.00000         rack rack1
-3  2.00000             host host2
4   1.00000                 osd.4
5   1.00000                 osd.5
-4  2.00000             host host3
6   1.00000                 osd.6
7   1.00000                 osd.7
-11 2.00000     dc dc2
-8  2.00000         rack rack2
-5  2.00000             host host4
8   1.00000                 osd.8
9   1.00000                 osd.9
``` 

依据自己的需要可以创建出五花八门的crush map。

----------
#### 编辑 crush map 
创建之后会在本地目录生成一个crushmap的二进制文件，我们可以通过crushtool工具进行反编译 
``` shell 
#crushtool  -d  crushmap  -o map.txt 
[root@localhost ceph ]# ll
total 8
-rw-r--r-- 1 root root  833 Oct  9 15:13 crushmap
-rw-r--r-- 1 root root 2351 Oct  9 15:20 map.txt  ##可编辑文件

```
打开本地的`map.txt`我们会发现 `crush type `部分只有我们创建的那几个层级(如下所示)，不再是默认的crushmap的有10 个层级。
``` shell 
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable straw_calc_version 1

# devices
device 0 device0
device 1 device1
device 2 device2
device 3 device3
device 4 device4
device 5 device5
device 6 device6
device 7 device7
device 8 device8
device 9 device9

# types
type 0 device
type 1 host
type 2 rack
type 3 dc
type 4 default
# buckets

.....
```
我们为`map.txt`添加我们自己的`crush rule `
``` shell
rule  custom {
        ruleset 1
        type replicated
        min_size 1
        max_size 10
        step take default
        step choose firstn 1 type dc
        step chooseleaf  firstn 0 type host
        step emit
}
```
`rule  custom`简要说明：
> 1.在root 层级里选择一个 `dc`级层作为后续隔离域的基础
> 2.在选中的 `dc`级层里，以**host** 为隔离域，选择**osd**    

编辑后我们需要的crushmap之后，再次通过`crushtool` 进行编译
```shell 
[root@localhost]# crushtool  -c map.txt  -o map.bin 
[root@localhost]# ll
total 12
-rw-r--r-- 1 root root  833 Oct  9 15:13 crushmap
-rw-r--r-- 1 root root 1071 Oct  9 15:36 map.bin
-rw-r--r-- 1 root root 2575 Oct  9 15:31 map.txt

```
----
#### 测试 crush map
``` shell 
[root@loclahost ]# crushtool -i map.bin  --test --show-statistics --rule 1 --min-x 1 --max-x 5 --num-rep 2  --show-mappings 

rule 1 (custom), x = 1..5, numrep = 2..2
CRUSH rule 1 x 1 [9]  #映射关系,即Object  1 映射到了 OSD 9
CRUSH rule 1 x 2 [7,4] #Object 2 映射到了 OSD 7、4
CRUSH rule 1 x 3 [7,4]
CRUSH rule 1 x 4 [2,0]
CRUSH rule 1 x 5 [7,4]
rule 1 (custom) num_rep 2 result size == 1: 1/5
rule 1 (custom) num_rep 2 result size == 2: 4/5

```
`--test ` 表示调用`crushtool`里的测试功能。`--show-statistics` 表示显示统计结果。 `--rule 1` 表示使用**rule 1** ,即我们自己创建的`rule custom ` . ` --min-x 1 --max-x 5` 表示创建的object的数量（如果不指定，表示创建 1024 ) .` --num-rep 2  `表示创建的副本数为**2** 。` --show-mappings  ` 表示显示具体的映射关系。

``` 
rule 1 (custom) num_rep 2 result size == 1: 1/5
rule 1 (custom) num_rep 2 result size == 2: 4/5
```
>  1. 前者表示1/5的object 成功映射到了一个osd上 ##表示映射有不成功
>  2. 后者表示4/5的object 成功映射到了两个osd上
> 
> **解析：**
> 
>   从前文的cursh map里可知，当crush 选择 `dc`层级的**`dc2`**时，由于在此层级下只有一个host，且隔离域刚好为host，所以无法再映射第二副本
> 
![Alt text](http://7xj51m.com1.z0.glb.clouddn.com/QQ截图20151009155310.png)



此外，还可以添加`--tree ` 显示crushmap的树状结构，` --show-choose-tries ` 显示映射时`retry`的次数。这个值可以在`tunables` 部分里找到。可以通过`--set-choose-total-tries` 强制指定。
``` shell
[root@localhost]# crushtool -i map.bin  --test --show-statistics --rule 1 --min-x 1 --max-x 5 --num-rep 2  --show-mappings --set-choose-total-tries  1 -o newmap   ##强制指只制尝试一次
rule 1 (custom), x = 1..5, numrep = 2..2
CRUSH rule 1 x 1 [9]  ##映射失败
CRUSH rule 1 x 2 [7]  ##映射失败
CRUSH rule 1 x 3 [7,4]
CRUSH rule 1 x 4 [2,0]
CRUSH rule 1 x 5 [7,4]
rule 1 (custom) num_rep 2 result size == 1: 2/5
rule 1 (custom) num_rep 2 result size == 2: 3/5
``` 
>  显然这个值会影响CRUSH的分布效果，默认值为50 

`--show-utilization` 可以显示OSD的实际的object映射数以及目标映射值
``` 
  device 0:      stored : 1  expected : 0.5
  device 2:      stored : 1  expected : 0.5
  device 4:      stored : 2  expected : 0.5
  device 7:      stored : 3  expected : 0.5
  device 9:      stored : 1  expected : 0.5

``` 

----------

**更多的设置可以通过    `man  crushtool`  获取**
