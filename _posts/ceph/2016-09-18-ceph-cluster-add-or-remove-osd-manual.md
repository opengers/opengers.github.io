---
title: "ceph集群中进行osd的手动添加移除"
author: opengers
layout: post
permalink: /ceph/ceph-cluster-add-or-remove-osd-manual/
categories: ceph
tags:
  - ceph-deploy
  - cluster
---

1.使用ceph-deploy添加一个新节点(node4)
预检 - node4
ntp同步
hostname设置
hosts添加
ceph源(手动设置ceph源，ceph-deploy使用的源连接慢)
ceph-deploy默认使用root用户登陆
admin-node密钥登录node4
管理节点必须能够通过 SSH 无密码登陆node4。如果 ceph-deploy 以某个普通用户登录，那么这个用户必须有无密码使用 sudo 的权限。
从 Infernalis 版起，用户名 “ceph” 保留给了 Ceph 守护进程。如果 Ceph 节点上已经有了 “ceph” 用户，升级前必须先删掉这个用户

依赖包 - node4
yum install -y yum-utils snappy leveldb gdiskpython-argparse gperftools-libs ntpdate yum-plugin-priorities

安装ceph软件包 - node4
首先规划osd磁盘，以及journal盘位置
管理节点上执行ceph-deploy
ceph-deploy install --no-adjust-repos node4
#若只是做为客户端，就只安装mon组件
ceph-deploy install --no-adjust-repos --mon 192.168.6.161

拷贝管理密钥环到node4
ceph-deploy admin node4
这样node4上可以以管理员权限执行ceph命令



2.ceph-deploy添加一个osd
node4上可以执行ceph命令之后
ceph-deploy osd prepare node4:vdb:/dev/vdf
ceph-deploy osd activate node4:vdb1:/dev/vdf1



2.手动添加一个osd
第1步ceph新节点node5启用之后，登录新节点node5手动添加一个osd进程
ceph osd create [id]
16
osd id会自动分配，根据自动分配的osd，设置下面使用的osd

#创建osd数据目录
mkdir /var/lib/ceph/osd/{cluster-name}-{osd-number}
mkdir /var/lib/ceph/osd/ceph-16
chown ceph:ceph -R /var/lib/ceph/osd/ceph-16

#格式化osd所用磁盘vdb
#sgdisk是GPT分区工具 1:0:0表示创建第1个分区,使用整个磁盘
sgdisk --new=1:0:0 --change-name=1:'ceph data' --partition-guid=1:a6ad83ec-c161-4667-877f-d24087f38b12 --typecode=1:a6ad83ec-c161-4667-877f-d24087f38b12 --mbrtogpt -- /dev/vdb
mkfs.xfs /dev/vdb1 -f
mount /dev/vdb1 /var/lib/ceph/osd/ceph-16

#初始化osd数据目录
ceph-osd -i 16 --mkfs --mkkey
cd /var/lib/ceph/osd/ceph-16
rm -f /var/lib/ceph/osd/ceph-16/journal

#更改journal日志盘路径
#清楚磁盘其它分区(重新添加时需要)
sgdisk --zap-all --clear --mbrtogpt /dev/vdf
#新建分区
uuidgen
b3897364-8807-48eb-9905-e2c8400d0cd4
#1:0:+7G 第一个分区，7G大小
sgdisk --new=1:0:+7G --change-name=1:'ceph journal' --partition-guid=1:b3897364-8807-48eb-9905-e2c8400d0cd4 --typecode=1:b3897364-8807-48eb-9905-e2c8400d0cd4 --mbrtogpt -- /dev/vdf
mkfs.xfs /dev/vdf1
ln -s /dev/disk/by-partuuid/b3897364-8807-48eb-9905-e2c8400d0cd4 journal
chown ceph:ceph -R /var/lib/ceph/osd/ceph-16
chown ceph:ceph /var/lib/ceph/osd/ceph-16/journal
#osd执行不成功手动更改journal权限
(ceph-osd --mkjournal -i 16 && chown ceph:ceph /var/lib/ceph/osd/ceph-16/journal)

#注册osd key
ceph auth add osd.16 osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-16/keyring

#创建client.bootstrap-osd key文件(本节点新建第一个osd时才需要)
ceph auth get-or-create client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
chmod 600 /var/lib/ceph/bootstrap-osd/ceph.keyring
chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring

#添加此新节点node5到crush map(本节点新建第一个osd时才需要)
ceph osd crush add-bucket sata-node5 host
#移动到root:staa,rack:stat-rack01下面
ceph osd crush move sata-node5 root=sata rack=sata-rack01

#添加此osd.16到CRUSH map，才能接收数据
ceph osd crush add osd.16 0.01900 root=sata rack=sata-rack01 host=sata-node5

#osd添加完成，目前处于down状态，启动osd.16之后，变为up且in
systemctl start ceph-osd@16


2.手动移除osd
osd正常运行是up 且 in
ceph osd out {osd-num}
out之后，ceph开始重新平衡，拷贝此osd上数据到其它osd，状态变为up且out

systemctl stop ceph-osd@{osd-num}
stop osd进程之后，状态变为down

ceph osd crush remove osd.10
删除 CRUSH 图的对应 OSD 条目，它就不再接收数据了

ceph auth del osd.10
移除osd认证key

ceph osd rm 10
删除osd 10，ceph osd tree中移除 