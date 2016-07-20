---
title: kvm在线添加硬件
author: opengers
layout: post
permalink: /virtualization/kvm-online-add-device/
categories: virtualization
tags:
  - virtualization
  - kvm
  - attach-device
format: quote
---

> centos5.x版本不支持在线调整内存，cpu，以下在centos6.x,7.x测试  
centos6.x, 7.x平台下，cpu core数只能在线增加，不能在线减小  
在线调整配置应该保证虚拟机重启之后依然生效  

## 查看虚拟机信息  
文中使用虚拟机domain为"cos"
``` html
<memory unit='KiB'>4194304</memory>
<currentMemory unit='KiB'>2097152</currentMemory>
```
为此虚拟机最大分配4G内存,目前使用2G，因此下文内存最大可以调整4G，否则需要关闭虚拟机，调整该最大分配值

``` html
<vcpu placement='static' current='2'>8</vcpu>
```
为虚拟机分配最多8个core,目前使用2个core，因此下文cpu在线增加最多到8个core

要使在线添加硬件重启后依然生效，需要在命令上加"–config"，"--live或"--persistent"参数，具体使用virsh setmem --help查看  
以下操作虚拟机处于运行状态

## 内存调整
#临时调整为4G
virsh setmem cos 4G

#调整的同时更改xml文件，永久调整
virsh setmem cos 2G --config --live
#--config: 设置的同时更改虚拟机xml文件，这样就可以保证虚拟机重启后仍然生效
#–live: 在线调整
#其它添加硬件的命令同样可以使用上面两个参数

#减小为2G
virsh setmem cos 2G --config --live

能够在线调整的最大内存不能超过为虚拟机分配的最大内存（上面xml文件中设置`<memory unit='KiB'>4194304</memory>`最大为4G）

virsh setmem domain 4  这样的操作会使虚拟机直接挂掉！
注意调整内存大小需要带单位，默认单位为"K" 

-----------------------------------------------------------------------------------

内存在线调整的原理是利用了virtio-balloon技术，技术可以在客户机运行时动态地调整它所占用的宿主机内存资源，而不需要关闭客户机。该技术能够：

当宿主机内存紧张时，可以请求客户机回收利用已分配给客户机的部分内存，客户机就会释放部分空闲内存。若其内存空间不足，可能还会回收部分使用中的内存，可能会将部分内存换到交换分区中。
当客户机内存不足时，也可以让客户机的内存气球压缩，释放出内存气球中的部分内存，让客户机使用更多的内存
内存调整过程：

KVM 发送请求给 VM 让其归还一定数量的内存给KVM。
VM 的 virtio_balloon 驱动接到该请求。
VM 的驱动是客户机的内存气球膨胀，气球中的内存就不能被客户机使用。
VM 的操作系统归还气球中的内存给VM
KVM 可以将得到的内存分配到任何需要的地方。
KM 也可以将内存返还到客户机中。
更多信息：http://www.cnblogs.com/sammyliu/p/4543657.html

cpu调整(只能增大，不能减小)
#cpu增大为4 core
virsh setvcpus cos 4 --config --live
#增大为8 core
virsh setvcpus cos 8 --config --live

同样，能够动态调整的最大VCPU个数也不能超过为虚拟机设置的最大VCPU数量

添加移除硬盘
#创建新lv磁盘test-data1
lvcreate -L 20G -n test-data1 CentOS_kvm
#添加新lv磁盘
virsh attach-disk cos  /dev/CentOS_kvm/test-data1 vdb --cache none --subdriver raw --persistent
#vdb：新磁盘盘符，若添加第二块磁盘，就是vdc
#–subdriver raw: lv卷也是raw格式，若添加qcow2格式磁盘，要使用–subdriver qcow2 
#--persistent: 重启生效，相当于–config --live

在线移除硬盘

#可以查看虚拟机所有磁盘
virsh domblklist cos
#永久移除vdb磁盘
virsh detach-disk cos vdb --persistent

-----------------------------------------------------------------------------------

attach-disk支持很有限的参数，比如当新添加的磁盘是块设备，就没法设置AIO特性。
attach-device才是更通用的添加硬件方法，添加硬件的同时可以配置相应参数，比如我们需要添加 /dev/CentOS_kvm/test-data1这块新磁盘
首先配置一份磁盘的xml文件，配置需要的参数项，比如"cache=none","io=native"这些，xml如下

[root@centos7 ~]# cat disk.xml
----------------------------------------
<disk type='block' device='disk'>
<driver name='qemu' type='raw' cache='none' io='native'/>
<source dev='/dev/CentOS_kvm/centos02_lij_data1'/>
<target dev='vdb' bus='virtio'/>
</disk>
----------------------------------------
然后使用attach-device命令添加
virsh attach-device centos7.2_6_152_lij disk.xml --persistent
#persistent保证了永久生效 

添加移除网卡
virsh attach-interface --domain cos --type bridge --source br1 --model virtio --persistent
#--type bridge: 虚拟机使用桥接模式
#--source br1 指定使用的网桥

#可以查看新添加的网卡，及其mac地址
virsh domiflist cos
#根据mac地址移除网卡
virsh detach-interface cos bridge 52:54:00:a2:29:2a --persistent 

 ---------------------------------------------------------------------------------------

上面使用attach-interface方式添加网卡，也可以使用"virsh attach-device"配合xml文件添加硬件，比如需要添加新网卡eth2

#复制一份网卡配置的xml文件
cat eth2-nic.xml
----------------------------------------
<interface type='bridge'>
<source bridge='br1'/>
<model type='virtio'/
</interface>
----------------------------------------

#永久生效
virsh attach-device cos eth2-nic.xml --persistent

在线添加光盘
virsh attach-disk cos /data_lij/iso/CentOS-6.4-x86_64-bin-DVD1.iso vdd 

