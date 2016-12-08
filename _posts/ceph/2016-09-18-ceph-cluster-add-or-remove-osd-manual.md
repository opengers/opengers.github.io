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

><small>系统centos7.2    
ceph版本 ceph version 10.2.2   
测试服务器为kvm虚拟机</small>

## 添加一个物理节点(node4)   

要使用`ceph-deploy`工具，首先需要cd到`ceph-deploy`部署目录下，这里为`data/ceph/deploy/`  

**环境设置**  

要添加一个新物理节点，首先需要设置新节点环境   

* ntp同步
* hostname设置
* hosts添加
* ceph源(手动设置ceph源，ceph-deploy使用的源连接慢)
* 配置ceph-deploy默认使用root用户登陆通过密钥登录node4
* 管理节点必须能够通过 SSH 无密码登陆node4。如果 ceph-deploy 以某个普通用户登录，那么这个用户必须有无密码使用sudo的权限。

以上具体请参考之前文章[使用ceph-deploy工具部署ceph集群](http://www.isjian.com/ceph/ceph-deploy-ceph-storage-cluster/)

**新节点安装依赖包**  

``` shell
yum install -y yum-utils snappy leveldb gdiskpython-argparse gperftools-libs ntpdate yum-plugin-priorities
```

**安装ceph软件包**

可以使用`ceph-deploy`工具远程安装 

``` shell
ceph-deploy install --no-adjust-repos node4

#若只是做为客户端执行ceph命令，不添加新节点，就只需安装命令行工具包即可
ceph-deploy install --no-adjust-repos --cli node5
```

还可以登录到`node4`上，手工执行安装   

``` shell
yum install ceph
```

**管理密钥**  

拷贝管理密钥环到node4，可以在`node4`上以管理员权限执行ceph命令  

``` shell
ceph-deploy admin node4
```

这里我们已经准备好了一个新节点，可以用来扩容osd  

## 使用ceph-deploy添加一个osd  

`node4`上环境设置好之后，我们可以用磁盘`vdb`添加一个osd节点    

``` shell
ceph-deploy osd prepare node4:vdb:/dev/vdf
ceph-deploy osd activate node4:vdb1:/dev/vdf1

#vdf1是osd节点使用的日志分区
```

## 手动添加一个osd

上面是通过`ceph-deploy`快速添加一个osd节点，下面我们手动添加另一个osd节点，首先需要登录`node4`主机   

我们可以用磁盘`vdc`手动添加一个osd节点，使用磁盘`vdf`的分区`vdf2`作为日志盘    

``` shell
#[id]可以不需要，osd id会自动分配，这个id即是下面步骤要使用的osd.16
ceph osd create [id]
16

#创建osd数据目录
#mkdir /var/lib/ceph/osd/{cluster-name}-{osd-number}
mkdir /var/lib/ceph/osd/ceph-16
chown ceph:ceph -R /var/lib/ceph/osd/ceph-16

#生成一个uuid
uuidgen
a6ad83ec-c161-4667-877f-d24087f38b12
#创建分区vdc1
#sgdisk是GPT分区工具 1:0:0表示创建第1个分区,使用整个磁盘
sgdisk --new=1:0:0 --change-name=1:'ceph data' --partition-guid=1:a6ad83ec-c161-4667-877f-d24087f38b12 --typecode=1:a6ad83ec-c161-4667-877f-d24087f38b12 --mbrtogpt -- /dev/vdc
#格式化
mkfs.xfs /dev/vdc1 -f
#挂载
mount /dev/vdc1 /var/lib/ceph/osd/ceph-16

#初始化osd数据目录
ceph-osd -i 16 --mkfs --mkkey
#--mkfs 创建空的object目录，也会初始化journal，此初始化的journal为文件
#--mkkey 创建osd key

#更改journal日志盘路径
#可能需要清除磁盘分区(重新添加时需要，这里不需要)
#sgdisk --zap-all --clear --mbrtogpt /dev/vdf
#生成一个uuid
uuidgen
b3897364-8807-48eb-9905-e2c8400d0cd4
#新建分区
#1:0:+7G 第一个分区，7G大小
sgdisk --new=0:0:+7G --change-name=1:'ceph journal' --partition-guid=0:b3897364-8807-48eb-9905-e2c8400d0cd4 --typecode=0:b3897364-8807-48eb-9905-e2c8400d0cd4 --mbrtogpt -- /dev/vdf
mkfs.xfs /dev/vdf1
cd /var/lib/ceph/osd/ceph-16
#删除默认生成的journal
rm -f journal
ln -s /dev/disk/by-partuuid/b3897364-8807-48eb-9905-e2c8400d0cd4 journal
chown ceph:ceph -R /var/lib/ceph/osd/ceph-16
chown ceph:ceph /var/lib/ceph/osd/ceph-16/journal
#osd执行不成功手动更改journal权限
#(ceph-osd --mkjournal -i 16 && chown ceph:ceph /var/lib/ceph/osd/ceph-16/journal)

#注册osd key
ceph auth add osd.16 osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-16/keyring

#创建client.bootstrap-osd key文件(本节点新建第一个osd时才需要)
ceph auth get-or-create client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
chmod 600 /var/lib/ceph/bootstrap-osd/ceph.keyring
chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring

#添加此新节点node4到crush map(本节点新建第一个osd时才需要)
ceph osd crush add-bucket node4 host
#移动到root:default,rack:rack01下面
ceph osd crush move node4 root=default rack=rack01

#添加此osd.16到CRUSH map，才能接收数据
ceph osd crush add osd.16 1.00 root=default rack=rack01 host=node4

#osd添加完成，目前处于down状态，启动osd.16之后，变为up且in
systemctl start ceph-osd@16
```

## 手动移除osd  

osd正常运行是up 且 in状态   

``` shell
#out之后，ceph开始重新平衡，拷贝此osd上数据到其它osd，此osd状态变为up且out
ceph osd out 10

#stop osd进程之后，状态变为down 且 out
systemctl stop ceph-osd@10

#删除 CRUSH 图的对应 OSD 条目，它就不再接收数据了
ceph osd crush remove osd.10

#移除osd认证key
ceph auth del osd.10

#从osd中删除osd 10，ceph osd tree中移除
ceph osd rm 10
```