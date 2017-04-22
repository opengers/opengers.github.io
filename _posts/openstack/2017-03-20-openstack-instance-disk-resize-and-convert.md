---
title: "openstack底层技术-实例磁盘扩容及格式转换"
author: opengers
layout: post
permalink: /virtualization/openstack-instance-disk-resize-and-convert/
categories: virtualization
tags:
  - openstack
  - kvm
  - convert
format: quote
---

><small>主要总结下openstack虚拟机的磁盘扩容以及不同格式(raw/rbd/qcow2/lvm)间的转换</small>    

* TOC
{:toc} 

本文所使用宿主机，及kvm版本如下     

``` shell
#cat /etc/centos-release
CentOS Linux release 7.2.1511 (Core) 

#rpm -qa |grep qemu
qemu-img-1.5.3-105.el7_2.4.x86_64
qemu-kvm-common-1.5.3-105.el7_2.4.x86_64
qemu-kvm-1.5.3-105.el7_2.4.x86_64
libvirt-daemon-driver-qemu-1.2.17-13.el7_2.5.x86_64
```

## 几种磁盘格式介绍      

**qcow2**  

qcow2是qemu模拟器支持的一种磁盘镜像。它可以用一个文件的形式来表示一块固定大小的块设备磁盘。与普通的raw格式相比，有以下特性   

* 更小的空间占用，即使文件系统不支持空洞(holes)      
* 支持写时拷贝（COW, copy-on-write，镜像文件只反映底层磁盘的变化   
* 支持快照（snapshot），镜像文件能够包含多个快照的历史(快照链，backfile)   
* 可选择基于 zlib 的压缩方式     

创建qcow2磁盘格式      

``` shell
#qemu-img create -f qcow2 disk1.qcow2 40G
Formatting 'disk1.qcow2', fmt=qcow2 size=42949672960 encryption=off cluster_size=65536 lazy_refcounts=off 

#qemu-img info disk1.qcow2 
image: disk1.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
...

#ls -lh disk1.qcow2 
-rw-r--r-- 1 root root 193K Apr 22 16:13 disk1.qcow2
```

**raw**   

raw格式是最常见的一种磁盘格式，平常所使用的文件,iso镜像等都是raw格式      

``` shell
#qemu-img info /etc/passwd
image: /etc/passwd
file format: raw
virtual size: 2.0K (2048 bytes)
disk size: 4.0K

#qemu-img create -f raw disk2.raw 40G
Formatting 'disk2.raw', fmt=raw size=42949672960 

#qemu-img info disk2.raw 
image: disk2.raw
file format: raw
virtual size: 40G (42949672960 bytes)
disk size: 0

#ls -lh disk2.raw 
-rw-r--r-- 1 root root 40G Apr 22 16:15 disk2.raw
```

**RBD**    

rbd是ceph提供的块设备，与物理磁盘，LV卷一样，可以在其上安装系统，或者挂载到实例上作为数据盘         

``` shell
#centos7下的qemu-img支持创建rbd格式磁盘    
#qemu-img create rbd:vms/disk3.rbd 40G    
Formatting 'rbd:vms/disk3.rbd', fmt=raw size=42949672960 cluster_size=0 

#qemu-img info rbd:vms/disk3.rbd
image: rbd:vms/disk3.rbd
file format: raw
virtual size: 40G (42949672960 bytes)
disk size: unavailable     
```    

## 关于磁盘设备与文件系统      

扩容前，我们先了解下磁盘设备与文件系统，这是了解磁盘扩容的基础        

假如有一台40G根磁盘的centos6虚拟机需要扩容到80G。首先关机，然后使用`qemu-img resize disk +40G`扩容系统盘到80G，启动虚拟机，使用`fdisk`命令可以看到磁盘大小为80G，然而使用`df -h`查看却仍为40G左右，并没有增大。真正能供我们写数据的大小仍为40G         

文件系统类型很多，ext3/ext4/xfs/btrfs等，文件系统是建立在磁盘分区之上，像`UUID`,`inode`这些是文件系统层级的概念，没有经过分区格式化的磁盘设备是不能使用的，我们在系统中使用`df -h`看到的`/dev/vda1,/dev/vda2`是挂载到系统中的`vda`磁盘设备上所建立的两个分区，这两个分区可能使用了vda整块磁盘容量，也可能只占用了部分容量，因此如果要扩容某台虚拟机，首先要扩容磁盘容量，然后使分区上文件系统识别磁盘大小变化并相应扩容inode数量        

