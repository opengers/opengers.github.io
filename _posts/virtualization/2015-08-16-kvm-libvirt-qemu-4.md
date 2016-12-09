---
title: "kvm libvirt qemu实践系列(四)-kvm虚拟机在线调整配置"
author: opengers
layout: post
permalink: /virtualization/kvm-libvirt-qemu-4/
categories: virtualization
tags:
  - virtualization
  - kvm
  - libvirt
format: quote
---

>- centos5.x版本不支持动态调整内存，CPU，以下是在centos6.x上测试
- 在线调整配置应该保证虚拟机重启之后依然生效，即调整的同时需要更改xml文件
- centos6.x平台下，cpu core数只能在线增加，不能在线减小

# 查看虚拟机信息   

``` shell
    shell>  virsh dumpxml cos_v1 | head -n 10
    <domain type='kvm' id='9' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
      <name>cos_v1</name>
      <uuid>d39efd06-6629-aa4a-7241-b36400eade2d</uuid>
      <memory unit='KiB'>4194304</memory>
      <currentMemory unit='KiB'>2097152</currentMemory>
      <vcpu placement='static' current='2'>8</vcpu>            
      <os>
        <type arch='x86_64' machine='rhel6.5.0'>hvm</type>
        <boot dev='hd'/>
        <boot dev='cdrom'/>
```
根据上面配置，首先需要明确

``` shell
<memory unit='KiB'>4194304</memory>
<currentMemory unit='KiB'>2097152</currentMemory>
#为此虚拟机最大分配4G内存,目前使用2G，因此下文内存在线调整区间`1G-4G`

<vcpu placement='static' current='2'>8</vcpu>
#为虚拟机分配最多8个core,目前使用2个core，因此下文cpu在线增加最多到8个core
```

# 内存，cpu调整

### 在线调整虚拟机内存

``` shell
#调整为4G
virsh setmem cos_v1 4G

#减小为2G
virsh setmem cos_v1 2G

#调整的同时更改xml文件
virsh setmem cos_v1 2G --config --live

#使用virsh setmem --help 查看在线调整的同时更改xml文件，以保证调整永久生效
```
能够在线调整的最大内存不能超过为虚拟机分配的最大内存（上面xml文件中设置`<memory unit='KiB'>4194304</memory>`最大为4G）调整范围`1G-4G`

### 在线调整虚拟机CPU(只能增大，不能减小)

``` shell
virsh setvcpus cos_v1 4
virsh setvcpus cos_v1 8
```
同样，能够动态调整的最大VCPU个数也不能超过为虚拟机设置的最大VCPU数量


# 在线添加,移除硬盘

### 添加qcow2格式硬盘

``` shell
#创建qcow2格式的新磁盘,大小为40G
qemu-img create -f qcow2 /data/vhosts/test/cos_v1-add1.disk 40G
virsh attach-disk cos_v1 /data/vhosts/test/cos_v1-add1.disk vdb --cache none --subdriver qcow2 --config --persistent
#虚拟机根磁盘为vda，因此这里使用vdb表示新添加磁盘
#--config 参数同时更新虚拟机xml文件，确保重启后依然生效
```

### 添加raw格式硬盘
``` shell
#创建raw格式的新磁盘,大小为40G
qemu-img create -f raw /data/vhosts/test/cos_v1-add2.disk 40G
virsh attach-disk lnmptest-107 /data/vhosts/test/cos_v1-add2.disk vdc --cache none --subdriver raw --config --persistent
```

### 在线移除硬盘

``` shell
#可以查看虚拟机所有磁盘
virsh domblklist cos_v1
virsh detach-disk cos_v1 vdb
```

# 网卡，CD-ROM添加

### 在线添加网卡

``` shell
virsh attach-interface --domain cos_v1 --type network --source default --model virtio --config
#可以查看新添加的网卡
virsh domiflist cos_v1
```

### 在线添加光盘

``` shell
virsh attach-disk centosbase /data_lij/iso/CentOS-6.4-x86_64-bin-DVD1.iso vdd
```