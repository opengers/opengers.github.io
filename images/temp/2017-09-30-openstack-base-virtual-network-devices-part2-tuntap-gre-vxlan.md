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

可以把tun/tap看成数据管道，它一端连接主机协议栈，另一端连接用户程序   
