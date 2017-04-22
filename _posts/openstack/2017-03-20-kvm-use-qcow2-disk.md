---
title: "openstack实例磁盘扩容"
author: opengers
layout: post
permalink: /virtualization/openstack-instance-disk-resize/
categories: virtualization
tags:
  - openstack
  - kvm
  - resize
format: quote
---

><small>主要总结下openstack实例的磁盘扩容，有三种格式raw/qcow2/rbd块设备</small>    

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

qcow2磁盘格式是qemu模拟器支持的一种磁盘镜像。它可以用一个文件的形式来表示一块固定大小的块设备磁盘。与普通的raw格式相比，有以下特性

* 更小的空间占用，即使文件系统不支持空洞(holes)     
* 支持写时拷贝（COW, copy-on-write，镜像文件只反映底层磁盘的变化   
* 支持快照（snapshot），镜像文件能够包含多个快照的历史   
* 可选择基于 zlib 的压缩方式     

创建qcow2磁盘格式    

``` shell
#qemu-img create -f qcow2 disk1.qcow2 40G
Formatting 'disk1.qcow2', fmt=qcow2 size=42949672960 encryption=off cluster_size=65536 lazy_refcounts=off 

#qemu-img info disk1.qcow2 
image: disk1.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 196K
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false

#ls -lh disk1.qcow2 
-rw-r--r-- 1 root root 193K Apr 22 16:13 disk1.qcow2
```

**raw**   

raw格式应该是最常见的一种磁盘格式，平常所使用的文件都是raw格式    

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

rbd磁盘是ceph提供的块设备，可以在其上安装系统，或者挂载到实例上作为数据卷     

``` shell
#qemu-img create rbd:vms/disk3.rbd 40G
Formatting 'rbd:vms/disk3.rbd', fmt=raw size=42949672960 cluster_size=0 

#qemu-img info rbd:vms/disk3.rbd
image: rbd:vms/disk3.rbd
file format: raw
virtual size: 40G (42949672960 bytes)
disk size: unavailable
```    

