---
title: "如何在线更改使用rbd磁盘的虚拟机的monitor ip"
author: opengers
layout: post
permalink: /openstack/change-guest-monitor-ip-with-rbd/
categories: openstack
tags:
  - openstack
  - libvirt
  - rbd
format: quote
---

><small>使用rbd块设备的虚拟机其xml文件中配置有monitor ip，本文介绍如何在虚拟机不重启条件下更改其monitor ip</small>     

Ceph中使用最广泛最成熟的应该是RBD块设备，Ceph提供了librbd库，其它应用比如KVM / libvirt可以通过librbd库远程与rbd交互。如下是一台使用rbd的centos虚拟机 `rbdguest`，我们看下其xml中是如何连接rbd块设备的             

``` html 
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <auth username='libvirt'>
        <secret type='ceph' uuid='e63e4b32-280e-4b00-982a-9d3859cf4699'/>
      </auth>
      <source protocol='rbd' name='vms/rbd_206_96'>
     vmdisk   <host name='172.16.200.76' port='6789'/>
        <host name='172.16.200.77' port='6789'/>
        <host name='172.16.200.78' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
    </disk>
```

可以看到此虚拟机磁盘使用的rbd块设备为`vms/rbd_206_96`，其配置有三台monitor：172.16.200.76-78。那么，假如我们的ceph集群添加或删除monitor，monitor节点ip可能需要改变，那怎么更改虚拟机中已经设置的monitor ip呢，虚拟机上可能已运行有业务，不能重启        

如果直接更改xml文件而不重启虚拟机配置是不会生效的。这里有一个思路，libvirt支持在线迁移虚拟机，我们可以尝试先修改xml文件，然后在线迁移此虚拟机。那么迁移后的虚拟机将会使用新的monitor ip信息，而且由于是在线迁移，虚拟机也不用重启      

# 设置虚拟机支持在线迁移         

第一步当然是要保证在线迁移没问题。我这里有两台centos7宿主机 host-20080 和 host-20081，为了使虚拟机在多台宿主机之间互相迁移，每台宿主机都需要配置。作为测试，我这里关闭了tls，如下是以 host-20081 为例 

首先更改libvirtd.conf配置                       

``` shell
[root@host-20081 ~]# cat /etc/libvirt/libvirtd.conf | grep '^[^#]'
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
listen_addr = "0.0.0.0"
auth_tcp = "none"
log_level = 3
log_outputs="3:file:/var/log/libvirtd.log"    
```

然后重启libvirtd      

``` shell
systemctl restart libvirtd   
```

最后，验证在线迁移是否正常。如下命令，把 host-20881 上虚拟机 rbdguest 在线迁移到宿主机 host-20880 上      

``` shell
[root@host-20081 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     rbdguest                       running
 
[root@host-20081 ~]# virsh migrate rbdguest qemu+tcp://172.16.200.80/system --live --persistent --compressed --abort-on-error --undefinesource
#--live 在线迁移
#--undefinesource  迁移成功后，undefine原宿主机上虚拟机    
``` 

之后，在 host-20880 上查看虚拟机 rbdguest 是否迁移成功      

``` shell
[root@host-20080 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 2     rbdguest                       running
 
#登录虚拟机查看信息     
[root@rbdguest ~]# uptime 
 22:34:06 up 53 min,  1 user,  load average: 0.00, 0.01, 0.01 
```

ok，在线迁移没问题，接下来，我们先更改虚拟机xml文件中monitor ip，然后再迁移          

# 更改虚拟机xml中monitor ip        

现在虚拟机rbdguest在宿主机 host-20880 上，首先使用`virsh edit`直接更改虚拟机monitor ip，比如移除第三台monitor：172.16.200.78。 我这里测试的是移除一个monitor ip，当然，更改所有monitor ip也是可以的    

``` shell
[root@host-20080 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     rbdguest                       running

[root@host-20080 ~]# virsh edit rbdguest
#删除掉下面这个 monitor 
#<host name='172.16.200.78' port='6789'/>
```

如下，现在rbdguest只剩下两台monitor    

``` html
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <auth username='libvirt'>
        <secret type='ceph' uuid='e63e4b32-280e-4b00-982a-9d3859cf4699'/>
      </auth>
      <source protocol='rbd' name='vms/rbd_206_96'>
        <host name='172.16.200.76' port='6789'/>
        <host name='172.16.200.77' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
    </disk>
```

然后，在线迁移到 host-20881 上     

``` shell
[root@host-20080 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     rbdguest                       running

[root@host-20080 ~]# virsh migrate rbdguest qemu+tcp://172.16.200.81/system --live --persistent --compressed --abort-on-error --undefinesource 
```

在宿主机 host-20881 上查看虚拟机rbdguest是否迁移成功     

``` shell
[root@host-20081 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 7     rbdguest                       running
```

