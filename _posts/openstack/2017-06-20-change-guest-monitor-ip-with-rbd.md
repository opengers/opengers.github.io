---
title: "如何更改使用rbd磁盘的虚拟机的monitor ip"
author: opengers
layout: post
permalink: /openstack/how-to-change-guest-monitor-ip-with-rbd-disk/
categories: openstack
tags:
  - openstack
  - libvirt
  - rbd
format: quote
---

><small>使用rbd块设备的虚拟机其xml文件中配置有monitor ip，本文介绍如何在虚拟机不重启条件下处理其monitor ip</small>     

* TOC
{:toc}

Ceph中使用最广泛最成熟的应该是RBD块设备，Ceph提供了librbd库，其它应用比如KVM / libvirt可以通过librbd库远程与rbd交互。如下是一台使用rbd的centos虚拟机 `rbdguest`，我们看下其xml中是如何连接rbd块设备的            

``` html 
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='network' device='disk'>
      <driver name='qemu' cache='none' io='native'/>
      <auth username='libvirt'>
        <secret type='ceph' uuid='e63e4b32-280e-4b00-982a-9d3859cf46f0'/>
      </auth>
      <source protocol='rbd' name='vms/centos7-20689'>
        <host name='172.16.200.117' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </disk>
```

可以看到此虚拟机磁盘连接的rbd块设备为`vms/centos7-20689`，其配置有一台monitor 172.16.200.117，monitor端口为默认的6789。那么，假如我们的ceph集群添加或删除monitor或者某台monitor挂掉，总之就是ceph集群的`monmap`地址需要改变，那怎么处理虚拟机中已经设置的monitor ip呢，虚拟机上可能已运行有业务，不能重启               

# 虚拟机是如何连接monitor的     

我们先来做个测试    

如下，这里有一个三节点的ceph集群，节点hostname为：`host-200116, host-200117, host-200118`，在节点`host-200116` 上有一台测试虚拟机`rbdguest`。虚拟机xml中配置了一个monitor：172.16.200.117       
  
``` shell
#查看集群monmap
[root@host-200116 ~]# ceph mon dump
dumped monmap epoch 7
epoch 7
fsid 61b0877f-695e-4405-884e-d639dba405bb
last_changed 2016-12-22 13:05:51.573448
created 2016-10-19 17:20:05.244477
0: 172.16.200.116:6789/0 mon.host-200116
1: 172.16.200.117:6789/0 mon.host-200117
2: 172.16.200.118:6789/0 mon.host-200118

[root@host-200116 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     rbdguest                       running

#xml中配置的monitor 
[root@host-200116 ~]# virsh dumpxml rbdguest |grep "port='6789'"
    <host name='172.16.200.117' port='6789'/>
```

我们知道kvm虚拟机实际上只是宿主机上的一个`qemu-kvm`进程，我们在宿主机`host-200116`上查看此虚拟机的`qemu-kvm`进程与哪台monitor建立有连接            

``` shell
#获取虚拟机进程 PID
[root@host-200116 ~]# ps -ef |grep rbdguest
qemu      585898       1 56 13:09 ?        00:00:13 /usr/libexec/qemu-kvm -name rbdguest -S -machine ... -drive file=rbd:vms/centos7-20689:id=libvirt:key=AQDDoQdfftbKMhAAIp8Dsi63C5soe+ysHGCqff==:auth_supported=cephx\;none:mon_host=172.16.200.117\:6789,if=none,id=drive-virtio-disk0,format=raw,cache=none,aio=native -device virtio-blk-pci,scsi=off,bus= ...     
#从qemu-kvm进程也可以看到，虚拟机当前设置的monitor为 mon_host=172.16.200.117\:6789       

#查看 PID 585898 当前连接的哪个monitor(monitor默认端口号为6789)       
[root@host-200116 ~]# netstat -tpn |grep 585898 | grep 6789
tcp        0      0 172.16.200.116:11026    172.16.200.117:6789     ESTABLISHED 585898/qemu-kvm
```