再考虑下我们安装虚拟机的过程，新建一块指定大小的磁盘设备，然后在其上新建分区，安装系统。刚新建磁盘设备时，磁盘上并无任何数据，之后新建分区格式化会向磁盘上写入inode, 一旦分区建立，inode数量就也确定，文件系统就形成了。关于inode，可以参考这里[理解inode](http://www.ruanyifeng.com/blog/2011/12/inode.html)         

## centos6实例下的扩容             

centos6默认磁盘格式为ext4，centos7系列使用的xfs，因此扩容方式也不同      

测试用的虚拟机为centos6.3系统，我们准备把根磁盘扩容到80G。扩容磁盘大小可以使用`qemu-img`工具，扩容文件系统主要是使用growpart等工具在系统启动时检测磁盘大小变化，并自动增大分区文件系统inode数量       

``` shell    
#virsh domblklist centos6
Target     Source
------------------------------------------------
vda        /data/test/centos6.qcow2

#qemu-img info /data/test/centos6.qcow2
image: /data/test/centos6.qcow2
file format: qcow2
virtual size: 40G (85899345920 bytes)
disk size: 1.2G
```

首先关闭虚拟机，resize磁盘到80G      

``` shell
virsh shutdown centos6

qemu-img resize centos6.qcow2 +40G

#rbd磁盘也可以使用rbd工具  
rbd resize vms/disk.rbd -s 400G
```

启动虚拟机，磁盘大小已变为80G，下面扩容其文件系统，需要注意的是，因为根磁盘在虚拟机运行期间处于挂载状态，无法扩容，只能在系统启动期间自动检测并扩容       
  
``` shell
#安装parted growpart
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/cloud-utils-growpart-0.27-10.el6.x86_64.rpm
yum install parted

wget https://github.com/flegmatik/linux-rootfs-resize/archive/master.zip 
unzip master
cd linux-rootfs-resize-master
./install
```

之后重启虚拟机，启动过程中可以看到inode会自动扩容       

![resize](/images/openstack/install-resize/resize1.png)    

## centos7实例下的磁盘扩容        

centos7中默认使用xfs格式，下面以xfs磁盘格式为例。     

同样，首先关机，扩容虚拟机磁盘       

``` shell
qemu-img resize centos7.qcow2 +40G    
```

然后启动虚拟机    

``` shell
#安装扩容工具  
#yum install cloud-utils-growpart

#磁盘已扩为80G
#fdisk -l
Disk /dev/vda: 85.9 GB, 85899345920 bytes, 167772160 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000b314e

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1            2048    33556479    16777216   82  Linux swap / Solaris
/dev/vda2   *    33556480    83886079    25164800   83  Linux

#根分区仍没变  
#df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        24G  1.6G   23G   7% /
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm

#我们把40G加到分区2上     
#growpart /dev/vda 2
CHANGED: partition=2 start=33556480 old: size=50329600 end=83886080 new: size=134210315,end=167766795

#然后扩容分区2文件系统    
[root@localhost ~]# xfs_growfs /
meta-data=/dev/vda2              isize=512    agcount=4, agsize=1572800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=6291200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=3071, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 6291200 to 16776289
```

## 常见虚拟机磁盘格式间的转换   

这里简单记录下不同磁盘格式间的转换，注意centos6的宿主机`qemu-img`命令不支持rbd格式   

``` shell  
#qcow2磁盘压缩  
qemu-img convert -c -O qcow2 centos.raw centos.qcow2

-f: 源文件格式,qemu-img会自动检测,可以省略
-c: 启用压缩,qcow2格式才支持
-O: 目标镜像格式

#lv卷转qcow2格式 
qemu-img convert -O qcow2 /dev/CentOS_kvm/centos6_6_163 centos6_6_163.qcow2

#qcow2格式转换为lv格式
qemu-img convert -p -O raw centos6_6_163.qcow2 /dev/CentOS_kvm/centos6_6_163     

#rbd块转qcow2
qemu-img convert -O qcow2 rbd:vms/centos7.rbd centos7.qcow2

#qcow2转为rbd格式
qemu-img convert disk.qcow2 rbd:vms/disk.rbd
```

## virt-resize

当然，`virt-resize`工具也能够扩容虚拟机磁盘    

现有一个16G的qcow2格式的centos镜像,需要扩容到50G    
* 查看分区

	virt-filesystems --long --parts --blkdevs -h -a /data/images/centos.qcow2

* 创建新镜像,大小为50G，要比旧镜像大    

	qemu-img create -f qcow2 /data/images/centos-50g.qcw2 50G

* 扩容虚拟机sda2分区   

	virt-resize --expand /dev/sda2 /data/images/centos.qcow2 /data/images/centos-50g.qcow2