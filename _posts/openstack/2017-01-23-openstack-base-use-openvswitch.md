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

就像条条大路都能走到罗马一样，折腾openstack肯定得了解下openvswitch(OVS)   

在了解OVS过程中，发现这两篇文章写得真心不错，推荐一下    
(OpenStack网络基础知识: OpenvSwitch使用指南)[http://fishcried.com/2016-02-09/openvswitch-ops-guide/]
(基于 Open vSwitch 的 OpenFlow 实践)[https://www.ibm.com/developerworks/cn/cloud/library/1401_zhaoyi_openswitch/]  

OVS是一款开源的虚拟交换机，其能够满足各种虚拟化平台对网络自定义的需求，在虚拟化平台上，OVS 可以为动态变化的端点提供2层交换功能，很好的控制虚拟网络中的访问策略、网络隔离、流量监控等等。OVS也支持OpenFlow协议，该协议允许OVS使用流表(FlowTable)来建立其端口之间数据包的交换规则。以上这些优点，使OVS成为openstack所实现的网络虚拟化中的基石       

本文会讨论下OVS在openstack中的使用，当然，如果你没有openstack环境，那就仅仅把本文看做介绍OVS的文章，也不影响阅读  

先来介绍下OVS
