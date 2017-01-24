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
 
关于OVS基础知识，下面这两篇已经介绍得很详细了，本文主要谈论一些关于OVS的其它话题，基本概念只简单总结下   
[OpenStack网络基础知识: OpenvSwitch使用指南](http://fishcried.com/2016-02-09/openvswitch-ops-guide/)  
[基于 Open vSwitch 的 OpenFlow 实践](https://www.ibm.com/developerworks/cn/cloud/library/1401_zhaoyi_openswitch/)    

OVS是一款开源的虚拟交换机，其能够满足各种虚拟化平台对网络自定义的需求，在虚拟化平台上，OVS可以为动态变化的端点提供2层交换功能，很好的控制虚拟网络中的访问策略、网络隔离、流量监控等等。OVS也支持OpenFlow协议，该协议允许OVS使用流表(FlowTable)   来建立其端口之间数据包的交换规则。以上这些优点，使OVS成为openstack所实现的网络虚拟化中的基石   
    
在Linux系统上，应用最广泛的还是系统自带的Linux Bridge，它是一个基于MAC地址学习的二层交换机，简单高效，但缺乏一些高级特性，比如VLAN tag,QOS,流表之类，而且在隧道协议支持方面，Linux Bridge只支持vxlan，OVS是gre或vxlan都支持，这也决定了OVS更适用于大规模虚拟化集群   

## OVS基础  

先看下OVS整体架构    

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-arch.png)

更详细的图   

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-details.png)  

OVS中的

