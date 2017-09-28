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

* TOC
{:toc} 

><small>IBM网站上有一篇高质量文章[Linux 上的基础网络设备详解](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/)。本文会参考文章部分内容，文中主要侧重于介绍OpenStack是如何使用这些虚拟网络设备构建出灵活的云网络，这些网络设备包括Bridge，VLAN，tun/tap, veth，vxlan/gre。本文先介绍Bridge和VLAN相关，其它在下一篇part2中介绍</small>   

