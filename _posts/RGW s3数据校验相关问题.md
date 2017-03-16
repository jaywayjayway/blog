title: RGW s3数据校验相关问题
date: 2017-03-14 12:44:10
tags:
 - rgw
categories: ceph
---
## 前言
随着云时代的来临，海量存储的需求出现了井喷。首当其次的是AWS的`s3`存储。基于ceph的rgw实现的s3是不少中小厂商的不二选择。今天鄙人就在这里简聊关于s3中数据校验及分片上传的相关问题。


## 上传验证
不少朋友首先就会想到用hash方法来校验数据的完整性。其中md5是最简单、通用的方式。在s3中的object会添加etag的元数据来表示此对象的md5(非分块上传模式)

生成一个1MB大小的文件，并验证它的md5值

```
[root@hz-ceph-01 mnt]# dd if=/dev/urandom  of=1M.txt bs=1M count=1
记录了1+0 的读入
记录了1+0 的写出
1048576字节(1.0 MB)已复制，0.103442 秒，10.1 MB/秒
[root@hz-ceph-01 mnt]# md5sum  1M.txt
a16adff9e15406d413ccc5751a49d735  1M.txt

```

使用`s3cmd `工具，把此文件上传到`test` bucket里
```
[root@hz-ceph-01 mnt]# s3cmd  put  1M.txt s3://test
upload: '1M.txt' -> 's3://test/1M.txt'  [1 of 1]
 1048576 of 1048576   100% in    0s     4.80 MB/s  done
```
通过`radosgw-admin`查看此object的元数据

```
[root@hz-ceph-01 mnt]# radosgw-admin  object stat --bucket=test --object=1M.txt
{
    "name": "1M.txt",
    "size": 1048576,
    "policy": {
        "acl": {
            "acl_user_map": [
                {
                    "user": "test",
                    "acl": 15
                }
            ],
            "acl_group_map": [],
            "grant_map": [
                {
                    "id": "test",
                    "grant": {
                        "type": {
                            "type": 0
                        },
                        "id": "test",
                        "email": "",
                        "permission": {
                            "flags": 15
                        },
                        "name": "just test",
                        "group": 0
                    }
                }
            ]
        },
        "owner": {
            "id": "test",
            "display_name": "just test"
        }
    },
    "etag": "a16adff9e15406d413ccc5751a49d735\u0000",  ##  在这里 ##
    "tag": "7473751f-f731-411d-ac91-0bc835035560.14099.138\u0000",
    "manifest": {
        "objs": [],
        "obj_size": 1048576,
        "explicit_objs": "false",
        "head_obj": {
            "bucket": {
                "name": "test",
                "pool": "default.rgw.buckets.data",
                "data_extra_pool": "default.rgw.buckets.non-ec",
                "index_pool": "default.rgw.buckets.index",
                "marker": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
                "bucket_id": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
                "tenant": ""
            },
            "key": "",
            "ns": "",
            "object": "1M.txt",
            "instance": "",
            "orig_obj": "1M.txt"
        },
        "head_size": 524288,
        "max_head_size": 524288,
        "prefix": ".Ysz22Hik0lKyuDDpvN-rJwUP9xsw8td_",
        "tail_bucket": {
            "name": "test",
            "pool": "default.rgw.buckets.data",
            "data_extra_pool": "default.rgw.buckets.non-ec",
            "index_pool": "default.rgw.buckets.index",
            "marker": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
            "bucket_id": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
            "tenant": ""
        },
        "rules": [
            {
                "key": 0,
                "val": {
                    "start_part_num": 0,
                    "start_ofs": 524288,
                    "part_size": 0,
                    "stripe_max_size": 4194304,
                    "override_prefix": ""
                }
            }
        ],
        "tail_instance": ""
    },
    "attrs": {
        "user.rgw.content_type": "application\/octet-stream\u0000",
        "user.rgw.pg_ver": "�\u0000\u0000\u0000\u0000\u0000\u0000\u0000",
        "user.rgw.source_zone": "\u0000\u0000\u0000\u0000",
        "user.rgw.x-amz-date": "Tue, 14 Mar 2017 15:07:48 +0000\u0000",
        "user.rgw.x-amz-meta-s3cmd-attrs": "uid:0\/gname:root\/uname:root\/gid:0\/mode:33188\/mtime:1489503916\/atime:1489503969\/md5:a16adff9e15406d413ccc5751a49d735\/ctime:1489503916\u0000",
        "user.rgw.x-amz-storage-class": "STANDARD\u0000"
    }
}
```
> - 可以通过jq工具做过滤,如下所示:
> - *radosgw-admin  object stat --bucket=test --object=1M.txt  | jq .etag*

