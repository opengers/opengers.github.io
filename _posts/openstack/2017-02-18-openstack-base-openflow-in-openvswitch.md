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

[上篇文章](http://www.isjian.com/openstack/openstack-base-use-openvswitch/) 介绍了OVS和openflow一些基础知识，本篇主要介绍下OVS中的OpenFlow语法，以及分析下openstack Neutron+OVS网络模式下的流表项             

# OpenFlow介绍           

上篇文章已经说过，OpenFlow是一个开源的用于管理交换机流表的协议，其通过灵活强大的流(flow)规则来对进入交换机的数据包进行转发/修改或DROP。OpenFLow协议原理本身就很复杂，而其控制器的研究更为复杂。我们文中主要讨论的是其在OVS中的具体应用以及通过修改OVS中的flow来控制数据包的转发。        

OpenFlow的介绍上提到的`OpenFlow协议实现了控制层面和转发层面的分离`，控制层面就是指这里的OpenFlow控制器，分离就是说控制器负责控制转发规则，OVS则负责执行转发，他们可以通过IP网络使用OpeenFlow协议连接，不需要位于同一台主机上     

# OVS工作模式     

OVS有多种工作模式，可以是一个简单的基于MAC地址学习的二层交换机(像Linux bridge一样)，也可以连接OpenFLow控制器作为一个SDN交换机，最常用场景还是作为SDN交换机(比如OpenStack Neutron OVS+vxlan/gre网络模式)，这里根据OpenStack Neutron中对OVS的使用总结一下OVS不同的工作模式                   
  
**使用OpenFlow flows的转发策略**                

- OVS连接OpenFLow控制器，控制器下发flows到OVS，OVS按照下发的flows执行数据转发。当有新的MAC地址加入(新建VM)，或者MAC地址从一个Port移到另一个Port上时(虚拟机迁移)，控制器会更新流表规则以匹配此改变，可见外部控制器决定着OVS中的流表规则，需要注意的是可以是同一个控制器管理多台计算节点上的OVS    

- 比如openstack OVS+Vxlan网络部署模式下的`br-tun`网桥就是依据OpenFlow flows完成转发，其连接了一个OpenFlow控制器`Controller "tcp:127.0.0.1:6633"`                 

- 还有一些其它话题，比如当某条流表项中的执行动作为`normal`时，OpenFlow会把匹配到这条规则的数据包丢给OVS自身处理，这些数据包就不再匹配其它的流表规则。还有当外部控制器由于网络故障无法连接时，OVS会根据`fail_mode: secure`中的设置项决定要如何处理            

**OpenFlow控制器+MAC地址学习**      

- OpenFlow flows中执行动作action可以为`NORMAL`，`NORMAL`意思是丢给OVS自身完成转发，不再匹配flows。也即匹配到此flow的数据包之后会脱离flow，走正常的MAC学习转发策略。    

- openstack OVS+Vxlan网络部署模式下的网桥`br-int`，`br-ext`就是这种模式，其连接了Neutron实现的OpenFlow控制器，也用到了`NORMAL`指令    

- OVS自身就支持MAC学习的转发策略，跟上种模式一个明显的区别是OpenFlow只做必要的干预，干预后的数据包通过`NORMAL`使OVS对此数据包做后续的转发   
         
**基于MAC地址学习的转发策略**               

- 此种模式下没有OpenFlow的参与，类似Linux Bridge，只是简单的基于MAC地址完成转发   

- 考虑第一个数据包进入OVS的情况，由于之前没有任何数据包进入，也没有flows规则，OVS无法知道第一个数据包应该从哪个端口发出，此时只能依靠学习喽，OVS会把数据包转发到除了进入Port之外的所有Port，然后根据应答数据包的进入Port来学习MAC地址对应的Port，就像Linux Bridge那样。这种情况下OVS依然可以为Port设置Vlan tag，Linux Bridge不支持设置Vlan    

- 使用ovs-vsctl新建的网桥，默认是没有控制器存在的，是一个简单的二层交换机        

**手动建立流表规则**             

- 前面提到`ovs-ofctl`工具可以通过OpenFlow协议配置OVS中的flows，那我们就自己`add-br`一个网桥，然后建立一些流表项观察数据包转发规则，测试或学习OpenFlow协议时可以这么干    

上面介绍了三种工作模式，总结下就是若OVS中有openflow flow的存在，数据包就会先匹配流表规则，

# OVS中flow语法        

OpenFlow flow支持通配符，优先级，多表数据结构。下图是一个进入OVS的数据包(Packet)被flow处理的过程，flow定义了交换机端口之间数据包的转发规则，OVS中可以有一个或者多个流表(flow table)，每个流表包括多条流表项(Flow entrys)，每条流表项包含多个匹配字段(match fields)、匹配成功后要执行的指令集(action set)和统计信息。

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)             

作为测试，我们不需要通过连接控制器去生成流表项，而是使用`ovs-ofctl`去操作OVS中的流表项。`ovs-ofctl`接收多个逗号或空格分开的`field=value`型字段

     