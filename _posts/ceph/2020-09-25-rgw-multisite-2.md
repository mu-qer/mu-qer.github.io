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
这里有两个ceph集群，其中zone1将作为master, zone2将作为slave:
- 集群1包括：zone-1, rgw-1, rgw-2
- 集群2包括：zone-2, rgw-3, rgw-4

其中，rgw-1 和 rgw-3是两个集群用于sync的网关。rgw-2 和 rgw-4是两个集群各自对外提供服务的网关，具体信息如下：

| rgw name | address          |
| -------- | -------          |
| rgw-1    | 192.168.2.27:80  |
| rgw-2    | 192.168.2.40:80  | 
| rgw-3    | 192.168.2.166:80 |
| rgw-4    | 192.168.2.167:80 |

 
# 3.multisite 集群搭建
multisite集群模型如下:

![rgw-multisite-cluster](https://mu-qer.github.io/assets/img/ceph/2020-09-25-rgw-multisite-cluster-01.jpg)

## 3.1 创建system user,并更新zone1
如果本zone已经有系统级用户, 那么可以跳过4.1, 直接开始4.2操作

- [1] 创建同步sync的系统级用户

```
radosgw-admin user create --uid="sync-admin" --display-name="sync-admin" --system
```

- [2] 更新zone的key信息
access-key 和 secret 填写的是上一步创建的系统级用户的access-key和secret-key
```
radosgw-admin zone modify --rgw-zone=zone1 --access-key=xxx --secret=xxxx 
```

- [3] 更新period

```
radosgw-admin period update --commit
```

- [4] 更新ceph.conf, 找到对应的rgw配置区，并重启rgw实例

```
[client.rgw.dev-1]
rgw zone = zone1

systemctl list-units|grep rgw
systemctl restart ceph-rgw@.xxxxx.service
```

## 3.2 操作zone2

- [1] 从master zone拉取realm配置信息

```
radosgw-admin realm pull --url=http://192.168.2.27:80 --access-key=xxxxxxxx --secret=xxxxxxxx --rgw-realm=petrel
```
> 使用的access-key和secret就是同步的master中的系统用户的信息

- [2] 设置realm为default
```
radosgw-admin realm default --rgw-realm=petrel
```

- [2] 从master zone拉取period配置信息

```
radosgw-admin period pull --url=http://192.168.2.27:80 --access-key=xxxxxx --secret=xxxxxxxxx --rgw-realm=petrel
```
- [3] 将slave zone添加到zonegroup:petreloss中

```
radosgw-admin zonegroup add --rgw-zonegroup=petreloss --rgw-zone=zone2
```

- [4] 配置slave zone的endpoints, access-key, secret-key```
radosgw-admin zone modify --rgw-zone=zone2 --endpoints=http://192.168.2.166:80 --access-key=xxx --secret=xxx
```

- [5] 更新period

```
radosgw-admin period update --commit
```
- [6] 更新rgw配置，重启rgw

```
[client.rgw.dev-4]
rgw zone = zone2

systemctl list-units|grep rgw
systemctl restart ceph-rgw@.xxxxx.service
```

# 4 测试结果
做完以上部署后，可以直接观察原master zone/slave zone中的用户、bucket、数据文件的同步情况。
总结：
- master zone中的原有数据文件将会同步到 slave zone中
- master zone中的原有用户信息将会同步到 slave zone中
- master zone中的原有bucket信息将会同步到 slave zone 中
- slave zone中的原有数据文件、用户、bucket将不会同步到 master zone中


# 5 问题发现
再做扩展已有集群的同步测试中发现：
> master zone/slave zone的 placement 不同将不能进行数据同步，即：数据同步必须在相同的 placement-target中

