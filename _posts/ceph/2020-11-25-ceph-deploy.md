---
layout: post
title: ceph-ploy使用
date: 2020-11-25 23:30:09
categories: Ceph
description: ceph-deploy
tags: Ceph
---

做之前需要清理一下ceph原有的配置：
ceph-deploy purge SH-IDC1-10-121-2-165 SH-IDC1-10-121-2-167 SH-IDC1-10-121-2-169
ceph-deploy purgedata SH-IDC1-10-121-2-165 SH-IDC1-10-121-2-167 SH-IDC1-10-121-2-169
-------------------------------------------------------------------------------------------
#mons
ceph-deploy new SH-IDC1-10-121-2-165 SH-IDC1-10-121-2-167 SH-IDC1-10-121-2-169

ceph-deploy mon create-initial 

ceph-deploy admin SH-IDC1-10-121-2-165 SH-IDC1-10-121-2-167 SH-IDC1-10-121-2-169

#mgr
ceph-deploy mgr create SH-IDC1-10-121-2-165 SH-IDC1-10-121-2-167 SH-IDC1-10-121-2-169


#osd
ceph-deploy disk list SH-IDC1-10-121-2-165 SH-IDC1-10-121-2-167 SH-IDC1-10-121-2-169

{
[SH-IDC1-10-121-2-165][INFO  ] Running command: fdisk -l
[SH-IDC1-10-121-2-165][INFO  ] Disk /dev/sda: 599.6 GB, 599584145408 bytes, 1171062784 sectors
[SH-IDC1-10-121-2-165][INFO  ] Disk /dev/nvme0n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-165][INFO  ] Disk /dev/nvme2n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-165][INFO  ] Disk /dev/nvme3n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-165][INFO  ] Disk /dev/nvme4n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-167][DEBUG ] connected to host: SH-IDC1-10-121-2-167
[SH-IDC1-10-121-2-167][DEBUG ] detect platform information from remote host
[SH-IDC1-10-121-2-167][DEBUG ] detect machine type
[SH-IDC1-10-121-2-167][DEBUG ] find the location of an executable
[SH-IDC1-10-121-2-167][INFO  ] Running command: fdisk -l
[SH-IDC1-10-121-2-167][INFO  ] Disk /dev/sda: 599.6 GB, 599584145408 bytes, 1171062784 sectors
[SH-IDC1-10-121-2-167][INFO  ] Disk /dev/nvme1n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-167][INFO  ] Disk /dev/nvme3n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-167][INFO  ] Disk /dev/nvme2n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-167][INFO  ] Disk /dev/nvme0n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-169][DEBUG ] connected to host: SH-IDC1-10-121-2-169
[SH-IDC1-10-121-2-169][DEBUG ] detect platform information from remote host
[SH-IDC1-10-121-2-169][DEBUG ] detect machine type
[SH-IDC1-10-121-2-169][DEBUG ] find the location of an executable
[SH-IDC1-10-121-2-169][INFO  ] Running command: fdisk -l
[SH-IDC1-10-121-2-169][INFO  ] Disk /dev/sda: 599.6 GB, 599584145408 bytes, 1171062784 sectors
[SH-IDC1-10-121-2-169][INFO  ] Disk /dev/nvme1n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-169][INFO  ] Disk /dev/nvme2n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-169][INFO  ] Disk /dev/nvme0n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
[SH-IDC1-10-121-2-169][INFO  ] Disk /dev/nvme3n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
}


ceph-deploy osd create --data /dev/nvme0n1 SH-IDC1-10-121-2-165
{
	ceph-deploy osd create --data {target-device} {target-host} 命令即可新增 OSD
}

解决 overwrite-conf报错：
ceph-deploy --overwrite-conf osd create --data /dev/nvme0n1 SH-IDC1-10-121-2-167

如果遇到
{
	[SH-IDC1-10-121-2-167][DEBUG ] find the location of an executable
[SH-IDC1-10-121-2-167][INFO  ] Running command: /usr/sbin/ceph-volume lvm zap /dev/nvme1n1
[SH-IDC1-10-121-2-167][WARNIN] -->  RuntimeError: command returned non-zero exit status: 1
[SH-IDC1-10-121-2-167][DEBUG ] --> Zapping: /dev/nvme1n1
[SH-IDC1-10-121-2-167][DEBUG ] --> --destroy was not specified, but zapping a whole device will remove the partition table
[SH-IDC1-10-121-2-167][DEBUG ] Running command: wipefs --all /dev/nvme1n1
[SH-IDC1-10-121-2-167][DEBUG ]  stderr: wipefs: error: /dev/nvme1n1: probing initialization failed: Device or resource busy
[SH-IDC1-10-121-2-167][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/ceph-volume lvm zap /dev/nvme1n1
}

需要在/dev/disk/by-id下查找对应的磁盘信息
ls -la

dm-name-ceph-xxxxx --> ../../dm-3

然后把对应的dm-name-ceph--xxxx删掉，然后删掉/dev/dm-3, 最后删掉 /dev/mapper/dm-name-ceph--xxxxx
再重启即可。



rbd create -p mdt_pool --image mdt_rbd -s 1T
rbd create -p ${pool_name} --image ${image_name} -s ${size}

查看：
rbd -p ost_pool list

===============================

rbd map mdt_rbd --pool mdt_pool
{
	rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable mdt_pool/mdt_rbd object-map fast-diff deep-flatten".
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address
}

解决：
rbd feature disable mdt_rbd exclusive-lock, object-map, fast-diff, deep-flatten --pool mdt_pool