上面可以看到，虚拟机当前与monitor `172.16.200.117:6789` 建立有TCP连接       

然后，我们关闭节点`host-200117`上的mon进程，再查看虚拟机所连接monitor是否有变化     

``` shell
#关闭host-200117上的 monitor
[root@host-200117 ~]# systemctl start ceph-mon.target
[root@host-200117 ~]# ps -ef |grep ceph-mon
root     3705983 3689311  0 13:10 pts/3    00:00:00 grep --color=auto ceph-mon
#mon进程已经关闭

#返回节点host-200116上，查看当前虚拟机连接的monitor    
[root@host-200116 ~]# netstat -tpn |grep 585898 | grep 6789
tcp        0      0 172.16.200.116:27739    172.16.200.116:6789     ESTABLISHED 585898/qemu-kvm
```

当前虚拟机连接的monitor变为`172.16.200.116:6789`        

现在问题是，我们虚拟机xml中配置的monitor是172.16.200.117，之前查看`qemu-kvm`进程其设置的也只有`mon_host=172.16.200.117\:6789`，那当monitor 172.16.200.117挂掉之后，虚拟机是如何切换到 monitor 172.16.200.116上的呢          

# 连接monitor原理分析         

原因在于虚拟机进程`qemu-kvm`选择连接哪个monitor是依靠其获取的`monmap`来决定的，而不是仅仅连接其xml中配置的那个monitor，我们从头来分析下虚拟机启动连接rbd磁盘流程           

- 新建虚拟机`rbdguest`和其使用的rbd磁盘，这部分就不说了    
- 在xml中配置虚拟机所使用的rbd磁盘并指定集群的monitor ip，就是这一行`<host name='172.16.200.117' port='6789'/>` (当然还有cephx认证这些)     
- virsh启动虚拟机，启动后，`qemu-kvm`进程会根据xml中配置的monitor ip和端口去连接ceph集群，从而获取集群`monmap`,(当前集群有116/117/118这三个monitor)，并保存到本地       
- 获取`monmap`之后，虚拟机就明白我当前共有三个monitor可以连接               
- 然后stop掉`host-200117`上的mon进程后，集群`monmap`发生变化，虚拟机进程`qemu-kvm`会同步更新其保存的`monmap`      
- 之后虚拟机检测到172.16.200.117这一monitor不可用，然后根据更新后的`monmap`切换到另一个可用monitor      

就算集群中所有的monitor ip都改变也是这个情况。比如现在集群中有`A，B，C`三个monitor，虚拟机V xml中配置的这三个monitor并且虚拟机处于running状态。现在我们把三个monitor ip分别改为`A1，B1，C1`(更改步骤一般是先添加一个新的monitor，然后删除旧的monitor)，更改过程中虚拟机V也会同步更新其自身保存的`monmap`。因此不管集群monitor ip如何更改，虚拟机进程`qemu-kvm`始终能获取最新的monitor ip。于是虚拟机就会去连接`A1，B1，C1`，尽管此时虚拟机xml中配置的monitor还是`A, B, C`       

**要明白的是，虚拟机xml中配置的monitor ip或其`qemu-kvm`进程参数中设置的`mon_host=xxx`，其作用仅仅是在虚拟机启动时，告诉虚拟机根据这个ip地址可以获取ceph集群的`monmap`**    

# 总结      

因此，其实根本就不用去更改虚拟机设置的monitor ip，当你更改集群的`monmap`，虚拟机会同步更新其自身保存的`monmap`,从而使用更新后的ip连接monitor，而连接monitor之后，就能获取集群的`osdmap`，从而连接rbd设备    

当然，为了永久生效，还是应该使用`virsh edit`更改下xml中配置的monitor，仅仅是为了下次虚拟机关机再开机之后，能够正常连接ceph获取monmap       
(本文完)    