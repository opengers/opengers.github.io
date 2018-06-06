---
title: 用命令实现一个vpc网络(bridge+vxlan)                        
author: opengers
layout: post
permalink: /openstack/create-a-vpc-network-with-commands/
categories: openstack
tags:
  - openstack
  - vpc
  - namespace
  - vxlan
---   

* TOC
{:toc}    

><small>openstack网络中常用的一种vpc实现方式为bridge+vxlan，本文会通过敲命令方式一步一步地模仿实现一个这样的vpc网络(同网段跨节点通信，可以dhcp，绑定浮动IP等)，这样能够更深入理解vpc的实现原理，本文主要基于openstack中的VPC</small>        

VPC(Virtual Private Cloud)，即虚拟私有云，是openstack或各大云平台提供给用户的一个隔离的专属的网络环境，其构建在通用硬件设备之上，具有很大灵活性，允许我们自定义自己的网络拓扑，ip网段，网关，路由规则等。               
        
- 虚拟意思是其用软件实现，所有的VPC共用底层硬件资源池，底层硬件资源由云管理员维护，对用户透明，既然是虚拟的，就是说用户的VPC网络不与底层某几台特定硬件设备绑定，底层硬件设备可能经常新旧更替，但用户无感知                 

- 私有意思是不同的VPC之间逻辑隔离，A用户创建的VPC跟平台上B用户创建的VPC之间逻辑隔离，无法通信，逻辑隔离的主流实现方式是基于隧道技术，每个VPC都有一个独立的隧道id(tunnel id)，一个VPC内的虚拟机之间的通信数据包都会经过隧道封装(带有tunnel id)，然后送到物理网络上进行传输。逻辑隔离而不是物理隔离，也就是说A用户的虚拟机可能与B用户的虚拟机位于底层同一物理机上，但A，B用户因为其VPC的隧道id不同(位于两个平行的路由平面)，所以无法进行通信。这种逻辑隔离的好处就是可以通过软件控制，比硬件隔离灵活很多。当然，上面说的VPC间隔离是指二层隔离，三层隔离需要靠安全组实现。那如果需要不同VPC实现二层互联也是可以的，比如可以通过IPsec-VPN，专线接入等方式                                
- 既然称为虚拟私有云，而不是虚拟路由器或虚拟交换机，也就意味着在VPC内你可以通过软件定义网络的方式创建虚拟路由器，虚拟交换机，定义IP网段，路由表，网关等，当然也可以利用路由器实现不同IP网段互通，随你定义。         

上面简单介绍了下VPC，接下来我们仿照openstack Neutron中bridge+vxlan实现VPC的方式，来用命令实现一个类似的VPC网络             

# 准备硬件环境               

第一步当然是准备我们的硬件环境，这里需要三台节点(物理机或虚拟机，如果用虚拟机，需要注意虚拟机嵌套)，三台节点要求如下         

| 网卡 | 网络节点ip(ctl) | 计算节点node-1 ip | 计算节点node-2 ip | 用途 |
| --------- | -------- | -------- | -------- | -------- |
| eth0 | 172.16.207.228  |  172.16.207.229 | 172.16.207.230 | 管理网络，openstac中用于API/MQ/db通信连接，此测试环境中仅仅作为ssh连接使用 |
| eth1 | 10.1.1.228 | 10.1.1.229 | 10.1.1.230 | 隧道网络，vpc内虚拟机数据包封装后，通过此网络传输到其它节点上 |
| eth2 | UP即可，不需要配置IP | 不需要此接口 | 不需要此接口 | 外部网络，vpc内虚拟机通过此接口以NAT方式访问外网 |
{:.mbtablestyle}       
<br />

如下，我使用的是在同一物理机上建三台虚拟机做节点           

``` shell
[root@jpcloud inadm]# virsh list --all | grep vpc
 284   vpc-test1                      running
 285   vpc-test2                      running
 286   vpc-test3                      running

[root@jpcloud inadm]# virsh domiflist vpc-test1
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet1      bridge     br1        virtio      52:54:00:5e:28:ec
vnet2      bridge     br-tun     virtio      52:54:00:a5:fd:7d
vnet3      bridge     br1        virtio      52:54:00:9b:b3:a5

[root@jpcloud inadm]# virsh domiflist vpc-test2
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet4      bridge     br1        virtio      52:54:00:d4:a5:d4
vnet5      bridge     br-tun     virtio      52:54:00:d5:1f:6a

[root@jpcloud inadm]# virsh domiflist vpc-test3
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet6      bridge     br1        virtio      52:54:00:31:46:69
vnet7      bridge     br-tun     virtio      52:54:00:51:8f:b9

[root@jpcloud inadm]# brctl show
bridge name     bridge id               STP enabled     interfaces
br1             8000.f8bc1212cbe1       no              em2
                                                        vnet1
                                                        vnet3
                                                        vnet4
                                                        vnet6

br-tun          8000.f8bc1212cbe2       no              vnet2
                                                        vnet5
                                                        vnet7
```

- 本环境中不部署openstack，因此管理网络接口eth0只是用作ssh到机器上去           
- eth1用作隧道传输数据使用，不需要访问外网，只需要保证三台节点ip互通即可，我这里是建了个bridge `br-tun`桥接了三台机器。eth1不必须，可以与eth0共用    
- vpc内虚拟机访问外网需要通过iptables NAT转换，eth2用作外部接口，我这里和eth0都放在br1网桥上                          
- node-1和node-2是两台用于创建虚拟机的计算节点，后面我们会看到同一VPC内同一IP网段位于不同节点上的虚拟机是如何通过隧道进行通信           
- 其实三台节点都只需要一块网卡eth0就可以了，只是这种把所有通信流量混在一起的方式不利于理解各个网络的实际作用，而且真正生产环境也不会只用一块网卡这种做法            

# 新建一个vpc     

``` shell
#node-1 && node-2
[root@node-1 ~]# ip link add vxlan-200 type vxlan id 200 dev eth1 dstport 8472
[root@node-1 ~]# brctl addbr brq-200
[root@node-1 ~]# brctl addif brq-200 vxlan-200
[root@node-1 ~]# ip link set brq-200 up

#新建虚拟机vm1
    <interface type='bridge'>
      <mac address='52:54:00:98:8c:40'/>
      <source bridge='brq-200'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
[root@node-1 ~]# virsh start vm1
[root@node-1 ~]# virsh domiflist vm1
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     brq-200    virtio      52:54:00:98:8c:40

[root@node-2 ~]# virsh domiflist vm2
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     brq-200    virtio      52:54:00:35:ac:92    

```





