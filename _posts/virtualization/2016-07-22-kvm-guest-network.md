---
title: "网络虚拟化在kvm中的应用"
author: opengers
layout: post
permalink: /virtualization/network-virtualization-on-kvm/
categories: virtualization
tags:
  - virtualization
  - kvm
  - libvirt
format: quote
---

> Linux中可以创建多种虚拟网络设备，Bridge/OVS，虚拟路由器，tap/tun设备等，这些设备就像物理中的网络设备一样，组成了虚拟机网路拓扑  
本文Host为CentOS7.2，qemu-kvm，libvirt等软件使用yum安装

### Linux中的虚拟网络设备  
云计算的最大特点就是弹性计算，物理网络中各种网络设备相对不容易变动，无法做到弹性扩展。网络虚拟化要解决的就是如何在通用的物理硬件设备上虚拟出各种各样的网络拓扑，并且做到灵活变更。Linux系统中可以创建多种网络设备，利用这些虚拟设备，我们可以组建出复杂灵活的虚拟机网络
