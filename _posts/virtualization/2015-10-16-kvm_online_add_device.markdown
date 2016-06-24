---
author: lijian
comments: true
date: 2015-10-16 02:06:15+00:00
layout: post
link: http://blog.isjian.com/2015/10/kvm_online_add_device/
slug: kvm_online_add_device
title: kvm虚拟机在线调整硬件配置
wordpress_id: 420
categories:
- 虚拟化
---

<blockquote>#centos5.x版本不支持动态调整内存，CPU，以下是在centos6.x上测试</blockquote>




#### 1.查看虚拟机信息


<!-- more -->

    
    shell>  virsh dumpxml cos_v1 | head -n 10
    <domain type='kvm' id='9' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
      <name>cos_v1</name>
      <uuid>d39efd06-6629-aa4a-7241-b36400eade2d</uuid>
      <memory unit='KiB'>4194304</memory>                     --最大分配内存为4G,目前使用2G
      <currentMemory unit='KiB'>2097152</currentMemory>
      <vcpu placement='static' current='2'>8</vcpu>           --虚拟机分配最大VCPU是8个,目前使用2个 
      <os>
        <type arch='x86_64' machine='rhel6.5.0'>hvm</type>
        <boot dev='hd'/>
        <boot dev='cdrom'/>




#### 2.在线调整虚拟机内存(增大或减小)



    
    #调整为4G
    virsh setmem cos_v1 4G
    
    #调整为2G
    virsh setmem cos_v1 2G


_#能够在线调整的最大内存不能超过为虚拟机分配的最大内存（上面xml文件中设置的最大为4G），否则需要关闭虚拟机上调最大内存_


#### 3.在线调整虚拟机CPU(只能增大，不能减小)



    
    virsh setvcpus cos_v1 4
    virsh setvcpus cos_v1 8


_#同样，能够动态调整的最大VCPU个数也不能超过为虚拟机设置的最大VCPU数量_


#### **4.在线添加硬盘**


#添加qcow2格式硬盘

    
    #创建qcow2格式的新磁盘,大小为40G
    qemu-img create -f qcow2 /data/vhosts/test/cos_v1-add1.disk 40G
    virsh attach-disk cos_v1 /data/vhosts/test/cos_v1-add1.disk vdb --cache none --subdriver qcow2 --config --persistent
    #虚拟机根磁盘为vda，因此这里使用vdb表示新添加磁盘
    #--config 参数同时更新虚拟机xml文件，确保重启后依然生效




#添加raw格式硬盘






    
    #创建raw格式的新磁盘,大小为40G
    qemu-img create -f raw /data/vhosts/test/cos_v1-add2.disk 40G
    virsh attach-disk lnmptest-107 /data/vhosts/test/cos_v1-add2.disk vdc --cache none --subdriver raw --config --persistent







#### **5.在线移除硬盘**



    
    #可以查看虚拟机所有磁盘
    virsh domblklist cos_v1
    virsh detach-disk cos_v1 vdb




#### **6.在线添加网卡**



    
    virsh attach-interface --domain cos_v1 --type network --source default --model virtio --config
    #可以查看新添加的网卡
    virsh domiflist cos_v1




#### **7.在线添加光盘**



    
    virsh attach-disk centosbase /data_lij/iso/CentOS-6.4-x86_64-bin-DVD1.iso vdd
