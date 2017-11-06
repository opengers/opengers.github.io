---
title: openstack底层技术-各种虚拟网络设备之二(tun/tap,veth,vxlan/grep)      
author: opengers
layout: post
permalink: /openstack/openstack-base-virtual-network-devices-part2-tuntap-gre-vxlan/
categories: openstack
tags:
  - openstack
  - tun/tap
  - veth
  - vxlan/gre
---

[openstack底层技术-各种虚拟网络设备之一(Bridge,VLAN)](http://www.isjian.com/openstack/openstack-base-virtual-network-devices-part1-bridge-and-vlan/)     
[openstack底层技术-各种虚拟网络设备之二(tun/tap,veth,vxlan/grep)](http://www.isjian.com/openstack/openstack-base-virtual-network-devices-part2-tuntap-gre-vxlan/)     

* TOC
{:toc}    

上文介绍了Bridgge和VLAN设备，本文继续介绍tun/tap，vxlan/gre等设备，很多时候这些设备不是独立使用的，下文会看到     

# tun/tap   

为了理解`tun/tap`具体是做什么的，先来考虑这个问题，KVM虚拟化中虚拟机实现为Linux中一个标准进程，跟其它普通进程一样接受Linux进程调度器管理，像下面这样`instance-00000145`实际上为主机中的一个`qemu-kvm`进程   

``` shell
[root@compute03 ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 23    instance-00000145              running
 
[root@compute03 ~]# ps -ef |grep instance-00000145
qemu     30792     1  0 Sep26 ?        00:21:23 /usr/libexec/qemu-kvm -name guest=instance-00000145 ...
```

那么这样一个用户空间进程

在计算机网络中，TUN与TAP是操作系统内核中的虚拟网络设备。不同于普通靠硬件网路板卡实现的设备，这些虚拟的网络设备全部用软件实现，并向运行于操作系统上的软件提供与硬件的网络设备完全相同的功能。
TAP 等同于一个以太网设备，它操作第二层数据包如以太网数据帧。TUN模拟了网络层设备，操作第三层数据包比如IP数据封包。
操作系统通过TUN/TAP设备向绑定该设备的用户空间的程序发送数据，反之，用户空间的程序也可以像操作硬件网络设备那样，通过TUN/TAP设备发送数据。在后种情况下，TUN/TAP设备向操作系统的网络栈投递（或“注入”）数据包，从而模拟从外部接受数据的过程。

可以把tun/tap看成数据管道，它一端连接主机协议栈，另一端连接用户程序   
