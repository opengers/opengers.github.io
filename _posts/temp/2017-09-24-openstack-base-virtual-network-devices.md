---
title: openstack底层技术-使用虚拟网络设备
author: opengers
layout: post
permalink: /openstack/openstack-base-virtual-network-devices/
categories: openstack
tags:
  - openstack
  - tun/tap
  - veth
---

* TOC
{:toc}    

><small>IBM网站上有一篇高质量文章[Linux 上的基础网络设备详解](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/)相信很多人都已经看过。本文会参考文章部分内容，主要侧重于介绍OpenStack是如何使用这些虚拟网络设备构建出灵活的云网络，这些网络设备包括Bridge，Vlan，tun/tap, veth，vxlan/gre</small>      

OpenStack一般分为计算，存储，网络三部分。考虑构建一个灵活的可扩展的云网络环境，而物理网络架构一般是固定和难于扩展的，因此虚拟网络设备将更有优势。Linux平台上实现了各种不同功能的虚拟网络设备，包括`Bridge,Vlan,tun/tap,veth pair,vxlan/gre，...`，这些虚拟设备就像一个个积木块一样，被OpenStack组合用于构建虚拟网络。 还有火热的Docker，其技术实现脱胎于Linux平台上的`namspace`,以及更早的`chroot`。    

文中会牵涉虚拟机，所以用名词"主机"明确表示一台物理机，"接口"指挂载到网桥上的网络设备          

``` shell
CentOS Linux release 7.3.1611 (Core) 
Linux controller 3.10.0-514.16.1.el7.x86_64 #1 SMP Wed Apr 12 15:04:24 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
OpenStack社区版 Newton
```   

# Linux Bridge   

内核模块`bridge`     

``` shell
[root@controller ~]# modinfo bridge
filename:       /lib/modules/3.10.0-514.16.1.el7.x86_64/kernel/net/bridge/bridge.ko
```

Bridge是Linux上一款工作在数据链路层的虚拟交换机，它可以绑定若干个网络设备(em1,eth0,tap,..)，这些网络设备被虚拟为接口连接到Bridge上，从接口收到的数据包都被直接送到Bridge中(bridge内核模块)，在Bridge进行一个类似物理交换机的查MAC端口映射表，转发，更新MAC端口映射表这样的处理逻辑。从而数据包可以被转发到另一个接口/丢弃/广播/发往上层协议栈。由此Bridge实现了数据交换的功能。如果使用`tcpdump`在Bridge接口上抓包，是可以抓到桥上所有接口进出的包            

跟物理交换机不同的是，运行Bridge的是一个Linux主机，Linux本身也需要IP地址与外部通信。但被添加到Bridge上的接口设备是不能配置IP地址的，Bridge及其接口设备工作在链路层，对路由系统不可见。不过Bridge本身可以设置IP地址，可以认为当使用`brctl addbr br0`新建一个`br0`网桥时，系统自动创建了一个同名的隐藏`br0`网络设备，因此我们给`br0`设置IP地址，Bridge收到目的地址为此IP的数据包会直接将数据包发往主机协议栈上层，也即主机收到了发往自身的数据包。             

![bridge](/images/openstack/openstack-virtual-devices/bridge.png)  

当Bridge从某个接口(vnet0)收到数据包后，对数据包的处理有如下几种，也就是上图中的`bridging decision`位置：   

- 包目的MAC为Bridge本身MAC地址，就是收到发往主机自身的数据包，交给上层协议栈(`br0 本机`)，当然无需转发给其它接口     
- 广播包，转发到Bridge上的所有接口      
- 单播&&存在于MAC端口映射表，查表直接转发到对应接口(vnet1)          
- 单播&&不存在于MAC端口映射表，泛洪到Bridge连接的所有接口           
- 数据包目的地址接口不是网桥接口，桥不处理，交给上层协议栈           

*Bridge设置防火墙*        

Linux上防火墙是通过`netfiler`这个内核框架实现，用户层工具则是iptables/ebtables等。iptables工作在IP层，只能过滤IP数据包；ebtables则工作在数据链路层，只能过滤以太网帧(比如匹配源目的MAC地址)。Bridge的出现使Linux上设置防火墙变得复杂，没有Bridge存在时，Linux主机从某个网卡收到的数据包都是发往本机的，数据包会依次经过...,链路层(ebtables)，IP层(iptables),...解封装最后到达应用层，这样当数据到达IP层时可以很方便用iptables过滤。但是若Bridge存在时，可以参考上图，Bridge上的接口vnet0从外部收到二层数据帧，关键在于此数据包不一定是发往本机，可能是发往Bridge上的另一个设备接口vnet1(tap设备，连接用户层程序，比如虚拟机网卡)，此时经过`bridging decision`之后，数据包会被转发到vnet1，结果是连接vnet1的虚拟机网卡eth0收到数据包(此时数据包已进入虚拟机内部)。可以看出来，整个过程此数据包不会进入主机内核协议栈上层(上图中向上的箭头路径)，因此位于主机IP层的iptables根本无法过滤此数据包(当然可以选择在虚拟机内部使用iptables过滤)。如果从防火墙角度看，在`bridging decision`之后，数据包会走链路层防火墙ebtables的`forward`链(上图中蓝色方框的`filter-forward`)        

