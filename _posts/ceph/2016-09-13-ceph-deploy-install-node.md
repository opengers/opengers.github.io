---
title: "使用ceph-deploy工具部署ceph集群"
author: opengers
layout: post
permalink: /ceph/ceph-deploy-ceph-storage-cluster/
categories: ceph
tags:
  - ceph-deploy
  - cluster
format: quote
---

><small>系统centos7.2    
ceph版本 ceph version 10.2.2   
测试服务器为kvm虚拟机</small>

ceph的部署过程在官网有[详细记录](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/)，[这里](http://docs.ceph.org.cn/start/quick-start-preflight/)也有翻译的中文文档，本篇文章是记录下自己的部署过程，服务器使用kvm虚拟机，只测试功能，分配如下  
节点 | 服务 | cluster network | public network |
:-- | :-- | :-- | :-- | 
node1 | osd.{1,2,3,4},mon.node1  |  eth1:172.31.6.174/24  | 192.168.6.174/22 |
node2 | osd.{5,6,7,8},mon.node2  |  eth1:172.31.6.175/24  | 192.168.6.175/22 |
node3 | osd.{9,10,11,12},mon.node3  | eth1:172.31.6.176/24 | 192.168.6.176/22 |
{:.mbtablestyle}      

1.硬件

ceph为廉价普通硬件而设计，在其上可以构建PB级的数据集群，其数据高可靠性主要依靠软件层良好的算法来保障。因为是廉价硬件，因此在一个大型集群中，硬件故障会很频繁发生，对于一个设计良好的ceph集群，这并不会对集群造成任何影响

Ceph 元数据服务器对 CPU 敏感，它会动态地重分布它们的负载，所以你的元数据服务器应该有足够的处理能力
Ceph 的 OSD 运行着 RADOS 服务、用 CRUSH 计算数据存放位置、复制数据、维护它自己的集群运行图副本，因此 OSD 需要一定的处理能力（如双核 CPU ）
监视器只简单地维护着集群运行图的副本，因此对 CPU 不敏感；但必须考虑机器以后是否还会运行 Ceph 监视器以外的 CPU 密集型任务

元数据服务器和监视器必须可以尽快地提供它们的数据，所以他们应该有足够的内存，至少每进程 1GB
OSD 的日常运行不需要那么多内存（如每进程 500MB ）差不多了；然而在恢复期间它们占用内存比较大（如每进程每 TB 数据需要约 1GB 内存）。通常内存越多越好
btrfs 尚未稳定到可以用于生产环境的程度，但它可以同时记日志并写入数据，而 xfs 和 ext4 却不能。

单个驱动器容量越大，其对应的 OSD 所需内存就越大，特别是在重均衡、回填、恢复期间。根据经验， 1TB 的存储空间大约需要 1GB 内存。

Ceph 最佳实践指示，你应该分别在单独的硬盘运行操作系统、 OSD 数据和 OSD 日志。

你可以在同一主机上运行多个 OSD ，但要确保 OSD 硬盘总吞吐量不超过为客户端提供读写服务所需的网络带宽；还要考虑集群在每台主机上所存储的数据占总体的百分比，如果一台主机所占百分比太大而它挂了，就可能导致诸如超过 full ratio 的问题，此问题会使 Ceph 中止运作以防数据丢失。

OSD 数量较多（如 20 个以上）的主机会派生出大量线程，尤其是在恢复和重均衡期间。很多 Linux 内核默认的最大线程数较小（如 32k 个），如果您遇到了这类问题，可以把 kernel.pid_max 值调高些。理论最大值是 4194303 。例如把下列这行加入 /etc/sysctl.conf 文件
kernel.pid_max = 4194303

2.架构
admin-node：ceph-deploy部署节点
node1，node2，node3  存储集群，3 mon

3.预检 - node
ntp同步
hostname设置
hosts添加
ceph源(手动设置ceph源，ceph-deploy使用的源连接慢)
admin-node密钥登录各个node(ceph-deploy部署)

4.依赖包 - node
yum install -y yum-utils snappy leveldb gdiskpython-argparse gperftools-libs ntpdate
#repo优先级
yum -y install yum-plugin-priorities
从 Infernalis 版起，用户名 “ceph” 保留给了 Ceph 守护进程。如果 Ceph 节点上已经有了 “ceph” 用户，升级前必须先删掉这个用户

5.安装ceph-deploy工具 - admin-node
yum install ceph-deploy
管理节点必须能够通过 SSH 无密码地访问各 Ceph 节点。如果 ceph-deploy 以某个普通用户登录，那么这个用户必须有无密码使用 sudo 的权限。

6.安装ceph软件包 - node
首先规划osd磁盘，以及journal盘位置
ceph-deploy install --no-adjust-repos node1 node2 node3

#######
ceph-deploy uninstall {hostname [hostname] ...}
ceph-deploy purge {hostname [hostname] ...}

7.清除数据
如果只想清除 /var/lib/ceph 下的数据、并保留 Ceph 安装包
ceph-deploy purgedata {hostname} [{hostname} ...]

要清理掉 /var/lib/ceph 下的所有数据、并卸载 Ceph 软件包，用 purge 命令
ceph-deploy purge {hostname} [{hostname} ...]

#清除磁盘 zap
ceph-deploy disk list {node-name [node-name]...}
ceph-deploy disk zap {osd-server-name}:{disk-name}

8.创建ceph集群 - node
ceph-deploy new --cluster-network 172.31.6.0/24 --public-network 192.168.4.0/22 node1 node2 node3

#######
这会创建一个ceph.conf配置文件和一个监视器密钥环
ceph.conf至少包括
fsid
mon_initial_members
mon_host

默认ceph使用集群名ceph，可以下面自定义集群名称
ceph-deploy --cluster {cluster-name} new {host [host], ...}
多个集群注意端口不要冲突

Ceph Monitors 之间默认使用 6789 端口通信， OSD 之间默认用 6800:7300 这个范围内的端口通信

9.添加mons
ceph-deploy mon create node1 node2 node3

##########
write cluster configuration to /etc/ceph/{cluster}.conf
/var/lib/ceph/mon/ceph-node1/keyring
systemctl enable ceph-mon@node1
systemctl start ceph-mon@node1

在一主机上新增监视器时，如果它不是由 ceph-deploy new 命令所定义的，那就必须把 public network 加入 ceph.conf 配置文件

ceph-deploy mon destroy {host-name [host-name]...}
确保你删除一监视器后，其余监视器仍能达成一致。如果不可能，删除它之前先加一个。

10.key管理
ceph-deploy gatherkeys node1

########
在准备一台主机作为 OSD 或元数据服务器时，你得收集monitor keys and the OSD and MDS bootstrap keyrings初始密钥环

不再使用 ceph-deploy （或另建一集群）时，你应该删除管理主机上、本地目录中的密钥。可用下列命令：
ceph-deploy forgetkeys

11.osd管理
创建集群，安装ceph包，收集密钥之后，就可以创建osd了
ceph-deploy目录下ceph.conf
-----------------------
osd_journal_size = 7168
osd_pool_default_size = 2

osd_pool_default_pg_num = 512
osd_pool_default_pgp_num = 512
rbd_default_features = 3
-----------------------
ceph-deploy osd prepare node1:vdb:/dev/vdf node1:vdc:/dev/vdf node1:vdd:/dev/vdf node1:vde:/dev/vdf node2:vdb:/dev/vdf node2:vdc:/dev/vdf node2:vdd:/dev/vdf node2:vde:/dev/vdf node3:vdb:/dev/vdf node3:vdc:/dev/vdf node3:vdd:/dev/vdf node3:vde:/dev/vdf
prepare 命令只准备 OSD 。在大多数操作系统中，硬盘分区创建后，不用 activate 命令也会自动执行 activate 阶段（通过 Ceph 的 udev 规则）

ceph-deploy osd activate node1:vdb1:/dev/vdf1 node1:vdc1:/dev/vdf2 node1:vdd1:/dev/vdf3 node1:vde1:/dev/vdf4 node2:vdb1:/dev/vdf1 node2:vdc1:/dev/vdf2 node2:vdd1:/dev/vdf3 node2:vde1:/dev/vdf4 node3:vdb1:/dev/vdf1 node3:vdc1:/dev/vdf2 node3:vdd1:/dev/vdf3 node3:vde1:/dev/vdf4
activate 命令会让 OSD 进入 up 且 in 状态，此命令所用路径和 prepare 相同

###########
/usr/bin/systemctl enable ceph-osd@4
/usr/bin/systemctl start ceph-osd@4

在一个节点运行多个 OSD 守护进程、且多个 OSD 守护进程共享一个日志分区时，你应该考虑整个节点的最小 CRUSH 故障域，因为如果这个 SSD 坏了，所有用其做日志的 OSD 守护进程也会失效

12.修改配置同步
要允许一主机以管理员权限执行 Ceph 命令，用 admin 命令
ceph-deploy admin {host-name [host-name]...}

要把改过的配置文件分发给集群内各主机，可用 config push 命令。
ceph-deploy --overwrite-conf config push node{1..3}

