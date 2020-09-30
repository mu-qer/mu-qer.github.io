---
layout: post
title: rgw multisite 扩展已有集群
date: 2020-09-25 23:30:09
categories: Ceph
description: ceph-rgw多活
tags: Ceph
---


# 1. 说明
本篇文章介绍两个已有集群融合后的数据同步
> 上篇文章 rgw multisite写了关于两个集群之间zone的数据同步部署和测试。其中zone1,zone2之间满足条件：
> - 1. zone1: 新集群 && zone2: 新集群
> - 2. zone1: 已有集群 && zone2: 新集群

# 2. 现有的两个集群环境说明
## zone1


## zone2
 
# 4.multisite 集群搭建
multisite集群模型如下:

![rgw-multisite-cluster](https://mu-qer.github.io/assets/img/ceph/2020-09-15-rgw-multisite-cluster-01.JPG)

## 4.1 创建master-zone/zg/realm
在master zone集群中创建 realm, zonegroup, master zone

- [1] 创建realm, 设为 default

```
radosgw-admin realm create --rgw-realm=petrel --default
```
- [2] 创建master zonegroup

```
radosgw-admin zonegroup create --rgw-zonegroup=petreloss --endpoints=http://192.168.2.27:80 --rgw-realm=petrel --master --default
```
- [3] 创建master zone

```
radosgw-admin zone create --rgw-zonegroup=petreloss --rgw-zone=zone1 --master --default --endpoints=https://192.168.2.27:80
```
- [4] 删除默认产生的pool,zone,zonegroup等

```
radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
radosgw-admin zone delete --rgw-zone=default
radosgw-admin zonegroup delete --rgw-zonegroup=default
radosgw-admin period update --commit
```
- [5] 删除默认的pool

```
pools=`rados lspools | grep default`; for pool in ${pools[@]}; do rados rmpool $pool $pool --yes-i-really-really-mean-it; done
```
> 在删除这些default pool的时候需要自己检测下自己的集群是否需要这些pool，不需要即可删掉

- [6] 创建同步sync的系统级用户

```
radosgw-admin user create --uid="sync-admin" --display-name="sync-admin" --system
```

- [7] 更新zone的key信息
access-key 和 secret 填写的是上一步创建的系统级用户的access-key和secret-key
```
radosgw-admin zone modify --rgw-zone=zone1 --access-key=xxx --secret=xxxx 
```

- [8] 更新period

```
radosgw-admin period update --commit
```

- [9] 更新ceph.conf, 找到对应的rgw配置区，并重启rgw实例

```
[client.rgw.dev-1]
rgw zone = zone1

systemctl list-units|grep rgw
systemctl restart ceph-rgw@.xxxxx.service
```

## 4.2 创建secondary zone
这里有两个集群，master zone, slave zone, 但元数据的操作必须在master zone中执行，如：创建用户或者bucket, 在secondary zone中执行会被重定向到master zone中。

- [1] 从master zone拉取realm配置信息

```
radosgw-admin realm pull --url=http://192.168.2.27:80 --access-key=xxxxxxxx --secret=xxxxxxxx --rgw-realm=petrel
```
> 使用的access-key和secret就是同步的系统用户的信息

- [2] 从master zone拉取period配置信息

```
radosgw-admin period pull --url=http://192.168.2.27:80 --access-key=xxxxxx --secret=xxxxxxxxx --rgw-realm=petrel
```
- [3] 创建secondary zone
```
radosgw-admin zone create --rgw-zonegroup=petreloss --rgw-zone=zone2  --access-key=xxxxxxxx --secret=xxxxxxxxxxx --endpoints=http://192.168.2.166:80
```

- [4] 删掉 secondary zone上的default zone 还有pool

```
radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
radosgw-admin zone delete --rgw-zone=default
radosgw-admin zonegroup delete --rgw-zonegroup=default
radosgw-admin period update --commit
```

```
pools=`rados lspools | grep default`; for pool in ${pools[@]}; do rados rmpool $pool $pool --yes-i-really-really-mean-it; done
```
- [5] 更新period
```
radosgw-admin period update --commit
```
- [6] 更新rgw配置
```
[client.rgw.dev-4]
rgw zone = zone2
```

# 5 check 同步操作
## 5.1 查看 sync 的状态
- master zone同步状态

```
[root@ceph-3 ~]# radosgw-admin sync status
          realm 21f8c114-44da-4ff2-9a39-b8503b1f03ba (petrel)
      zonegroup 7c622bcb-f89c-4e92-aec8-503f4b03b57b (petreloss)
           zone 8b322746-4cb8-44e9-8498-a99d56d41bef (zone1)
  metadata sync no sync (zone is master)
      data sync source: 33053987-0427-4808-bfc9-ba742eded198 (zone2)
                        syncing
                        full sync: 0/128 shards
                        incremental sync: 128/128 shards
                        data is caught up with source
```
- secondary zone同步状态

```
[root@ceph-6 ~]# radosgw-admin sync status
          realm 21f8c114-44da-4ff2-9a39-b8503b1f03ba (petrel)
      zonegroup 7c622bcb-f89c-4e92-aec8-503f4b03b57b (petreloss)
           zone 33053987-0427-4808-bfc9-ba742eded198 (zone2)
  metadata sync syncing
                full sync: 0/64 shards
                incremental sync: 64/64 shards
                metadata is caught up with master
      data sync source: 8b322746-4cb8-44e9-8498-a99d56d41bef (zone1)
                        syncing
                        full sync: 0/128 shards
                        incremental sync: 128/128 shards
                        data is caught up with source
```


## 5.2 测试核对

- 在secondary zone上上传文件

```
radosgw-admin user create --uid=user1 --display-name="user1"
s3cmd mb s3://sync-bucket

#随便上传几个文件
s3cmd put test.5M.gz s3://sync-bucket/2020-09-21.test.5M.gz
s3cmd put test.5M.gz.CLOD s3://sync-bucket/2020-09-21.test.5M.gz.CLOD
s3cmd put user.md.json s3://sync-bucket/2020-09-21.user.md.json
```

- 再次分别查看master 和 secondary的同步状态

```
# master zone1
[root@ceph-3 ~]# radosgw-admin sync status
          realm 21f8c114-44da-4ff2-9a39-b8503b1f03ba (petrel)
      zonegroup 7c622bcb-f89c-4e92-aec8-503f4b03b57b (petreloss)
           zone 8b322746-4cb8-44e9-8498-a99d56d41bef (zone1)
  metadata sync no sync (zone is master)
      data sync source: 33053987-0427-4808-bfc9-ba742eded198 (zone2)
                        syncing
                        full sync: 0/128 shards
                        incremental sync: 128/128 shards
                        data is behind on 1 shards
                        behind shards: [112]
                        oldest incremental change not applied: 2020-09-20 22:53:24.0.065104s [112]                      
```
> data is caught up with source: 表示数据一致

- 查看下master zone/ secondary zone的池中是否数据同步

```
[root@ceph-3 ~]# rados ls -p zone1.rgw.buckets.data
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1_2020-09-21.test.5M.gz
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1_2020-09-21.user.md.json
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1_2020-09-21.test.5M.gz.CLOD
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1__shadow_.5zrK69P9CUwXs2LKDvNRxsKeY1eEkCj_1
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1__shadow_.P1UjP47iitHm-AGz1a9cZEa0S2CRDJL_1
```
```
[root@ceph-6 test_s3]# rados -p sh-zone2.rgw.buckets.data ls
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1_2020-09-21.test.5M.gz
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1__shadow_.P9F074SmDXYKY4rGsga2FSbCmj_-10d_1
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1_2020-09-21.user.md.json
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1__shadow_.KDcDWZUSFysHFkZkP1N0t3fVsZWtgFJ_1
8b322746-4cb8-44e9-8498-a99d56d41bef.24510.1_2020-09-21.test.5M.gz.CLOD
```
完全一样，测试验证通过

# 6 注意
> - 配置多个endpoint 用作数据同步，避免单个rgw压力过大
> - 客户端读写用的endpoint应该跟同步endpoint分离

# 7 报错处理
所有操作做完之后开始验证 radosgw-admin sync status, 也许会出现以下报错，这里仅给出可能解决报错的建议，具体问题还需具体分析：
出现以下报错后，首先看下集群状态是否正常：ceph -s

## 7.1 master zone： failed to retrieve sync info: (5) Input/output error
> - 可能的解决办法：
>> 查看下slave的rgw是否挂掉，若挂掉需重启。
> - 可能出现这个问题的原因：
>> 按官网文档，修改rgw zone之后是需要重启的，如果之前rgw zone配置的zone和当前的zone重名而没有重启，则将会导致rgw出错, 进而导致zone对应的pool无法建立，最终导致I/O error.

 
## 7.2 master/slave zone: permission denied 
> - 可能是解决办法（如果你是在测试环境中，则可以按如下办法进行）：
>> 清空 .rgw.root pool, 并重启rgw, 重新进行实验
> - 可能出现这个问题的原因：
>> .rgw.root pool中的记录或许存在之前某时间点的旧数据, 而master/slave使用的是不同的数据从而导致问题发生。
> - 生产环境勿这样操作
