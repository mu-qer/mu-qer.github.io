---
layout: post
title: rgw-multisite
date: 2020-09-15 23:30:09
categories: ceph
description: ceph-rgw多活
tags:
- Ceph
- linux
- 网络编程
- 存储引擎
---


[TOC]

# 1.multisite 说明
> 多数据中心 (multisite) 旨在实现异地双活，提供了备份容灾的能力。 

> 主节点在对外提供服务时，用户数据在主节点落盘后即向用户回应“写成功”应答，然后实时记录数据变化的相关日志信息。备节点则实时比较主备数据差异，并及时将差异化数据拉回备节点。异步复制技术适用于远距离的容灾方案，对系统性能影响较小。

# 2.概念说明
- realm代表一个命名空间, 一个 realm 包含一个或多个zonegroup (zg之前被称为 region, 一般表示一个数据中心), 一个zg包含一个或多个zone, zone包含bucket, bucket中存放数据

> 每个realm都有与之对应period（表示一个realm的有效期）。每个period及时地代表了zonegroup的状态和zone的配置。每次需要对zonegroup或者zone做修改的时，需要更新period，并提交。
- 单个数据中心的配置一般由一个zonegroup组成，这个zonegroup包含一个zone和一个或者多个rgw实例。在这些rgw中可以平衡网关请求。
> - 元数据在同一realm的zone之间进行同步
> - 实体数据只在同一zg中的主zone和从zone之间同步,无法跨zg访问实体数据信息。

# 3.multisite 结构
- zone: 对应一个独立的集群，有一组 RGW 对外提供服务
- zonegroup: 每个zg可包含多个 zone, zone之间同步数据和元数据
- realm: 每个realm包含多个zg, zg之间同步数据
结构如下：
![rgw-multisite-struct](https://mu-qer.github.io/assets/img/ceph/2020-09-15-rgw-multisite-struct-01.JPG)

> master zone 和 secondly zone 有两种模式：active-active 和 active-passive。
> - active-active 模式下master和slave 都可以读写，数据会自动同步 
> - active-passive 下，只能在master 写入
 
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
radosgw-admin zonegroup create --rgw-zonegroup=petreloss --endpoints=http://10.5.8.242 --rgw-realm=petrel --master --default
```
- [3] 创建master zone
```
radosgw-admin zone create --rgw-zonegroup=petreloss --rgw-zone=zone1 --master --default --endpoints=https://10.5.8.242:80
```
- [4] 删除默认产生的pool,zone,zonegroup等
```
radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
radosgw-admin period update --commit
radosgw-admin zone delete --rgw-zone=default
radosgw-admin period update --commit
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

- [7] 更新ceph.conf, 找到对应的rgw配置区，并重启rgw实例
```
[client.rgw.dev-1]
rgw_zone = zone1
```

## 4.2 创建secondary zone
这里有两个集群，master zone, slave zone, 但元数据的操作必须在master zone中执行，如：创建用户或者bucket, 在secondary zone中执行会被重定向到master zone中。

- [1] 从master zone拉去realm配置信息
```
radosgw-admin realm pull --url=http://10.5.8.242:80 --access-key=xxxxxxxx --secret=xxxxxxxx --rgw-realm=petrel
```
> 使用的access-key和secret就是同步的系统用户的信息
- [2] 从master zone拉取period配置信息
```
radosgw-admin period pull --url=http://10.5.8.242:80 --access-key=xxxxxx --secret=xxxxxxxxx --rgw-realm=petrel
```
- [3] 创建secondary zone
```
radosgw-admin zone create --rgw-zonegroup=petreloss --rgw-zone=zone2  --access-key=xxxxxxxx --secret=xxxxxxxxxxx --endpoints=http://10.5.8.242:80
```

- [4] 删掉 secondary zone上的default zone 还有pool

```
radosgw-admin zonegroup remove --rgw-zonegroup=default --rgw-zone=default
radosgw-admin period update --commit
radosgw-admin zone delete --rgw-zone=default
radosgw-admin period update --commit
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
rgw_zone = zone2
```

# 5 check 同步操作
## 5.1 查看 sync 的状态
- master zone同步状态
```
radosgw-admin sync status
TODO: 代码截图
```
- secondary zone同步状态
```
radosgw-admin sync status
TODO: 代码截图
```
> 你会发现这里有很多error，日志中也会出现mdlog, datalog not found的问题，这个时候不要慌，这是因为, 没有上传数据的时候，mdlog和datalog有很多shard 是空的，只要上传多一部分数据就可以

## 5.2 测试核对
- 在secondary zone上上传 50 MB 文件
```
radosgw-admin user create --uid=user1 --display-name="user1"
s3cmd mb s3://test1
dd if=/dev/zero of=test.50M.gz bs=5M count=10
s3cmd  put test.100M.gz s3://test1
```
- 再次分别查看master 和 secondary的同步状态
```
radosgw-admin sync status
TODO: master结果
radosgw-admin sync status
TODO: secondary 结果
```
> data is caught up with source: 表示数据一致

- 查看下master zone/ secondary zone的池中是否数据同步

```
rados ls -p zone1.rgw.buckets.data
TODO: master 结果
```
```
rados ls -p zone1.rgw.buckets.data
TODO: secondary 结果
```
完全一样，测试验证通过

# 6 注意
> - 配置多个endpoint 用作数据同步，避免单个rgw压力过大
> - 客户端读写用的endpoint应该跟同步endpoint分离


 
