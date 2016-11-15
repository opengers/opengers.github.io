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

# 使用virt-install安装虚拟机    

首先需要在ceph中为创建一个用户，用于libvirtd程序访问ceph，这里创建`libvirt`这个用户，过程省略  

在Host上配置一个secret key关联`libvirt`用户，详细过程参考ceph官方文档[USING LIBVIRT WITH CEPH RBD](http://docs.ceph.com/docs/master/rbd/libvirt/)   

``` shell
virsh secret-list
 UUID                                  Usage
--------------------------------------------------------------------------------
 e63e4b32-280e-4b00-982a-9d3xxxxxxx  ceph client.libvirt secret
 
#可以使用virsh secret-get-value e63e4b32-280e-4b00-982a-9d3xxxxxxx 得到libvirt用户认证密钥  
```

libvirt支持创建存储池，我们这里创建一个存储池`libvirtpool`，连接ceph中的pool `vms`     

``` shell
virsh pool-dumpxml libvirtpool
<pool type='rbd'>
  <name>libvirtpool</name>     #libvirt pool 名称
  <uuid>1ceacccc-97b4-44f4-a7f4-xxxxxxxx</uuid>
  <capacity unit='bytes'>25180075769856</capacity>
  <allocation unit='bytes'>1067251105920</allocation>
  <available unit='bytes'>24416501055488</available>
  <source>
    <host name='172.16.1.10' port='6789'/>  #ceph monitor信息
    <host name='172.16.1.11' port='6789'/>
    <host name='172.16.1.12' port='6789'/>
    <name>vms</name>     #这里的name为ceph中的某个pool
    <auth type='ceph' username='libvirt'>     #创建的ceph认证用户libvirt
      <secret uuid='e63e4b32-280e-4b00-982a-9d3xxxxxxx'/>   #此uuid关联到libvirt的密钥 
    </auth>
  </source>
</pool>
```

上面创建了存储池`libvirtpool`，现在我们在池中创建一个20G的卷(vol)作为虚拟机系统盘，为后面的`virt-install`命令使用           

``` shell
virsh vol-create-as libvirtpool virt-1 --capacity 20G --format raw
Vol virt-1 created
#在ceph集群中创建一个rbd磁盘virt-1，位于vms池中

#查看libvirtpool中所有卷
virsh vol-list libvirtpool
 Name                 Path                                    
------------------------------------------------------------------------------                             
 virt-1               libvirtpool/virt-1
```

使用`virt-install`安装虚拟机系统，命令如下       

``` shell
virt-install \
--force \
--name virt-install-1 \
--ram 1024 \
--vcpus 1 \
--os-type linux \
--location http://serverip/centos7.2-x86_64 \
--disk vol=libvirtpool/virt-1 \
--accelerate \
--clock offset=localtime \
--network bridge=br0,model=virtio \
--network bridge=br1,model=virtio \
--extra-args 'ks=http://serverip/cblr/svc/op/ks/system/centos72.ks ksdevice=eth1 ip=172.16.1.58 netmask=255.255.255.0 dns=223.5.5.5 gateway=172.16.1.1' \
--vnc \
--vnclisten=0.0.0.0 \
--wait 0
#注意--disk中使用的卷为libvirtpool/virt-1
```

安装并不成功，报错信息如下    

``` shell
ERROR    internal error: process exited while connecting to monitor: 2016-10-23T11:43:30.639659Z qemu-kvm: -drive file=rbd:libvirtpool/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789,if=none,id=drive-ide0-0-0,format=raw: error connecting
2016-10-23T11:43:30.640078Z qemu-kvm: -drive file=rbd:libvirtpool/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789,if=none,id=drive-ide0-0-0,format=raw: could not open disk image rbd:libvirtpool/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789: Could not open 'rbd:libvirtpool/virt-1:auth_supported=none:mon_host=172.16.1.10\:6789': Operation not supported

Domain installation does not appear to have been successful.
```

google了一下，这里也有讨论[virt-install copies part of a pool's definition into a domain's disk definition - should it copy more or less?](https://www.redhat.com/archives/virt-tools-list/2016-January/msg00006.html)，问题在于在ceph开启`cephx`认证的情况下，当前版本的`virt-install`无法把libvirt pool中的认证内容`<auth ... </auth>`传递给生成的虚拟机xml文件，导致虚拟机无法访问rbd磁盘     

我们先来输出虚拟机xml文件看看，并不真正安装。要输出xml文件，只需要添加`--print-xml`参数，它告诉`virt-install`根据我们传递的配置参数输出最终的xml文件    

``` shell
virt-install \
--print-xml \
--force \
...
```

输出xml如下    

``` shell
...
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='network' device='disk'>
      <driver name='qemu' cache='none' io='native'/>
      <source protocol='rbd' name='vms/virt-1'>
        <host name='172.16.200.104' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
    </disk>
...
```   

可以看到，输出的xml文件中丢失了`cephx`认证部分的内容，没了认证，当然无法访问rbd磁盘，既然是缺失认证，那自然就想到可以修改`virt-install`程序中生成虚拟机xml文件部分代码，强制传递缺失的认证内容进去    

# 添加cephx认证部分   

`virt-install`有一个`--debug`参数可以帮我们快速定位问题，通过调试，可以发现生成xml文件的代码位于`/usr/share/virt-manager/virtinst/guest.py`，我们需要修改此文件    

``` shell
#vim /usr/share/virt-manager/virtinst/guest.py

#首先需要定义两个变量，传递ceph认证内容, 下面的uuid即为上面virsh secret-list 的输出
auth_secret = '''
      <auth username='libvirt'>
        <secret type='ceph' uuid='e63e4b32-280e-4b00-982a-9d3xxxxxxx'/>
      </auth>
'''
ceph_monitors = '''
        <host name='172.16.200.104' port='6789'/>
        <host name='172.16.200.105' port='6789'/>
        <host name='172.16.200.106' port='6789'/>
'''

#更改_build_xml函数
#此函数中start_xml变量表示首次安装虚拟机所使用的xml内容，其中包含用于安装系统的引导参数，只使用一次  
#final_xml变量表示虚拟机安装后所使用的xml文件
    def _build_xml(self, is_initial):
        log_label = is_initial and "install" or "continue"
        disk_boot = not is_initial

        start_xml = self._get_install_xml(install=True, disk_boot=disk_boot)
        final_xml = self._get_install_xml(install=False)

#添加------------start
        rgx_qemu = re.compile('(<driver name="qemu"[^>]*?>)')
        rgx_auth = re.compile('(?<=<source protocol="rbd" name=")([^>]*?">).*?(?= *?</source>)',re.S)

        start_xml = rgx_qemu.sub('\\1' + auth_secret,start_xml)
        start_xml = rgx_auth.sub('\\1' + ceph_monitors,start_xml)

        final_xml = rgx_qemu.sub('\\1' + auth_secret,final_xml)
        final_xml = rgx_auth.sub('\\1' + ceph_monitors,final_xml)
#添加------------end

        logging.debug("Generated %s XML: %s",
                      log_label,
                      (start_xml and ("\n" + start_xml) or "None required"))
        logging.debug("Generated boot XML: \n%s", final_xml)
        
        return start_xml, final_xml
#上面添加部分作用是修改start_xml,final_xml这两个变量，在其中加入ceph认证部分的内容(上面定义的两个变量)
```

传递认证后的xml文件示例如下     

``` shell
...
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='network' device='disk'>
      <driver name='qemu' cache='none' io='native'/>
      <auth username='libvirt'>
        <secret type='ceph' uuid='e63e4b32-280e-4b00-982a-9d3xxxxxxx'/>
      </auth>
      <source protocol='rbd' name='vms/virt-1'>
        <host name='172.16.200.104' port='6789'/>
        <host name='172.16.200.105' port='6789'/>
        <host name='172.16.200.106' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
    </disk>
...
```

如上，修改之后，再次运行安装命令，即可正常安装虚拟机   