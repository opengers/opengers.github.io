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

><small>本文依然跟之前"openstack底层技术-xxx"文章一样，不讨论具体的openstack配置，而是介绍下openstack中使用到的虚拟网络设备，包括Bridge，Vlan，tun/tap, veth，vxlan/gre</small>    

IBM网站上有一篇高质量文章[Linux 上的基础网络设备详解](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/)相信很多人都已经看过。本文尝试以IBM这篇文章为基础以及结合在openstack中的具体应用，来更全面了解下这些Linux上的虚拟网络设备     

本文使用环境如下   

``` shell
CentOS Linux release 7.3.1611 (Core) 
Linux controller 3.10.0-514.16.1.el7.x86_64 #1 SMP Wed Apr 12 15:04:24 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
OpenStack社区版 Newton
```   

OpenStack一般分为计算，存储，网络三部分。考虑构建一个灵活的可扩展的云网络环境，而物理网络架构一般是固定和难于扩展的，因此虚拟网络设备将更有优势。Linux平台上实现了各种不同功能的虚拟网络设备，包括`Bridge,Vlan,tun/tap,veth pair,vxlan/gre，...`，这些虚拟设备就像一个个积木块一样，被OpenStack组合用于构建虚拟网络。 就像Docker，其脱胎于Linux平台上的`namspace`,以及更早的`chroot`。   

# Linux Bridge   

使用的内核模块`bridge`     

``` shell
[root@controller ~]# modinfo bridge
filename:       /lib/modules/3.10.0-514.16.1.el7.x86_64/kernel/net/bridge/bridge.ko
...
```

Bridge应该是Linux上用的最多的虚拟设备之一，Bridge类似现实中的物理交换机，它可以绑定若干个网络设备(emx,ethx,tap,..)，这些网络设备被虚拟为接口连接到Bridge上，从接口收到的数据包都被直接送到Bridge中(bridge内核模块)，Bridge进行一个类似物理交换机的查MAC端口映射表，转发，更新MAC端口映射表这样的处理逻辑。从而数据包可以被转发到另一个接口/丢弃/广播/发往上层协议栈。由此Bridge实现了数据包在多个接口之前的交换处理。      

跟物理交换机不同的一点是，运行Bridge的是一个Linux主机，Linux主机本身也需要能够通过IP收发数据。但被添加到Bridge上的接口设备是不能配置IP地址的，他们是Bridge上attach的二层设备，对路由系统不可见。不过Bridge本身可以设置IP地址，可以认为当使用`brctl addbr br0`新建一个`br0`网桥时，系统自动创建了一个同名的隐藏`br0`网络设备，给`br0`设置IP后，网络中其它主机便可以通过路由表或IP在三层定位`br0`，当有目的地址为此IP的数据包到达`br0`时，`br0`     会直接将数据包发往上层协议栈，也即Bridge认为它收到了目的地址为自身的数据包。当然你可以选择主机上没有被桥接的物理网卡设置IP地址   
![bridge](/images/openstack/openstack-virtual-devices/bridge.png)    

在Linux中，`route -n`输出的最后一列`Iface`列出了主机路由系统发送数据的可用网络设备。当Bridge设置IP后，Bridge设备也会出现在这里   

Bridge从某个接口收到的数据包，可以根据以下情况进行转发      

- 单播并且目的MAC地址为此网桥上其它接口，直接转发到对应接口   
- 端口映射表中无法找到匹配目的MAC地址的项，发送到Bridge连接的所有接口    
- 包目的MAC为Bridge本身MAC地址，收到发往自身的数据包，交给上层协议栈        
- 数据包目的地址接口不是网桥接口，桥不处理，交给上层协议栈     

   

Linux上与Bridge类似可以做网桥的是OVS，他们其中一个区别是OVS中的接口支持划分不同vlan，Bridge不支持划分vlan接口，

# Vlan     

# TUN/TAP      

#Veth pair        

# gre/vxlan  

# 总结    
