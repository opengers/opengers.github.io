---
title: "ceph rbd块设备使用场景总结"                        
author: opengers
layout: post
permalink: /ceph/rbd-device-usage-scenarios/
categories: openstack
categories: ceph
tags:
  - ceph
  - rbd
---   

* TOC
{:toc}    

rbd是ceph提供的块设备，本文会总结下rbd块设备的使用场景，开始之前，我们需要理解什么是块设备，块设备是可以对其寻址的一种设备，比如硬盘，CD，RBD设备等，你可以告诉一个块设备"给我block为1000的数据，然后再给我block为1200的数据"。作为对比，对于一个文件系统(nfs，gluster，cephfs，...)，你只能说“读取/etc/1.txt”，文件系统可以直接mount使用，而一个块设备需要先格式化才能使用。   

如何搭建ceph集群就略过了，现在我们有了一个OK的ceph集群，先创建一个`test1`的rbd块               

``` shell
[root@ceph-1 ~]# rbd create -p vms -s 40G test1 
[root@ceph-1 ~]# rbd info vms/test1
rbd image 'test1':
        size 40960 MB in 10240 objects
        order 22 (4096 kB objects)
        block_name_prefix: rbd_data.16f1e72ae8944a
        format: 2
        features: layering
        flags: 
```     

# 直接使用rbd块设备                

在client端，直接使用librbd或      

首先要安装软件包  







