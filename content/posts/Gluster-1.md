+++
date = '2024-12-12T18:03:23+08:00'
draft = false
title = 'GlusterFS高可用逻辑'
+++

1. 卷信息如下

```bash

$gluster volume info test-pv0


VolumeName:test-pv0

Type:Replicate

VolumeID:23e35a9e-71cd-4f19-b2bf-715a5fdca9c4

Status:Started

SnapshotCount:0

NumberofBricks:1x3=3

Transport-type:tcp

Bricks:

Brick1:10.10.101.109:/data11/test-pv0

Brick2:10.10.101.155:/data11/test-pv0

Brick3:10.10.101.161:/data11/test-pv0

OptionsReconfigured:

transport.address-family:inet

storage.fips-mode-rchecksum:on

nfs.disable:on

performance.client-io-threads:off

```

1. 停掉10.10.101.155的brick
2. 挂载volume并往test文件中追加数据
3. 查看三个brick中数据信息

```bash

[root@10.10.101.109 ~]# getfattr -d -m . -e hex /data11/test-pv0/test

getfattr:Removingleading'/'fromabsolutepathnames

# file: data11/test-pv0/test

trusted.afr.dirty=0x000000000000000000000000

trusted.afr.test-pv0-client-1=0x000000030000000000000000//说明其他brick上有三次数据写入操作未成功

trusted.gfid=0xbd2c2f5bcde740789b8cf32a4dd0ab1d

trusted.gfid2path.68e44df6666f9ccc=0x30303030303030302d303030302d303030302d303030302d3030303030303030303030312f74657374

trusted.glusterfs.mdata=0x0100000000000000000000000065b2043e000000000f98d9f40000000065b2043e000000000f98d9f40000000065b0b922000000002b6bb31a


[root@10.10.101.155 ~]# getfattr -d -m . -e hex /data11/test-pv0/test

getfattr:Removingleading'/'fromabsolutepathnames

# file: data11/test-pv0/test

trusted.afr.dirty=0x000000000000000000000000

trusted.gfid=0xbd2c2f5bcde740789b8cf32a4dd0ab1d

trusted.gfid2path.68e44df6666f9ccc=0x30303030303030302d303030302d303030302d303030302d3030303030303030303030312f74657374

trusted.glusterfs.mdata=0x0100000000000000000000000065b1ffba00000000010c9c9e0000000065b1ffba00000000010c9c9e0000000065b0b922000000002b6bb31a


[root@10.10.101.161 ~]# getfattr -d -m . -e hex /data11/test-pv0/test

getfattr:Removingleading'/'fromabsolutepathnames

# file: data11/test-pv0/test

trusted.afr.dirty=0x000000000000000000000000

trusted.afr.test-pv0-client-1=0x000000030000000000000000//说明其他brick上有三次数据写入操作未成功

trusted.gfid=0xbd2c2f5bcde740789b8cf32a4dd0ab1d

trusted.gfid2path.68e44df6666f9ccc=0x30303030303030302d303030302d303030302d303030302d3030303030303030303030312f74657374

trusted.glusterfs.mdata=0x0100000000000000000000000065b2043e000000000f98d9f40000000065b2043e000000000f98d9f40000000065b0b922000000002b6bb31a


//追加数据只有前4byte数值+1

```

1. 新建文件test1并追加数据，前中4byte数值+1

```bash

[root@10.10.101.109 ~]# getfattr -d -m . -e hex /data11/test-pv0/test1

getfattr:Removingleading'/'fromabsolutepathnames

# file: data11/test-pv0/test1

trusted.afr.dirty=0x000000000000000000000000

trusted.afr.test-pv0-client-1=0x000000020000000200000000

trusted.gfid=0xf23be806564b4b39a7b07b60e1c3c41d

trusted.gfid2path.b1156a142bc44d74=0x30303030303030302d303030302d303030302d303030302d3030303030303030303030312f7465737431

trusted.glusterfs.mdata=0x0100000000000000000000000065b2055c0000000023eabb7d0000000065b204fc00000000289ebce50000000065b204fc0000000028746879

```

1. 新建目录，中后4 byte数值各+1

```bash

[root@10.10.101.109 ~]# getfattr -d -m . -e hex /data11/test-pv0/dir1/

getfattr:Removingleading'/'fromabsolutepathnames

# file: data11/test-pv0/dir1/

trusted.afr.test-pv0-client-1=0x000000000000000100000001

trusted.gfid=0x68cc6ff0323847ff9d8f94fcf15cb089

trusted.glusterfs.dht=0x000000000000000000000000ffffffff

trusted.glusterfs.mdata=0x0100000000000000000000000065b20aa40000000018f5df060000000065b20aa40000000018f5df060000000065b20aa40000000018f5df06

```

### 背景知识：

在GlusterFS中，读写操作由对应的模块（translator）完成，每个写操作可以拆为5个步骤：

1. 获取文件锁：对即将修改的文件进行加锁操作，防止有其他的写操作。
2. 预处理阶段：在所有brick上，对文件的’dirty’属性值+1。
3. 执行阶段：执行文件的修改操作，如写入或修改文件属性等。
4. 写完成阶段：对写入操作执行成功的brick，将文件的’dirty’属性值-1。同时当写入操作执行失败时，其他写入成功的brick中文件的’pending‘属性（trusted.afr.$VOLNAME-client-x）值+1，以标注有副本写入失败了，需后续处理。
5. 释放锁：释放1步中的锁。

> 注意：代码层面会做一些优化来减少频繁的锁操作。