两者对比就比较明显了，基本是一致的。
 
    "etag": "a16adff9e15406d413ccc5751a49d735\u0000"
    a16adff9e15406d413ccc5751a49d735  1M.txt
 
 > - 对于`etag`末尾的`\u0000`表示什么含义，目前鄙人还未知。了解的朋友希望能帮忙解释下
 
 
 接下来我们来聊一天分块上传中校验问题，关于s3 multipart upload 的详细原理及过程请参看 aws s3 文档 ，[请点我](http://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html)

鄙人简单的总结下上传的流程：

1. 本地使用` s3cmd put`  上传文件如果大于15MB的时候，默认会使用分片来上传。
2. 向s3服务端申请多片上传的`upload id` ,之后上传分片、列出分片、完成上传都需要用到这个 id 
3. 上传的时候还需要指向分片的编号（编号从1开始),本地把文件切出对应分片大小的数据块，然后通过 put 方法把分片数据块上传到s3 
4. 分片上传完成，s3会返回一个信息，并在http header里含有刚上传分片的元数据信息（即` etag` )
5. 本地通过比 etag 和 分片的md5值，判断分片数据是否完整上传成功
6. 重复 3、4、5 步骤直到所有分片都成功上传
7. 最后本地再发送一个完成上传的请求，请求里包含所有分片的编号和md5值，s3服务端就把这些分片组成一个完整的对象文件保存在指定的bucket里

###  注意
> - 3、4、5步骤可以通过多线程实现并发上传，但这需要自己开发

### 分片上传实例
创建个20MB 大小的 文件 `etag`
```
[root@hz-ceph-01 ~]# dd if=/dev/urandom  of=etag bs=1M count=20
20+0 records in
20+0 records out
20971520 bytes (21 MB) copied, 1.53815 s, 13.6 MB/s
```
查看etag的 `md5` 值 
```
[root@hz-ceph-01 ~]# md5sum etag
cbacae177df231e8b3714bf662a433d9  etag
```
上传文件`etag `
```
[root@hz-ceph-01 ~]# s3cmd  put etag  s3://test/
upload: 'etag' -> 's3://test/etag'  [part 1 of 2, 15MB] [1 of 1]  ## 每个分片默认大小为15MB ##
 15728640 of 15728640   100% in    2s     6.76 MB/s  done
upload: 'etag' -> 's3://test/etag'  [part 2 of 2, 5MB] [1 of 1]
 5242880 of 5242880   100% in    1s     3.74 MB/s  done
```
获取对象的元数据（只取 etag 部分)
```
[root@hz-ceph-01 ~]# radosgw-admin  object stat --bucket=test --object=etag | jq .etag
"c05bea71ce5d2b6eceaf46e9c347b22e-2\u0000"
```
疑问就出现了，为什么这个etag不再是上传文件的md5？那它又是什么呢？请下来鄙人就给看官们抽丝剥茧。

 **这个etag其中各分片数据块的md5值经过二进制转换再md5运算之后的结果**

下面我们通过手动方式来制造出这个'etag'值 


把etag文件按15MB大小做切片（最后一片按剩余大小切分，20MB=15MB+5MB）
```
[root@hz-ceph-01 ~]# dd if=etag  of=01 bs=15M count=1
1+0 records in
1+0 records out
15728640 bytes (16 MB) copied, 0.07331 s, 215 MB/s
[root@hz-ceph-01 ~]# dd if=etag  of=02 skip=15 bs=1M count=5
5+0 records in
5+0 records out
5242880 bytes (5.2 MB) copied, 0.00639034 s, 820 MB/s
```
查看分片的大小
```
[root@hz-ceph-01 ~]# du -sh *
15M	01
5.0M	02
4.0K	anaconda-ks.cfg
20M	etag
```
获取分片的md5值，重定向到一个文本文件
```
[root@hz-ceph-01 ~]# md5sum  01   | awk  '{print $1}' > checksums.txt
[root@hz-ceph-01 ~]# md5sum  02   | awk  '{print $1}' >> checksums.txt
```
查看文件内容
```
[root@hz-ceph-01 ~]# cat checksums.txt
38b979773d839d45670fd962d4e3eff3
3345b99b221433862a415e636098e98d
```

