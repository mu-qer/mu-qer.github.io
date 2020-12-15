---
layout: post
title: object数据格式分析
date: 2020-09-25 23:30:09
categories: Ceph
description: 数据格式
tags: Ceph
---

# 1. rgw 对象
rgw对象提供两种上传接口：整体上传和分段上传
其中，整体上传对象最大不能超过 rgw_max_put_size (默认5G)

# 2. 整体上传

- rgw_max_chunk_size
> rgw向下发送的IO大小, 也是rados首对象的大小

- rgw_obj_stripe_size
> rados中间对象的大小, 简称条带大小

整体上传时, 上传的对象对应一个rados对象, 该rados对象以原对象名命名, 原对象的元数据保存在该rados对象的扩展属性中。
当上传的对象大于首对象大小时, 将会被分解, 分解成一个首对象, 多个大小等于条带大小的中间对象, 和一个小于等于条带的尾对象。

header对象 + [ 中间对象1 + ... + 中间对象n ] + tail对象
   4MB            4MB              4MB        <=4MB

首对象以对象名命名, 在RGW中首对象称为 headobj, 首对象(headobj)的 "数据部分" 保存 max_chunk_size 大小的数据, 首对象的扩展属性保存了 "原对象的元数据信息" 和 "manifest对象信息"。

中间对象和尾对象保存原对象剩余的数据信息，中间对象和尾对象的命名格式如下：
bucket_id +"_" + "_shadow_" + 32bit随机字符串 + "_" + 条带编号  (编号从 1开始)
例如：
8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1__shadow_.sPQ50HnGJ3iXld7Dpvor7lDKsdlpbjH_1


## 2.1 大于4MB文件的存储(整体上传)

1>. 使用s3cmd上传了一个5MB文件：test.5M.gz.clod.1009

```
[root@muhongtao-ceph-3 ~]# s3cmd ls s3://sensebucket
2020-10-09 09:44      5242880  s3://sensebucket/test.5M.gz.clod.1009
```

2>. 查看对应的 data pool, 发现包括两个对象数据：
marker == bucketid

```
[root@ceph-3 ~]# rados -p class_hdd_pool_1.data ls
8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1__shadow_.sPQ50HnGJ3iXld7Dpvor7lDKsdlpbjH_1    #{marker}_${object_name}
8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1_test.5M.gz.clod.1009
```

3>. 分别查看这两个对象的stat:
使用命令查询 stat:
```
rados -p ${pool_name} stat ${marker}_${object_name}
```

```
[root@ceph-4 ~]# rados -p class_hdd_pool_1.data stat 8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1_test.5M.gz.clod.1009
class_hdd_pool_1.data/8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1_test.5M.gz.clod.1009 mtime 2020-10-09 05:44:15.000000, size 4194304

[root@ceph-4 mht_workspace]# rados -p class_hdd_pool_1.data stat 8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1__shadow_.sPQ50HnGJ3iXld7Dpvor7lDKsdlpbjH_1
class_hdd_pool_1.data/8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1__shadow_.sPQ50HnGJ3iXld7Dpvor7lDKsdlpbjH_1 mtime 2020-10-09 05:44:15.000000, size 1048576
```

两个object大小分别为 4MB、1MB.

4>. 看下bucket下整个test.5M.gz.clod.1009对象的stat

```
radosgw-admin object stat --bucket=sensebucket --object=test.5M.gz.clod.1009 > 5m.info
```