><small>What is bridge-nf?       
	
It is the infrastructure that enables {ip,ip6,arp}tables to see bridged IPv4, resp. IPv6, resp. ARP packets. Thanks to bridge-nf, you can use these tools to filter bridged packets, letting you make a transparant firewall. Note that bridge-nf is also referred to as bridge-netfilter and br-nf, the term bridge-nf should be preferred.<small>    
  
来自：[Bridge-nf Frequently Asked Questions](http://ebtables.netfilter.org/misc/brnf-faq.html)     

为了解决以上问题，Linux内核引入了`bridge_netfilter`，简称`bridge_nf`。还是参考上图说明，它使得主机IP层防火墙工具{ip,ip6,arp}tables能够"看见"Bridge中二层数据包(Link Layer)，从而我们能够在主机上使用iptables过滤指定的数据包，不管此数据包是发给主机本身，还是通过Bridge转发给某台虚拟机。上图中绿色方框的`managle-forward`和`filter-forward`即是iptables插入到链路层的链。因此完整的说，在`bridging decision`之后，数据包会依次经过链路层防火墙ebtables的`forward`链(上图中蓝色方框的`filter-forward`)，iptables的`managle-forward`和`filter-forward`链，之后被送入`vnet1`接口     

从Linux 2.6.1内核开始，可以通过设置内核参数开启`bridge_netfilter`机制。看名字就很容易知道具体作用       

``` shell
[root@controller ~]# sysctl -a |grep 'bridge-nf-'
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
...
``` 

然后在iptables中使用`-m physdev`引入相应模块，比如下面iptables规则：不跟踪从网桥`br1`的`vnet1`接口进入的数据包      

``` shell
#网桥
#brctl show
bridge name     bridge id               STP enabled     interfaces
br1             8000.f8bc1212c3a0       no              em1
                                                        vnet1
                                                        vnet3
#查看help
#iptables -m physdev -h
...
                             
#ptables -t raw -A PREROUTING -m physdev --physdev-in vnet1  -j NOTRACK    
```

在OpenStack部署中，若使用Bridge实现虚拟网络，其安全组功能就是依靠`bridge_nf`实现，计算节点上iptables才能"看见"并过滤发往其上所有instance的数据包，如下是OpenStack计算节点上部分iptables规则    

``` shell
...
-A neutron-filter-top -j neutron-linuxbri-local
-A neutron-linuxbri-FORWARD -m physdev --physdev-out tap10f15e45-aa --physdev-is-bridged -m comment --comment "Direct traffic from the VM interface to the security group chain." -j neutron-linuxbri-sg-chain
-A neutron-linuxbri-FORWARD -m physdev --physdev-in tap10f15e45-aa --physdev-is-bridged -m comment --comment "Direct traffic from the VM interface to the security group chain." -j neutron-linuxbri-sg-chain
...
```
 
Bridge+netfilter内容很多，下次会专门用一篇文章介绍OpenStack中的安全组实现，关键字有 `iptables+bridge+netfilter+conntrack`        

Linux上与Bridge类似都可以作为虚拟交换机的是OVS，主要区别是OVS支持vlan tag以及流表(例如openflow)等一些高级特性，Bridge只是单纯二层交换机也不支持vlan tag，OVS具体介绍参考这里[openstack底层技术-使用openvswitch](http://www.isjian.com/openstack/openstack-base-use-openvswitch/)         

# VLAN       

上面简单介绍过Bridge和OVS区别，要在Linux上实现一个带VLAN功能的虚拟交换机，OVS可以通过给不同port打不同tag实现vlan功能，而Bridge需要结合VLAN设备才能实现。VLAN又称虚拟网络，其基本原理是在二层协议里插入额外的VLAN协议数据（称为 802.1.q VLAN Tag)，同时保持和传统二层设备的兼容性。Linux 里的VLAN设备是对 802.1.q 协议的一种内部软件实现，模拟现实世界中的 802.1.q 交换机。详细介绍参考IBM文章中"VLAN device for 802.1.q"部分，这里不再重复，总结几点   

-     

  


# TUN/TAP      

#Veth pair        

# gre/vxlan  

# macvlan

#macvtap

# 总结    
