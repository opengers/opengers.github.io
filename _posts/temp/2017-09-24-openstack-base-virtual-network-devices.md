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

内核模块`bridge`     

``` shell
[root@controller ~]# modinfo bridge
filename:       /lib/modules/3.10.0-514.16.1.el7.x86_64/kernel/net/bridge/bridge.ko
...
```

Linux Bridge应该是最熟悉的一个，可以直接看IBM文章了解Bridge设备工作原理，这里总结下关于Bridge的特点   

Linux Bridge类似现实中的物理交换机，它可以绑定若干个以太网接口设备(emx,ethx,tap,..)，从而将它们桥接起来，Bridge依靠二层MAC地址在接口间转发数据包。对于Linux协议栈IP层来说，只看得到br0，因为桥接是在数据链路层实现的，上层不需要关心桥接的细节。于是协议栈上层需要发送的报文被送到br0，网桥设备的处理代码再来判断报文该被转发到eth0或是eth1，或者两者皆是；反过来，从eth0或从eth1接收到的报文被提交给网桥的处理代码，在这里会判断报文该转发、丢弃、或提交到协议栈上层。
而有时候eth0、eth1也可能会作为报文的源地址或目的地址，直接参与报文的发送与接收（从而绕过网桥）。
      
Linux下的bridge设备，对下层而言是一个桥设备，进行数据的转发（实际上对下也有接收能力，下一节讲）。而对上层而言，它就像普通的ethernet设备一样，有自己的IP和MAC地址，那么上层当然可以把它加入路由系统，并利用它发送数据啦，并且很容易想到，它的发射函数最终肯定是利用某个从设备的驱动去完成实际的发送的，这个和VLAN是相通的。


但是 Linux 里 Bridge 是通用网络设备抽象的一种，只要是网络设备就能够设定 IP 地址。当一个 bridge0 拥有 IP 后，Linux 便可以通过路由表或者 IP 表规则在三层定位 bridge0

Linux下的Bridge也是一种虚拟设备，这多少和vlan有点相似，它依赖于一个或多个从设备。与VLAN不同的是，它不是虚拟出和从设备同一层次的镜像设备，而是虚拟出一个高一层次的设备，并把从设备虚拟化为端口port
可见br设备是建立在从设备之上的（这些从设备可以是实际设备，也可以是vlan设备等），并且可以为br准备一个IP（br设备的MAC地址是它所有从设备中最小的MAC地址），这样该主机就可以通过这个br设备与网络中的其它主机通信了（详见发送功能框图）。
另外它的从设备被虚拟化为端口port，它们的IP及MAC都不再可用，且它们被设置为接收任何包，最终由bridge设备来决定数据包的去向：接收到本机、转发、丢弃（详见接收功能框图）。

linux内核支持网口的桥接（目前只支持以太网接口）。但是与单纯的交换机不同，交换机只是一个二层设备，对于接收到的报文，要么转发、要么丢弃。小型的交换机里面只需要一块交换芯片即可，并不需要CPU。而运行着linux内核的机器本身就是一台主机，有可能就是网络报文的目的地。其收到的报文除了转 发和丢弃，还可能被送到网络协议栈的上层（网络层），从而被自己消化



进入桥的数据报文分为几个类型，桥对应的处理方法也不同：
1.  报文是本机发送给自己的，桥不处理，交给上层协议栈；
2.  接收报文的物理接口不是网桥接口，桥不处理，交给上层协议栈；
3.  进入网桥后，如果网桥的状态为Disable，则将包丢弃不处理；
4.  报文源地址无效（广播，多播，以及00:00:00:00:00:00），丢包；
5.  如果是STP的BPDU包，进入STP处理，处理后不再转发，也不再交给上层协议栈；
6.  如果是发给本机的报文，桥直接返回，交给上层协议栈，不转发；
7.  需要转发的报文分三种情况：

1） 广播或多播，则除接收端口外的所有端口都需要转发一份；
2） 单播并且在CAM表中能找到端口映射的，只需要网映射端口转发一份即可；
3） 单播但找不到端口映射的，则除了接收端口外其余端口都需要转发；















**Bridge特点**     

**Bridge接口**     

**Bridge设置IP**     

**Bridge转发**    

**Bridge+Vlan**     

**Bridge+netfilter**     


# Vlan     

# TUN/TAP      

#Veth pair        

# gre/vxlan  

# 总结    
