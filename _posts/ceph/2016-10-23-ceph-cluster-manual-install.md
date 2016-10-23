---
title: "手工部署一个ceph集群"
author: opengers
layout: post
permalink: /ceph/deploy-a-ceph-cluster-manually/
categories: ceph
tags:
  - ceph deploy
  - manual
---

><small>系统centos7.2    
ceph版本 ceph version 10.2.3   
</small>
 
* TOC
{:toc}  

[手工部署ceph集群](http://www.isjian.com/ceph/deploy-a-ceph-cluster-manually/)  
[ceph-deploy部署ceph集群](http://www.isjian.com/ceph/deploy-a-ceph-cluster-use-ceph-deploy/)   

官方已经有了`ceph-deploy`这个部署工具，它可以方便地部署以及扩展一个ceph集群，参考我之前写的文章[使用ceph-deploy工具部署ceph集群](http://www.isjian.com/ceph/deploy-a-ceph-cluster-use-ceph-deploy/)，虽然如此，一步步手动安装一个ceph集群还是很有必要的，一方面手动安装可以更了解ceph配置过程，便于后期维护排错，另一方面可以为那些使用Chef,juiu,Puppet,...等工具编写部署脚本的开发者提供参考，本文是自己手动安装的步骤，这里记录一下   

## 环境预检    

本文依然使用三台服务器(m1,m2,m3)部署ceph集群，其中三个mon进程(mon.m1,mon.m2,mon.m3)，12个osd进程(osd.0 ~ osd.11)，节点分配如下  

----

节点hostname | 服务 | cluster network | public network
:-- | :-- | :-- | :--
m1(admin-node) | osd.{0,1,2,3},mon.m1  |  eth1:172.31.6.177/24  | eth0:192.168.6.177/22
m2 | osd.{4,5,6,7},mon.m2  |  eth1:172.31.6.178/24  | eth0:192.168.6.178/22
m3 | osd.{8,9,10,11},mon.m3  | eth1:172.31.6.179/24 | eth0:192.168.6.179/22

----

不管是`ceph-deploy`部署还是手动安装，安装前的预检工作是保证安装顺利进行的前提，之前的文章已经详细说过预检项，参考这里[环境预检](http://www.isjian.com/ceph/deploy-a-ceph-cluster-use-ceph-deploy/#section-1)，本文就不再重复  

安装过程中会涉及许多磁盘分区操作，此时`partprobe`命令就很有用了，它可以通知操作系统内核立即更新分区表，以应用我们的分区更改操作。特别是对一个磁盘删除分区后马上新建分区的场景，更需要首先运行`partprobe`让内核更新分区表，然后再重新分区   	

环境预检完成之后，首先需要安装ceph软件包，所有节点都要安装  

**安装ceph包**  

``` shell
yum install ceph
#安装过程会创建空的/etc/ceph文件夹
```

## monitor节点安装  

### monitor节点作用  

部署ceph第一步就是要把集群的Moitors运行起来，Monitors能够控制集群的多项行为，比如每个pool的副本数，每个OSD上的pg数，OSD进程间心跳间隔，是否启用认证(cephx)，...等，这些配置项都有默认值，我们可以根据需要调整优化     

根据文章开始分配，需要在三个节点(m1,m2,m3)上各安装一个mon进程，三节点配置文件是一样的，因此可以在一个节点上配置完成后复制到其它节点上，Monitors需要以下几项配置     

- `fsid` 集群的唯一标识符，`ceph -s`命令显示的第一行  
- `cluster name`可以随意指定，默认是`ceph`，默认的好处是我们在运行命令时不必明确指明集群名，但在有多个ceph集群的情况下，为了在命令行区分不同的集群(`ceph, us-west, us-east` ) ，可以使用集群名命名配置文件，比如`ceph.conf, us-west.conf,us-east.conf`，并且在命令行使用`ceph --cluster {cluster-name}`指定要操作的集群名       
- `Monitor Name` 集群中的每个monitor进程都有唯一的name，一般是主机名，我们这里也是用hostnmae    
- `Monitor Map` 安装Monitors需要生成`monitor map`，`monitor map`需要`fsid`，`cluster name`，和节点的hostname及其ip地址   
- `Monitor Keyring` monitors之间通信需要key认证  
- `Administrator Keyring` 命令行操作进群需要有合适的用户权限，默认使用`client.admin`用户，因此我们需要生成`client.admin`用户和它的key密钥  

下面我们在`m1`节点上操作，三个节点都需要操作的会特别说明  

#### 创建ceph.conf文件  

`ceph.conf`文件需要手动创建，最少需要设置下面三项参数   

``` shell
#uuidgen命令生成一个fsid
676a90f3-bcd1-4c62-9d26-251b17ebcb2a

#cat /etc/ceph/ceph.conf
fsid = 676a90f3-bcd1-4c62-9d26-251b17ebcb2a
mon_initial_members = m1,m2,m3
mon_host = 192.168.6.177,192.168.6.178,192.168.6.179
#这里指定三个monitor节点的ip，monitor对外，因此ip使用public network
```

#### 创建用户及key文件    

创建monitor key用于多个monitor间通信，保存在/tmp/ceph.mon.keyring   

	ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'

生成管理用户client.admin及其key，保存在/etc/ceph/ceph.client.admin.keyring
 
	ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'
	#key文件命令规则:{cluster name}-{user-name}.keyring

添加client.admin的key到ceph.mon.keyring文件    

	ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring

#### 创建monitor map   

上面说过，monitor map需要`fsid`，`cluster name`，和hostname及其ip地址，monitor map定义了集群fsid，monitor的ip及端口   

monmaptool工具可以创建，查看，修改集群的monitor map，相关用法可以查看man手册`man monmaptool`   

``` shell
#默认的monitor port是6789
#使用hostname，host IP，FSID生成一个monitor map，保存为/tmp/monmap，我们这里有三个monitor
monmaptool --create --add m1 192.168.6.177 --add m2 192.168.6.178 --add m3 192.168.6.179 --fsid 676a90f3-bcd1-4c62-9d26-251b17ebcb2a /tmp/monmap

#可以显示monitor map内容
monmaptool --print /tmp/monmap
```

#### 初始化monitor目录    

上面的操作，比如创建用户，key文件，monitor map等都是集群共用的，因此只创建一次就可以了，但是每个节点上的monitor目录名不一样，需要单独创建     

三个节点上分别创建monitor目录，目录名格式为`{cluster-name}-{hostname}`   

``` shell
#mkdir -p /var/lib/ceph/mon/{cluster-name}-{hostname}
m1> mkdir -p /var/lib/ceph/mon/ceph-m1
m2> mkdir -p /var/lib/ceph/mon/ceph-m2
m3> mkdir -p /var/lib/ceph/mon/ceph-m3
```

为了能够初始化m2,m3节点，需要拷贝相关文件到m2,m3主机上   

``` shell
#拷贝以下文件到m2,m3节点   
/tmp/monmap
/tmp/ceph.mon.keyring
/etc/ceph/ceph.conf

#三个节点上相关文件权限设置  
chown ceph:ceph /var/lib/ceph -R
chown ceph:ceph /etc/ceph -R
chown ceph:ceph /tmp/ceph.mon.keyring
chown ceph:ceph /tmp/monmap
```

开始初始化，初始化操作会生成monitor间通信使用的key文件/var/lib/ceph/mon/ceph-{hostname}/keyring文件，其来自上面生成的/tmp/ceph.mon.keyring  

``` shell
#sudo -u ceph ceph-mon --mkfs -i {hostname} --monmap {monmap} --keyring {keyring}
m1> sudo -u ceph ceph-mon --mkfs -i m1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
m2> sudo -u ceph ceph-mon --mkfs -i m2 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
m3> sudo -u ceph ceph-mon --mkfs -i m3 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
#-i m1 中的m1位monitor name
```

#### 获取monmap及key   

初始化完成后，/tmp/monmap，/tmp/ceph.mon.keyring文件可以删除，删除之后还可以使用命令获取  

``` shell
#获取monmap
ceph mon getmap -o monmap
#查看内容
monmaptool --print monmap
#查看mon. keyring文件(注意有个mon后点号)
ceph auth get mon.
```

#### 更新ceph.conf文件   

我们可以随时更新/etc/ceph/ceph.conf文件，添加一些参数       

``` shell
#cat /etc/ceph/ceph.conf
[global]
fsid = 676a90f3-bcd1-4c62-9d26-251b17ebcb2a
mon_initial_members = m1,m2,m3
mon_host = 192.168.6.177,192.168.6.18,192.168.6.179

auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

public_network = 192.168.4.0/22
cluster_network = 172.31.6.0/24

mon_pg_warn_max_per_osd = 1000

osd_journal_size = 7168
#日志分区大小，下面创建日志分区大小会自动读取这里的值
osd_pool_default_size = 2
osd_pool_default min size = 1
osd_pool_default_pg_num = 512
osd_pool_default_pgp_num = 512
rbd_default_features = 3
osd_crush_update_on_start = false
```

#### 启动ceph-mon进程   

三个节点上都要启动mon进程，下面以m1上启动ceph-mon@m1进程为例   

创建done文件，标记mon安装完成

	sudo -u ceph touch /var/lib/ceph/mon/ceph-m1/done

启动ceph-mon进程，启动时候我们需要传递hostname参数给systemctl，用以指明启动的monitor  

``` shell
#格式为ceph-mon@<hostname>.service 或者 ceph-osd@<osd_num>.service
systemctl start ceph-mon@m1
```

monitors运行之后，集群已经有了一个默认的CRUSH map，这个map中没有任何映射，因为还没添加osd，可以使用`ceph -s`查看集群状态，可以看到当前集群处于error状态，有0个osd存在  

## 安装osd节点  

一个节点上四个osd，每个osd使用一块硬盘，你也可以使用一个分区充当osd数据盘，下面我们以在m1节点上安装第一个osd(osd.0)为例说明安装过程，其它osd类似   

#### 添加osd

添加一个新osd，`id`可以省略，ceph会自动使用最小可用整数，第一个osd从0开始   

``` shell
#ceph osd create {id}
ceph osd create
0
```

#### 初始化osd目录  

创建osd.0目录，目录名格式`{cluster-name}-{osd-number}` 

``` shell
#mkdir /var/lib/ceph/osd/{cluster-name}-{osd-number}
mkdir /var/lib/ceph/osd/ceph-0
```

挂载osd.0的数据盘/dev/vdb   

``` shell
mkfs.xfs /dev/vdb
mount /dev/vdb /var/lib/ceph/osd/ceph-0
echo "UUID=61e895a5-03ed-47b0-8665-5245756d5046 /var/lib/ceph/osd/ceph-0   xfs defaults 0 0" >> /etc/fstab
```

初始化osd数据目录   

``` shell
ceph-osd -i 0 --mkfs --mkkey
#--mkkey要求osd数据目录为空
#这会创建osd.0的keyring /var/lib/ceph/osd/ceph-0/keyring
```

初始化后，默认使用普通文件/var/lib/ceph/osd/ceph-0/journal作为osd.0的journal，若只是测试环境，可以不用更改journal分区，跳过更改journal分区这一步骤   

#### 更改journal分区  

我们可以更改journal分区为ssd分区加速性能，这里使用SSD磁盘`/dev/vdf1`分区作为journal  

``` shell
#清楚磁盘其它分区(重新添加时需要)
#sgdisk --zap-all --clear --mbrtogpt /dev/vdf

#生成分区/dev/vdf1的uuid
uuidgen
b3897364-8807-48eb-9905-e2c8400d0cd4
#创建分区
#1:0:+7G 表示创建第一个分区，7G大小
sgdisk --new=1:0:+7G --change-name=1:'ceph journal' --partition-guid=1:b3897364-8807-48eb-9905-e2c8400d0cd4 --typecode=1:b3897364-8807-48eb-9905-e2c8400d0cd4 --mbrtogpt -- /dev/vdf
#格式化
mkfs.xfs /dev/vdf1
rm -f /var/lib/ceph/osd/ceph-0/journal 
ln -s /dev/disk/by-partuuid/b3897364-8807-48eb-9905-e2c8400d0cd4 /var/lib/ceph/osd/ceph-0/journal
chown ceph:ceph -R /var/lib/ceph/osd/ceph-0
chown ceph:ceph /var/lib/ceph/osd/ceph-0/journal
#初始化新的journal
ceph-osd --mkjournal -i 0
chown ceph:ceph /var/lib/ceph/osd/ceph-0/journal
```

#### 注册osd.0

``` shell
ceph auth add osd.0 osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-0/keyring
#ceph auth list 中出现osd.0
```

#### 加入crush map 

这是m1上新创建的第一个osd，CRUSH map中还没有m1节点，因此首先要把m1节点加入CRUSH map，同理，m2/m3节点也需要加入CRUSH map

```shell
#ceph osd crush add-bucket {hostname} host
ceph osd crush add-bucket m1 host
ceph osd crush add-bucket m2 host
ceph osd crush add-bucket m3 host
```

然后把三个节点移动到默认的root `default`下面   

``` shell
ceph osd crush move m1 root=default
ceph osd crush move m2 root=default
ceph osd crush move m3 root=default
```

添加osd.0到CRUSH map中的m1节点下面，加入后，osd.0就能够接收数据   

``` shell
#ceph osd crush add osd.0 0.01900 root=sata rack=sata-rack01 host=sata-node5
ceph osd crush add osd.0 0.01900 root=default host=m1
```

此时osd.0状态是`down`且`in`，`down`是因为我们还没有启动osd.0进程  

#### 启动ceph-osd进程   

需要向systemctl传递osd的id号以启动指定的osd进程，如下，我们准备启动osd.0进程 

``` shell
#systemctl start ceph-osd@{id}
systemctl start ceph-osd@0
systemctl enable ceph-osd@0
```

上面就是添加osd.0的步骤，可以接着在m1节点上添加osd.{1,2,3}，添加了这四个osd后，集群也未能达到OK状态，这是因为这四个osd分布在同一台节点上，若想要快速达到OK状态，可以先在每个节点上都添加一个osd再查看状态      