把文件转成二进制并进行md5运算
```
[root@hz-ceph-01 ~]# xxd  -r -p  checksums.txt  | md5sum
c05bea71ce5d2b6eceaf46e9c347b22e  -
```
以上结果跟我们先前上传的对象的etag值进行比对
**"c05bea71ce5d2b6eceaf46e9c347b22e-2\u0000"**

看到这结果，我想看官是不是豁然开郞了。etag值的`-2`表示2个分片。

下面重新上传etag文件，打开debug再观察下

```
[root@hz-ceph-01 ~]# s3cmd  put  etag  s3://test -d
DEBUG: s3cmd version 1.6.1
DEBUG: ConfigParser: Reading file '/root/.s3cfg'
DEBUG: ConfigParser: access_key->PY...17_chars...P
...<中间省略>

DEBUG: CHECK: etag  ## 本地校验etag文件 ##
DEBUG: PASS: u'etag'
INFO: Running stat() and reading/calculating MD5 values on 1 files, this may take some time...
DEBUG: DeUnicodising u'etag' using UTF-8
DEBUG: doing file I/O to read md5 of etag
DEBUG: DeUnicodising u'etag' using UTF-8
INFO: Summary: 1 local files to upload

## 添加元数据 ## 
DEBUG: attr_header: {'x-amz-meta-s3cmd-attrs': 'uid:0/gname:root/uname:root/gid:0/mode:33188/mtime:1489554799/atime:1489554908/md5:cbacae177df231e8b3714bf662a433d9/ctime:1489554799'} ## 添加元数据 ## 
DEBUG: DeUnicodising u'etag' using UTF-8
DEBUG: DeUnicodising u'etag' using UTF-8
DEBUG: DeUnicodising u'etag' using UTF-8
DEBUG: String 'etag' encoded to 'etag'
DEBUG: CreateRequest: resource[uri]=/etag?uploads
DEBUG: Using signature v2

 ## 签名header  ##
DEBUG: SignHeaders: 'POST\n\napplication/octet-stream\n\nx-amz-date:Wed, 15 Mar 2017 05:41:45 +0000\nx-amz-meta-s3cmd-attrs:uid:0/gname:root/uname:root/gid:0/mode:33188/mtime:1489554799/atime:1489554908/md5:cbacae177df231e8b3714bf662a433d9/ctime:1489554799\nx-amz-storage-class:STANDARD\n/test/etag?uploads'  ## 签名header  ##
DEBUG: Processing request, please wait...
DEBUG: get_hostname(test): test.s3.liudong.com

## 请求upload id  ## 
DEBUG: format_uri(): /etag?uploads
DEBUG: Sending request method_string='POST', uri='/etag?uploads', headers={'x-amz-meta-s3cmd-attrs': 'uid:0/gname:root/uname:root/gid:0/mode:33188/mtime:1489554799/atime:1489554908/md5:cbacae177df231e8b3714bf662a433d9/ctime:1489554799', 'content-type': 'application/octet-stream', 'Authorization': 'AWS PY59Q1KZG848ZU1RJIEP:nOc7HrZh1AqCqXOTS9q1D880FaI=', 'x-amz-date': 'Wed, 15 Mar 2017 05:41:45 +0000', 'x-amz-storage-class': 'STANDARD'}, body=(0 bytes)

## 返回结果，包含 uplaod id  ## 
DEBUG: Response: {'status': 200, 'headers': {'transfer-encoding': 'chunked', 'server': 'openresty/1.9.15.1', 'connection': 'keep-alive', 'x-amz-request-id': 'tx000000000000000000096-0058c8d419-3713-default', 'date': 'Wed, 15 Mar 2017 05:41:45 GMT', 'content-type': 'application/xml'}, 'reason': 'OK', 'data': '<?xml version="1.0" encoding="UTF-8"?><InitiateMultipartUploadResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Bucket>test</Bucket><Key>etag</Key><UploadId>2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup</UploadId></InitiateMultipartUploadResult>'}

DEBUG: get_hostname(test): test.s3.liudong.com
DEBUG: ConnMan.get(): re-using connection: http://test.s3.liudong.com#1
DEBUG: format_uri():

### 上传分片 1  ##  
/etag?partNumber=1&uploadId=2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup
    65536 of 15728640     0% in    0s   418.08 kB/sDEBUG: ConnMan.put(): connection put back to pool (http://test.s3.liudong.com#2)

## 上传结束，返回结果 ## 
DEBUG: Response: {'status': 200, 'headers': {'content-length': '0', 'accept-ranges': 'bytes', 'server': 'openresty/1.9.15.1', 'connection': 'keep-alive', 'etag': '"38b979773d839d45670fd962d4e3eff3"', 'x-amz-request-id': 'tx000000000000000000097-0058c8d419-3713-default', 'date': 'Wed, 15 Mar 2017 05:41:47 GMT'}, 'reason': 'OK', 'data': '', 'size': 15728640L}
 15728640 of 15728640   100% in    2s     6.09 MB/s  done
 
 ## 本地校验 md5 与 etag值  ## 
DEBUG: MD5 sums: computed=38b979773d839d45670fd962d4e3eff3, received="38b979773d839d45670fd962d4e3eff3"


...<忽略上传分片2>

DEBUG: MultiPart: Upload finished: 2 parts
DEBUG: MultiPart: Completing upload: 2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup
DEBUG: String 'etag' encoded to 'etag'
DEBUG: CreateRequest: resource[uri]=/etag?uploadId=2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup
DEBUG: Using signature v2
DEBUG: SignHeaders: 'POST\n\n\n\nx-amz-date:Wed, 15 Mar 2017 05:41:49 +0000\n/test/etag?uploadId=2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup'
DEBUG: Processing request, please wait...
DEBUG: get_hostname(test): test.s3.liudong.com
DEBUG: ConnMan.get(): re-using connection: http://test.s3.liudong.com#3
DEBUG: format_uri():
## 向服务端发送POST，完成上传 ## 
/etag?uploadId=2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup
DEBUG: Sending request method_string='POST', uri='/etag?uploadId=2~1SdbLkiHRWfXMXzcqiw5VaiqeH5HLup', headers={'content-length': '223', 'Authorization': 'AWS PY59Q1KZG848ZU1RJIEP:77yWJrG4+NZqSQM+Fx2ew8J2hFs=', 'x-amz-date': 'Wed, 15 Mar 2017 05:41:49 +0000'}, body=(223 bytes)

## 服务端返回上传对象的相关信息 ## 
DEBUG: Response: {'status': 200, 'headers': {'transfer-encoding': 'chunked', 'server': 'openresty/1.9.15.1', 'connection': 'keep-alive', 'x-amz-request-id': 'tx000000000000000000099-0058c8d41d-3713-default', 'date': 'Wed, 15 Mar 2017 05:41:49 GMT', 'content-type': 'application/xml'}, 'reason': 'OK', 'data': '<?xml version="1.0" encoding="UTF-8"?><CompleteMultipartUploadResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Location>test.s3.liudong.com</Location><Bucket>test</Bucket><Key>etag</Key><ETag>c05bea71ce5d2b6eceaf46e9c347b22e-2</ETag></CompleteMultipartUploadResult>'}
DEBUG: ConnMan.put(): connection put back to pool (http://test.s3.liudong.com#4)
```

