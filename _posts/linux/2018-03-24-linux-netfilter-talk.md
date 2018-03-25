---
title: openstack底层技术-netfilter框架        
author: opengers
layout: post
permalink: /openstack/openstack-base-netfilter/
categories: openstack
tags:
  - openstack
  - tun/tap
  - veth
  - vxlan/gre
---

[openstack底层技术-各种虚拟网络设备一(Bridge,VLAN)](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/)     
[openstack底层技术-各种虚拟网络设备二(tun/tap,veth)](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/)     

><small>第一篇文章介绍了Bridge和VLAN，本文继续介绍tun/tap，veth等虚拟设备，除了tun，其它设备都能在openstack中找到应用，这些各种各样的虚拟网络设备使网络虚拟化成为了可能</small>      

* TOC
{:toc}    

# tun/tap  