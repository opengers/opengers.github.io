---
title: openstack底层技术-openvswitch中的openflow流表
author: opengers
layout: post
permalink: /openstack/openstack-base-openflow-in-openvswitch/
categories: openstack
tags:
  - openstack
  - openflow
  - openvswitch
format: quote
---

* TOC
{:toc}    

[上篇文章](http://www.isjian.com/openstack/openstack-base-use-openvswitch/) 介绍了OVS基础知识，还有对OpenFlow协议及其在VOS中的作用也有介绍，本篇主要介绍下OpenFlow语法，以及分析下openstack+OVS网络模式下的流表项             

# OpenFlow介绍           

OpenFlow允许从远程控制网络交换器的数据包转送表，通过新增、修改与移除数据包控制规则与行动，来改变数据包转送的路径。比起用 访问控制表 (ACLs) 和路由协议，允许更复杂的流量管理。同时，OpenFlow允许不同供应商用一个简单，开源的协议去远程管理交换机    

OpenFlow控制器使用这种flows定义OVS数据转发策略。OpenFlow flows支持通配符，优先级，多表数据结构

OpenFlow flows可以有一个或者多个流表，每个流表中包括多条流表项，每条流表项包含：数据包头的信息、匹配成功后要执行的指令集(actions)和统计信息。 当数据包进入OVS后，OVS会将数据包和流表中的流表项进行匹配，如果发现了匹配的流表项，则执行该流表项中的指令集。

`ovs-ofctl`是一个监控和管理OpenFlow交换机的命令行工具，它支持任何使用OpenFlow协议的交换机，不仅仅是OVS      

OpenFlow的介绍上提到的`OpenFlow协议实现了控制层面和转发层面的分离`，控制层面就是指这里的OpenFlow控制器，分离就是说控制器负责控制转发规则，OVS则负责执行转发，他们可以通过IP网络使用OpeenFlow协议连接，不需要位于同一台主机上    

OpenFlow中的flows定义了交换机端口之间数据包的转发规则，以OVS为例，OVS交换机中可以有一个或者多个流表，每个流表包括多个流表项(Flow entrys)，每条流表项中的条目包含：数据包头的信息、匹配成功后要执行的指令和统计信息。当数据包进入OVS后，OVS会将数据包和flows中的流表项进行匹配以决定此数据包是被转发/修改或是DROP。     

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)    

OVS可以有多种工作模式，可以是一个简单的基于MAC地址学习的二层交换机，也可以连接OpenFLow控制器作为一个SDN交换机，最常用场景还是作为SDN交换机(比如OpenStack Neutron vxlan/gre网络模式)，这里根据OpenStack Neutron中对OVS的使用总结一下OVS不同的转发策略               
  
**使用OpenFlow flows的转发策略**                

- OVS连接OpenFLow控制器，控制器下发flows到OVS，OVS按照下发的flows执行数据转发。当有新的MAC地址加入(新建VM)，或者MAC地址从一个Port移到另一个Port上时(虚拟机迁移)，控制器会更新流表规则以匹配此改变，可见外部控制器决定着OVS中的流表规则，需要注意的是可以是同一个控制器管理多台计算节点上的OVS   

- openstack OVS+Vxlan网络部署模式下的`br-tun`网桥就是依据OpenFlow flows完成转发              

- 还有一些其它话题，比如当某条流表项中的执行动作为`normal`时，OpenFlow会把匹配到这条规则的数据包丢给OVS自身处理，这些数据包就不再匹配其它的流表规则。还有当外部控制器由于网络故障无法连接时， 这些情况到后面介绍流表规则时再讨论       

**OpenFlow控制器+MAC地址学习**      

- OpenFlow flows中执行动作action可以为`NORMAL`，`NORMAL`意思是丢给OVS自身完成转发，不再匹配flows。也即匹配到此flow的数据包会走正常的MAC学习转发策略。    

- openstack OVS+Vxlan网络部署模式下的网桥`br-int`，`br-ext`就是这种模式，其连接了Neutron实现的OpenFlow控制器          

**基于MAC地址学习的转发策略**               

- 此种模式下没有OpenFlow的参与，类似Linux Bridge，只是简单的基于MAC地址完成转发   

- 考虑第一个数据包进入OVS的情况，由于之前没有任何数据包进入，也没有flows规则，OVS无法知道第一个数据包应该从哪个端口发出，此时只能依靠学习喽，OVS会把数据包转发到除了进入Port之外的所有Port，然后根据应答数据包的进入Port来学习MAC地址对应的Port，就像Linux Bridge那样。这种情况下OVS依然可以为Port设置Vlan tag，但Linux Bridge不支持设置Vlan    

- 使用ovs-vsctl新建的网桥，默认是没有控制器存在的，是一个简单的二层交换机        

**手动建立流表规则**             

- 前面提到`ovs-ofctl`工具可以通过OpenFlow协议配置OVS中的flows，那我们就自己`add-br`一个网桥，然后建立一些流表项观察数据包转发规则，测试或学习OpenFlow协议时可以这么干   