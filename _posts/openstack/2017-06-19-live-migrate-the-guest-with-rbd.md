---
title: "使用rbd磁盘虚拟机的在线迁移"
author: opengers
layout: post
permalink: /openstack/live-migrate-the-guest-with-rbd/
categories: openstack
tags:
  - openstack
  - libvirt
  - rbd
format: quote
---

><small>使用rbd块设备的虚拟机在线迁移比较简单，共享存储下，只迁移配置文件，内存状态等</small>     

Ceph中使用最广泛最成熟的应该是RBD块设备，Ceph提供了librbd库，其它应用比如KVM / libvirt可以通过librbd库远程与rbd交互。如下是一台使用rbd的centos虚拟机 `rbdguest`，我们看下其xml中是如何连接rbd块设备的             

``` html 
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <auth username='libvirt'>
        <secret type='ceph' uuid='e63e4b32-280e-4b00-982a-9d3859cf4699'/>
      </auth>
      <source protocol='rbd' name='vms/rbd_206_96'>
        <host name='172.16.200.76' port='6789'/>
        <host name='172.16.200.77' port='6789'/>
        <host name='172.16.200.78' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
    </disk>
```

可以看到此虚拟机磁盘使用的rbd块设备为`vms/rbd_206_96`，其配置有三台monitor：172.16.200.76-78         
         
我这里有两台centos7宿主机 host-20080 和 host-20081，为了使虚拟机在多台宿主机之间互相迁移，每台宿主机都需要配置libvirtd。作为测试，我这里关闭了tls，如下是以 host-20081 为例     

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

