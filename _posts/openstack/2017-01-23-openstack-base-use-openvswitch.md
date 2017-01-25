---
title: openstack基础-使用openvswitch
author: opengers
layout: post
permalink: /openstack/openstack-base-use-openvswitch/
categories: openstack
tags:
  - virtualization
  - kvm
format: quote
---

## OVS介绍     

在过去，数据中心的服务器是直接连接在硬件交换机上，后来VMware实现了服务器虚拟化技术，使虚拟服务器(VMs)能够连接在虚拟交换机上，借助这个虚拟交换机，可以为服务器上运行的VMs或容器提供逻辑以太网接口，这些逻辑接口都连接到虚拟交换机上，有三种比较流行的虚拟交换机: VMware virtual switch, Cisco Nexus 1000V,和OpenvSwitch(OVS)   

在OVS之前，基于Linux的虚拟化平台比如KVM或Xen上，缺少一个功能丰富的虚拟交换机，因此OVS迅速崛起开始在Xen中流行起来，并且应用于越来越多的开源项目，比如openstack中的网络解决方案  

在虚拟交换机的控制器或管理工具方面，一些商业产品都集成有控制器或管理工具，比如Cisco 1000V的Virtual Supervisor Manager(VSM)，VMware的分布式交换机中的vCenter。而OVS需要借助第三方控制器或管理工具进行管理。例如OVS提供了对OpenFlow 协议的支持，我们就可以使用任何支持OpenFlow协议的控制器对OVS进行远程管理控制。OpenStack Neutron中的ML2插件也能够实现对OVS的管理，但这并不意味着必须要有一个控制器对OVS进行管理，在一些简单场景下，可以单纯把OVS作为一个基于MAC地址学习实现转发功能的二层交换机，就像普通物理交换机那样    

在基于Linux系统上，应用最广泛的还是系统自带的虚拟交换机Linux Bridge，它是一个单纯的基于MAC地址学习的二层交换机，简单高效，但同时缺乏一些高级特性，比如VLAN tag,QOS,流表之类，而且在隧道协议支持方面，Linux Bridge只支持vxlan，OVS是gre,vxlan,IPsec都支持，这也决定了OVS更适用于实现SDN技术   

OVS有以下特点   

- 精细的ACL和QoS策略
- 可以使用OpenFlow和OVSDB协议进行控制  
- Port绑定，隧道协议gre,vxlan,IPsec支持
- 适用于Xen，KVM，VirtualBox等虚拟化平台
- 支持标准的802.1Q VLAN model
- 支持组播功能
- ...

文章使用环境`centos7，openvswitch 2.5，OpenFlow versions 0x1:0x4`   

## OVS架构    

先看下OVS整体架构，主要组件       

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-arch.png)   

用户空间主要组件有数据库服务ovsdb-server，守护进程ovs-vswitchd，以及kernel中的datapath。最上面的Controller表示OVS的控制器，控制器与OVS是通过IP协议进行连接，不一定位于同一台主机上，下面分别介绍下各组件   

**ovsdb-server**    

`ovsdb-server`是实现OVS的数据库服务进程,存放OVS所有配置信息，像网桥名,Port,interface,tunnel等这些，OVS主进程`ovs-vswitchd`需要连接此数据库         

``` shell 
root     22166 22165  0 Jan17 ?        00:02:32 ovsdb-server /etc/openvswitch/conf.db -vconsole:emer -vsyslog:err -vfile:info --remote=punix:/var/run/openvswitch/db.sock --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --no-chdir --log-file=/var/log/openvswitch/ovsdb-server.log --pidfile=/var/run/openvswitch/ovsdb-server.pid --detach --monitor
```

`/etc/openvswitch/conf.db`是数据库文件存放位置，这种文件形式存储，保证了服务器重启不会影响其配置信息，可以使用`ovsdb-tool create`命令创建并初始化此数据库文件    
`--remote=punix:/var/run/openvswitch/db.sock` 实现了一个Unix sockets连接方式，`ovs-vswitchd`或其它工具通过socket连接管理此db    
`/var/log/openvswitch/ovsdb-server.log`是日志记录      

**ovs-vswitchd**   

`ovs-vswitchd`是OVS主进程，通过与ovsdb(`ovsdb-server`)交互实现OVS的所有功能特性，可以查看其进程信息          

``` shell
# ps -ef |grep ovs-vs
root     22176 22175  0 Jan17 ?        00:16:56 ovs-vswitchd unix:/var/run/openvswitch/db.sock -vconsole:emer -vsyslog:err -vfile:info --mlockall --no-chdir --log-file=/var/log/openvswitch/ovs-vswitchd.log --pidfile=/var/run/openvswitch/ovs-vswitchd.pid --detach --monitor
...
```   

`ovs-vswitchd`管理此主机上所有OVS实现的虚拟网桥，其通过socket`/var/run/openvswitch/db.sock`连接ovsdb，并且有自己的日志文件`/var/log/openvswitch/ovs-vswitchd.log`    
`ovs-vswitchd`在启动时会读取ovsdb中配置信息，然后配置`datapaths`和所有的虚拟网桥，当ovsdb中的配置信息改变时(例如使用ovs-vsctl工具)，`ovs-vswitchd`也会自动更新其配置以保持与数据库同步      

`ovs-vswitchd`在运行时需要加载datapath内核模块，datapath内核模块信息如下        

``` shell
# modinfo openvswitch
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/net/openvswitch/openvswitch.ko
license:        GPL
description:    Open vSwitch switching datapath
rhelversion:    7.2
srcversion:     F75F2B83324DCC665887FD5
depends:        libcrc32c
intree:         Y
...
```







更详细的图   

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-details.png)  

OVS中的


















https://www.sdxcentral.com/cloud/open-source/definitions/what-is-open-vswitch/
http://openvswitch.org/features/