从上面我们可以清晰地了解整个分片上传的过程。

##  SDK 实现分片上传
最常用的是 `boto`，为了更简单地演示，鄙人在此用REST的API来实现。由于需要签名认证过程比较繁琐，在此我借用下梯子。在github上面有个很不错的库[*python-requests-aws*](https://github.com/tax/python-requests-aws)，可以使用requests来实现s3的签名。[请点我](https://github.com/tax/python-requests-aws)

还是以上面提到的`etag`文件为例，`01`、`02`为分片.
我们一步一步拆解。

- 生成`upload id `
```
 # -*- coding: UTF-8 -*-
 # filename: uplaodid.py
import requests,sys
import xmltodict
from xmltodict import parse, unparse, OrderedDict
from awsauth import S3Auth
import time

host = 's3.liudong.com'
access_key = 'PY59Q1KZG848ZU1RJIEP'
secret_key = 'tlCP8M2d9b7rsOI7z6rE7TjJJyYPv36FGVXmTenB'
bucketname = 'test'
objectname = 'etag'

#申请uploadId
def getid():
    cmd = '/%s/%s?uploads' % (bucketname,objectname)
    url = 'http://%s%s' % (host,cmd)
    response = requests.post(url, auth=S3Auth(access_key, secret_key,service_url=host))
    UploadId = xmltodict.parse(response.content)['InitiateMultipartUploadResult']['UploadId']
    return UploadId
    
if __name__ == '__main__':
    print getid()
```
- 执行脚本获取`upload id`
```
[root@hz-ceph-01 ~]# python uploadid.py
2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW
```
- 通过s3cmd查看分片上传的`upload id`
```
[root@hz-ceph-01 ~]# s3cmd  multipart s3://test
s3://test/
Initiated	Path	Id
2017-03-16T02:32:59.303Z	s3://test/etag	2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW
```
- 上传分片 
```
 # -*- coding: UTF-8 -*-
 # filename: uploadpart.py
import requests,sys,os
import xmltodict
from xmltodict import parse, unparse, OrderedDict
from awsauth import S3Auth
import time

host = 's3.liudong.com'
access_key = 'PY59Q1KZG848ZU1RJIEP'
secret_key = 'tlCP8M2d9b7rsOI7z6rE7TjJJyYPv36FGVXmTenB'
bucketname = 'test'
objectname = 'etag'

#上传分片
def upload_part(UploadId,part,PartNumber):
    cmd = '/%s/%s?partNumber=%s&uploadId=%s' % (bucketname,objectname,PartNumber,UploadId)
    url = 'http://%s%s' % (host,cmd)
    with open(part, 'rb') as f:
        data = f.read()
    r = requests.put(url, auth=S3Auth(access_key, secret_key,service_url=host),data=data)
    print "status: %s" %(r.status_code )

if __name__ == '__main__':
    if len(sys.argv) == 4:
        uploadid = sys.argv[1]
        part = sys.argv[2]
        number= sys.argv[3]
    else:
         print 'uasage:%s [uploadid]  [partfile] [partnumber] ' % sys.argv[0]
         sys.exit(2)

    if os.path.exists(part):
        upload_part(uploadid,part,number)
    else:
        print "partifle is not exists"
        sys.exit(2)
```
- 执行脚本开始分片上传
```
[root@hz-ceph-01 ~]# python uploadpart.py  2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW 01 1
status: 200
[root@hz-ceph-01 ~]# python uploadpart.py  2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW 02 2
status: 200
```
- 在对应的pool里可以发现些踪迹
```
[root@hz-ceph-01 ~]# rados ls -p default.rgw.buckets.data
7473751f-f731-411d-ac91-0bc835035560.4201.1__multipart_etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW.2
7473751f-f731-411d-ac91-0bc835035560.4201.1__shadow_etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW.1_3
7473751f-f731-411d-ac91-0bc835035560.4201.1__shadow_etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW.1_1
7473751f-f731-411d-ac91-0bc835035560.4201.1__shadow_etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW.1_2
7473751f-f731-411d-ac91-0bc835035560.4201.1__shadow_etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW.2_1
7473751f-f731-411d-ac91-0bc835035560.4201.1__multipart_etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW.1
```
> - 注意：
> - 关于这个`shadow`跟`multpart`的关系还有待研究

- 最后发送完成上传的请求
```
 # -*- coding: UTF-8 -*-
 # filename: uploadend.py 
import requests,sys
import xmltodict
from xmltodict import parse, unparse, OrderedDict
from awsauth import S3Auth
import time

host = 's3.liudong.com'
access_key = 'PY59Q1KZG848ZU1RJIEP'
secret_key = 'tlCP8M2d9b7rsOI7z6rE7TjJJyYPv36FGVXmTenB'
bucketname = 'test'
objectname = 'etag'




def ending(UploadId):
    #获取当前所有分块列表
    cmd = '/%s/%s?uploadId=%s' % (bucketname,objectname,UploadId)
    url = 'http://%s%s' % (host,cmd)
    r = requests.get(url, auth=S3Auth(access_key, secret_key,service_url=host))

    print r.content
    partlist = []

    for i in xmltodict.parse(r.content)['ListPartsResult']['Part']:
        partlist.append({'PartNumber': i['PartNumber'], 'ETag': i['ETag']})

    obj = {'CompleteMultipartUpload': OrderedDict((
        ('Part',partlist),
    ))}

    #发送完成分块上传请求
    complete = xmltodict.unparse(obj,full_document=False)

    cmd = '/%s/%s?uploadId=%s' % (bucketname,objectname,UploadId)
    url = 'http://%s%s' % (host,cmd)
    r = requests.post(url, auth=S3Auth(access_key, secret_key,service_url=host),data=complete)
    print  "status: %s" % (r.status_code)


if __name__ == '__main__':
    if len(sys.argv) == 2:
        uploadid = sys.argv[1]
        ending(uploadid)
    else:
         print 'uasage:%s [uploadid]  ' % sys.argv[0]
         sys.exit(2)

```
- 执行完成脚本 `uploadend.py`
```
[root@hz-ceph-01 ~]# python uploadend.py  2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW
status: 200
```
- 查看s3里上传的`object`
```

[root@hz-ceph-01 ~]# radosgw-admin  object stat --bucket=test --object=etag
{
    "name": "etag",
    "size": 20971520,
    "policy": {
        "acl": {
            "acl_user_map": [
                {
                    "user": "test",
                    "acl": 15
                }
            ],
            "acl_group_map": [],
            "grant_map": [
                {
                    "id": "test",
                    "grant": {
                        "type": {
                            "type": 0
                        },
                        "id": "test",
                        "email": "",
                        "permission": {
                            "flags": 15
                        },
                        "name": "just test",
                        "group": 0
                    }
                }
            ]
        },
        "owner": {
            "id": "test",
            "display_name": "just test"
        }
    },
    "etag": "c05bea71ce5d2b6eceaf46e9c347b22e-2\u0000",
    "tag": "7473751f-f731-411d-ac91-0bc835035560.14099.335\u0000",
    "manifest": {
        "objs": [],
        "obj_size": 20971520,
        "explicit_objs": "false",
        "head_obj": {
            "bucket": {
                "name": "test",
                "pool": "default.rgw.buckets.data",
                "data_extra_pool": "default.rgw.buckets.non-ec",
                "index_pool": "default.rgw.buckets.index",
                "marker": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
                "bucket_id": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
                "tenant": ""
            },
            "key": "",
            "ns": "",
            "object": "etag",
            "instance": "",
            "orig_obj": "etag"
        },
        "head_size": 0,
        "max_head_size": 0,
        "prefix": "etag.2~AeLmBWjMBpYRDFqEHyT6BHaZ8-y9oxW",
        "tail_bucket": {
            "name": "test",
            "pool": "default.rgw.buckets.data",
            "data_extra_pool": "default.rgw.buckets.non-ec",
            "index_pool": "default.rgw.buckets.index",
            "marker": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
            "bucket_id": "7473751f-f731-411d-ac91-0bc835035560.4201.1",
            "tenant": ""
        },
        "rules": [
            {
                "key": 0,
                "val": {
                    "start_part_num": 1,
                    "start_ofs": 0,
                    "part_size": 15728640,
                    "stripe_max_size": 4194304,
                    "override_prefix": ""
                }
            },
            {
                "key": 15728640,
                "val": {
                    "start_part_num": 2,
                    "start_ofs": 15728640,
                    "part_size": 5242880,
                    "stripe_max_size": 4194304,
                    "override_prefix": ""
                }
            }
        ],
        "tail_instance": ""
    },
    "attrs": {
        "user.rgw.content_type": "\u0000",
        "user.rgw.pg_ver": "\u0019\u0000\u0000\u0000\u0000\u0000\u0000\u0000",
        "user.rgw.source_zone": "\u0000\u0000\u0000\u0000"
    }
}
```

对比先前通过`s3cmd`上传,可以在`attrs`部分发现点端倪。这部分属于用户自己定的元数据。

## 后记

以上是鄙人的在学习和应用s3过程中的一点积累。有疏忽之处请各位看官指正。欢迎各位一起学习Ceph,今后陆续发一些文章，敬请期待。。。
2017.3.15  执笔
