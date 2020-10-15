---
layout: post
title: object数据格式分析
date: 2020-09-25 23:30:09
categories: Ceph
description: 数据格式
tags: Ceph
---


# 1. 大于4MB文件的存储

1>. 使用s3cmd上传了一个5MB文件：test.5M.gz.clod.1009

```
[root@muhongtao-ceph-3 ~]# s3cmd ls s3://sensebucket
2020-10-09 09:44      5242880  s3://sensebucket/test.5M.gz.clod.1009
```

2>. 查看对应的 data pool, 发现包括两个对象数据：

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

```
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


