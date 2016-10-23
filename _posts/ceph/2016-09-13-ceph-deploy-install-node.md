---
title: "使用ceph-deploy工具部署ceph集群"
author: opengers
layout: post
permalink: /ceph/ceph-deploy-ceph-storage-cluster/
categories: ceph
tags:
  - ceph-deploy
  - cluster
---

><small>系统centos7.2    
ceph版本 ceph version 10.2.2   
测试服务器为kvm虚拟机</small>
 
* TOC
{:toc}  

[手工部署ceph集群](http://www.isjian.com/ceph/deploy-a-ceph-cluster-manually/)  
[ceph-deploy部署ceph集群](http://www.isjian.com/ceph/deploy-a-ceph-cluster-use-ceph-deploy/)   

ceph的部署过程在官网有[详细记录](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/)，[ceph中国社区](http://docs.ceph.org.cn/start/quick-start-preflight/)也有翻译的中文文档  

## 硬件     

ceph为廉价普通硬件而设计，在其上可以构建PB级的数据集群，其数据高可靠性主要依靠软件层良好的算法来保障。因为是廉价硬件，因此在一个大型集群中，硬件故障会很频繁发生，对于一个设计良好的ceph集群，这并不会对集群造成数据丢失    

本篇文章是记录下自己的部署过程，服务器使用kvm虚拟机，只测试功能，服务器分配如下    

----

节点 | 服务 | cluster network | public network
:-- | :-- | :-- | :--
node1(admin-node) | osd.{1,2,3,4},mon.node1  |  eth1:172.31.6.174/24  | eth0:192.168.6.174/22
node2 | osd.{5,6,7,8},mon.node2  |  eth1:172.31.6.175/24  | eth0:192.168.6.175/22
node3 | osd.{9,10,11,12},mon.node3  | eth1:172.31.6.176/24 | eth0:192.168.6.176/22

----

每个节点都有五块磁盘，前4块磁盘部署4个`osd`，第5块磁盘建立四个相等大小分区作为四个osd盘的日志分区。这样，集群共有12个`osd`进程，3个`monitor`进程。管理节点用作执行`ceph-deploy`命令，可以使用node1节点充当       

cluster network 是处理osd间的数据复制，数据重平衡，osd进程心跳检测的网络，其不对外提供服务，只在各个osd节点间通信，本文使用eth1网卡作为cluster network，三个节点网卡eth1桥接到同一个网桥br1上     

## 环境预检  

部署集群之前，需要进行环境准备，这些步骤应当设置于集群所有节点    
 
**ntp同步**  

各osd节点间需要设置时间同步，节点时钟偏差过大会引起pg异常  

**hostname设置**  

Cluster Map中会使用主机hostname作为名称表示，因此hostname需要好好规划  

**hosts添加**  

每个节点都添加集群所有节点的hosts，`ceph.conf`配置文件中会使用到，如下是`node1`节点的hosts  

``` shell
cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
#::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.31.6.174 node1
172.31.6.175 node2
172.31.6.176 node3
```

**配置ceph源**  

不要指望使用[ceph官方源](http://download.ceph.com/rpm-jewel/el7/)，这里需要使用国内第三方源，比如163的ceph源   

``` shell
[ceph]
name=Ceph packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/
enabled=1
gpgcheck=1
priority=2
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/
enabled=1
gpgcheck=1
priority=2
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/SRPMS/
enabled=0
gpgcheck=1
priority=2
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc
```

注意这里`priority`可以设置yum源的优先级，如果你其它源中也有ceph软件包，需要保证这里的`ceph.repo`的优先级`priority`比其它的高(值越小越高)，为此需要安装下面这个软件包，用以支持yum源的优先级  

``` shell
yum -y install yum-plugin-priorities
``` 

下面这些依赖包也需要在每个节点都安装   

``` shell
yum install -y yum-utils snappy leveldb gdiskpython-argparse gperftools-libs ntpdate
```

**内核调整**  

`osd`进程可以产生大量线程，如有需要，可以调整下内核最大允许线程数  

``` shell  
kernel.pid_max = 4194303
```

## 安装ceph-deploy工具     

`ceph-deploy`是ceph官方提供的部署工具，它通过ssh远程登录其它各个节点上执行命令完成部署过程，我们可以随意选择一台服务器安装此工具，为方便，这里我们选择`node1`节点安装`ceph-deploy`   

我们把`node1`节点上的`/data/ceph/deploy`目录作为`ceph-deploy`部署目录，其部署过程中生成的配置文件，key密钥，日志等都位于此目录下，因此下面部署应当始终在此目录下进行  

``` shell
yum install ceph-deploy

mkdir -p /data/ceph/deploy
```

`ceph-deploy`工具默认使用root用户SSH到各Ceph节点执行命令。为了方便，可以配置`ceph-deploy`免密码登陆各个节点。如果ceph-deploy以某个普通用户登陆，那么这个用户必须有无密码使用sudo的权限。  

## 安装ceph集群      

#### ceph软件包安装  

首先安装ceph软件包到三个节点上。上面我们已经配置好ceph源，因此这里使用`--no-adjust-repos`参数忽略设置ceph源   

``` shell
ceph-deploy install --no-adjust-repos node1 node2 node3
```

#### 创建ceph集群   

``` shell
ceph-deploy new --cluster-network 172.31.6.0/24 --public-network 192.168.4.0/22 node1 node2 node3

#上步会创建一个ceph.conf配置文件和一个监视器密钥环到各个节点的/etc/ceph/目录，ceph.conf中至少包括`fsid`，`mon_initial_members`，`mon_host`三个参数  

#默认ceph使用集群名ceph，可以使用下面命令创建一个指定的ceph集群名称
#ceph-deploy --cluster {cluster-name} new {host [host], ...}
```

`Ceph Monitors`之间默认使用`6789`端口通信， OSD之间默认用`6800:7300` 范围内的端口通信，多个集群应当保证端口不冲突  

#### 添加mons   

我们这里创建三个Monitor    

``` shell
ceph-deploy mon create node1 node2 node3

#上面命令效果如下
#1.write cluster configuration to /etc/ceph/{cluster}.conf
#2.生成/var/lib/ceph/mon/ceph-node1/keyring
#3.systemctl enable ceph-mon@node1
#4.systemctl start ceph-mon@node1
```

在一主机上新增监视器时，如果它不是由`ceph-deploy new`命令所定义的，那就必须把`public network`加入 ceph.conf配置文件    

#### key管理   

为节点准备认证key  

``` shell
ceph-deploy gatherkeys node1

#若有需要，可以删除管理主机上、本地目录中的密钥。可用下列命令：
#ceph-deploy forgetkeys
```

## osd创建  

创建集群，安装ceph包，收集密钥之后，就可以创建osd了    

#### 配置文件  

修改`ceph-deploy`目录`/data/ceph/deploy`下的`ceph.conf`    

``` shell
#/data/ceph/deploy/ceph.conf添加如下参数  
osd_journal_size = 7168
osd_pool_default_size = 2

osd_pool_default_pg_num = 512
osd_pool_default_pgp_num = 512
rbd_default_features = 3
```

#### 准备osd  

``` shell
ceph-deploy osd prepare node1:vdb:/dev/vdf node1:vdc:/dev/vdf node1:vdd:/dev/vdf node1:vde:/dev/vdf node2:vdb:/dev/vdf node2:vdc:/dev/vdf node2:vdd:/dev/vdf node2:vde:/dev/vdf node3:vdb:/dev/vdf node3:vdc:/dev/vdf node3:vdd:/dev/vdf node3:vde:/dev/vdf
```

每个节点上四个osd磁盘`vd{b,c,d,e}`，使用同一个日志盘`/dev/vdf`，`prepare`过程中ceph会自动在`/dev/vdf`上创建4个日志分区供4个osd使用，日志分区的大小由上步骤`osd_journal_size = 7168`(7G)指定，你应当修改这个值   
prepare 命令只准备 OSD。在大多数操作系统中，硬盘分区创建后，不用 activate 命令也会自动执行 activate 阶段（通过 Ceph 的 udev 规则）  

#### 激活osd  

``` shell
ceph-deploy osd activate node1:vdb1:/dev/vdf1 node1:vdc1:/dev/vdf2 node1:vdd1:/dev/vdf3 node1:vde1:/dev/vdf4 node2:vdb1:/dev/vdf1 node2:vdc1:/dev/vdf2 node2:vdd1:/dev/vdf3 node2:vde1:/dev/vdf4 node3:vdb1:/dev/vdf1 node3:vdc1:/dev/vdf2 node3:vdd1:/dev/vdf3 node3:vde1:/dev/vdf4

#vdf1,vdf2,vdf3,vdf4 为ceph自动创建的四个日志分区  
```

activate 命令会让 OSD 进入 up 且 in 状态，此命令所用路径和 prepare 相同。在一个节点运行多个OSD 守护进程、且多个 OSD 守护进程共享一个日志分区时，你应该考虑整个节点的最小 CRUSH 故障域，因为如果这个 SSD 坏了，所有用其做日志的 OSD 守护进程也会失效    

## ceph-deploy使用  
  
- 允许一主机以管理员权限执行 Ceph 命令  

``` shell
ceph-deploy admin {host-name [host-name]...}
```

- 把改过的配置文件分发给集群内各主机   

``` shell  
ceph-deploy --overwrite-conf config push node{1..3}
```

- ceph-deploy执行后的清除  

``` shell
#卸载指定节点上的ceph软件包  
ceph-deploy uninstall {hostname [hostname] ...}

#清除数据   
#如果只想清除 /var/lib/ceph下的数据、并保留Ceph安装包
ceph-deploy purgedata {hostname} [{hostname} ...]

#要清理掉 /var/lib/ceph 下的所有数据、并卸载 Ceph 软件包
ceph-deploy purge {hostname} [{hostname} ...]
```

- 清除磁盘操作   

``` shell
#查看某节点上所有磁盘
ceph-deploy disk list {node-name [node-name]...}
#清除指定磁盘上的分区，用于重装ceph集群
#ceph-deploy disk zap {osd-server-name}:{disk-name}
ceph-deploy disk zap node1:/dev/vdb
```

- monitor操作  

``` shell
#从某个节点上移除Ceph MON进程
ceph-deploy mon destroy {host-name [host-name]...}
#ceph集群至少需要一个mon进程，但一个mon进程无法保证高可靠性。确保你删除一个监视器后，集群仍能正常工作。
```