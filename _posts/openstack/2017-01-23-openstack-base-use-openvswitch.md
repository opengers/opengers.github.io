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
 
关于OVS基础知识，下面这两篇已经介绍得很详细了    
[OpenStack网络基础知识: OpenvSwitch使用指南](http://fishcried.com/2016-02-09/openvswitch-ops-guide/)  
[基于 Open vSwitch 的 OpenFlow 实践](https://www.ibm.com/developerworks/cn/cloud/library/1401_zhaoyi_openswitch/)    

OVS是一款开源的虚拟交换机，其能够满足各种虚拟化平台对网络自定义的需求，在虚拟化平台上，OVS 可以为动态变化的端点提供2层交换功能，很好的控制虚拟网络中的访问策略、网络隔离、流量监控等等。OVS也支持OpenFlow协议，该协议允许OVS使用流表(FlowTable)来建立其端口之间数据包的交换规则。以上这些优点，使OVS成为openstack所实现的网络虚拟化中的基石       

## OVS架构  

先来介绍下OVS
