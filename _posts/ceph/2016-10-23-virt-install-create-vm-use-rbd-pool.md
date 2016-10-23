---
title: "virt-install工具安装基于rbd磁盘的虚拟机"
author: opengers
layout: post
permalink: /ceph/virt-install-create-vm-use-rbd-pool/
categories: ceph
tags:
  - ceph-deploy
  - cluster
---

><small>系统centos7.2    
ceph版本 ceph version 10.2.2</small>

测试环境安装了一个ceph集群，准备使用rbd块设备作为虚拟机系统盘安装系统，步骤如下  

在Host上配置好了secret key  

``` shell
virsh secret-list
 UUID                                  Usage
--------------------------------------------------------------------------------
 e63e4b32-280e-4b00-982a-9d3xxxxxxx  ceph client.libvirt secret
```

创建一个新的libvirt pool   

``` shell
virsh pool-dumpxml vms
<pool type='rbd'>
  <name>vms</name>
  <uuid>1ceacccc-97b4-44f4-a7f4-xxxxxxxx</uuid>
  <capacity unit='bytes'>25180075769856</capacity>
  <allocation unit='bytes'>1067251105920</allocation>
  <available unit='bytes'>24416501055488</available>
  <source>
    <host name='172.16.1.10' port='6789'/>
    <host name='172.16.1.11' port='6789'/>
    <host name='172.16.1.12' port='6789'/>
    <name>vms</name>
    <auth type='ceph' username='libvirt'>
      <secret uuid='e63e4b32-280e-4b00-982a-9d3xxxxxxx'/>
    </auth>
  </source>
</pool>
```

使用`virsh vol-create-as`创建了一个20G的磁盘作为虚拟机系统盘     

``` shell
virsh vol-create-as vms virt-1 --capacity 20G --format raw
Vol virt-1 created

virsh vol-list vms
 Name                 Path                                    
------------------------------------------------------------------------------                             
 virt-1               vms/virt-1
```

使用`virt-install`安装虚拟机时，命令如下       

``` shell
virt-install \
--force \
--name virt-install-1 \
--ram 1024 \
--vcpus 1 \
--os-type linux \
--location http://serverip/centos7.2-x86_64 \
--disk vol=vms/virt-1 \
#使用上一步创建的卷virt-1pool
--accelerate \
--clock offset=localtime \
--network bridge=br0,model=virtio \
--network bridge=br1,model=virtio \
--extra-args 'ks=http://serverip/cblr/svc/op/ks/system/centos72.ks ksdevice=eth1 ip=172.16.1.58 netmask=255.255.255.0 dns=223.5.5.5 gateway=172.16.1.1' \
--vnc \
--vnclisten=0.0.0.0 \
--wait 0
```

安装并不成功，报错信息如下    

``` shell
ERROR    internal error: process exited while connecting to monitor: 2016-10-23T11:43:30.639659Z qemu-kvm: -drive file=rbd:vms/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789,if=none,id=drive-ide0-0-0,format=raw: error connecting
2016-10-23T11:43:30.640078Z qemu-kvm: -drive file=rbd:vms/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789,if=none,id=drive-ide0-0-0,format=raw: could not open disk image rbd:vms/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789: Could not open 'rbd:vms/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789': Operation not supported

Domain installation does not appear to have been successful.
```

google了一下，这里也有讨论[virt-install copies part of a pool's definition into a domain's disk definition - should it copy more or less?](https://www.redhat.com/archives/virt-tools-list/2016-January/msg00006.html)    

在ceph开启`cephx`认证的情况下，当前版本的`virt-install`无法把libvirt pool中的认证内容`<auth ... </auth>`传递给生成的虚拟机xml文件，导致虚拟机无法访问rbd磁盘   