```json
{
    "name": "test.5M.gz.clod.1009",
    "size": 5242880,
    "policy": {
        "acl": {
            .....
        },
        "owner": {
            "id": "1009-user-01",
            "display_name": "1009-user-01"
        }
    },
    "etag": "5f363e0e58a95f06cbe9bbc662c5dfb6",
    "tag": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95963.5",
    "manifest": {
        "objs": [],
        "obj_size": 5242880,
        "explicit_objs": "false",
        "head_size": 4194304,
        "max_head_size": 4194304,
        "prefix": ".sPQ50HnGJ3iXld7Dpvor7lDKsdlpbjH_",
        "rules": [
            {
                "key": 0,
                "val": {
                    "start_part_num": 0,
                    "start_ofs": 4194304,
                    "part_size": 0,
                    "stripe_max_size": 4194304,
                    "override_prefix": ""
                }
            }
        ],
        "tail_instance": "",
        "tail_placement": {
            "bucket": {
                "name": "sensebucket",
                "marker": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1",
                "bucket_id": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1",
                "tenant": "",
                "explicit_placement": {
                    "data_pool": "",
                    "data_extra_pool": "",
                    "index_pool": ""
                }
            },
            "placement_rule": "zone1-placement"
        },
        "begin_iter": {
            "part_ofs": 0,
            "stripe_ofs": 0,
            "ofs": 0,
            "stripe_size": 4194304,
            "cur_part_id": 0,
            "cur_stripe": 0,
            "cur_override_prefix": "",
            "location": {
                "placement_rule": "zone1-placement",
                "obj": {
                    "bucket": {
                        "name": "sensebucket",
                        "marker": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1",
                        "bucket_id": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1",
                        "tenant": "",
                        "explicit_placement": {
                            "data_pool": "",
                            "data_extra_pool": "",
                            "index_pool": ""
                        }
                    },
                    "key": {
                        "name": "test.5M.gz.clod.1009",			# ${marker}_${begin_iter.location.obj.key.name} 即为首分片名
                        "instance": "",
                        "ns": ""
                    }
                },
                "raw_obj": {
                    "pool": "",
                    "oid": "",
                    "loc": ""
                },
                "is_raw": false
            }
        },
        "end_iter": {
            "part_ofs": 4194304,
            "stripe_ofs": 4194304,
            "ofs": 5242880,
            "stripe_size": 1048576,
            "cur_part_id": 0,
            "cur_stripe": 1,
            "cur_override_prefix": "",
            "location": {
                "placement_rule": "zone1-placement",
                "obj": {
                    "bucket": {
                        "name": "sensebucket",
                        "marker": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1",
                        "bucket_id": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95969.1",
                        "tenant": "",
                        "explicit_placement": {
                            "data_pool": "",
                            "data_extra_pool": "",
                            "index_pool": ""
                        }
                    },
                    "key": {
                        "name": ".sPQ50HnGJ3iXld7Dpvor7lDKsdlpbjH_1",		# ${marker}_${end_iter.location.obj.key.name} 即为首分片名
                        "instance": "",
                        "ns": "shadow"
                    }
                },
                "raw_obj": {
                    "pool": "",
                    "oid": "",
                    "loc": ""
                },
                "is_raw": false
            }
        }
    },
    "attrs": {
        "user.rgw.content_type": "application/octet-stream",
        "user.rgw.pg_ver": "",
        "user.rgw.source_zone": "`<80>n$",
        "user.rgw.storage_class": "STANDARD",
        "user.rgw.tail_tag": "8fb2def7-7ccd-4803-a76a-566554e21b9e.95963.5",
        "user.rgw.x-amz-content-sha256": "c036cbb7553a909f8b8877d4461924307f27ecb66cff928eeeafd569c3887e29",
        "user.rgw.x-amz-date": "20201009T094415Z",
        "user.rgw.x-amz-meta-s3cmd-attrs": "atime:1602215145/ctime:1600336720/gid:0/gname:root/md5:5f363e0e58a95f06cbe9bbc662c5dfb6/mode:33188/mtime:1600336720/uid:0/uname:root"
    }
}            
```

# 3. 分段上传 

- rgw_multipart_min_part_size
> 针对分段上传, 客户端可以自定义上传的分段大小, 默认是5MB, 即：至少5MB

- rgw_max_put_size
> 当文件大于该值时将默认采用分段上传, 该值为5GB

## 3.1 分段上传数据格式
- 分段上传一个对象的时候, rgw按照条带大小将每个分段分成多个 rados对象.

- 每个分段的第一个rados对象的命名格式：
> bucketid + "_" + 上传的对象名 + "." + uploadid + "." + 分段编号 

- 每个分段的其余对象命名格式：
> bucketid + "_" + 上传的对象名 + "." + uploadid + "." + 分段编号 + "_" + 条带编号

