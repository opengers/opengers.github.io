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

[上篇文章](http://www.isjian.com/openstack/openstack-base-use-openvswitch/) 介绍了OVS和OpenFlow一些基础知识，本篇主要介绍下OVS中的OpenFlow语法，以及分析下openstack Neutron+OVS网络模式下的流表项             

# OpenFlow概念               

上篇文章已经说过，OpenFlow是一个开源的用于管理交换机流表的协议，其通过灵活强大的流(flow)规则来对进入交换机的数据包进行转发/修改或DROP。OpenFLow协议原理本身就很复杂，而其控制器的研究更为复杂。我们文中主要讨论的是其在OVS中的具体应用       

网上关于OpenFlow的介绍中经常提到`OpenFlow协议实现了控制层面和转发层面的分离`，控制层面就是指这里的OpenFlow控制器，分离就是说控制器负责控制转发规则，OVS则负责执行转发工作，他们可以通过IP网络使用OpenFlow协议连接，不需要位于同一台主机上     

# OVS中的OpenFLow flow介绍      

OpenFlow flow定义了交换机端口之间数据包的转发规则。下图是一个进入OVS的数据包(Packet)被flow处理的过程。OVS中可以有一个或者多个流表(flow table)，每个流表包括多条流表项(Flow entrys)，每条流表项包含多个匹配字段(match fields)、匹配成功后要执行的指令集(action set)和统计信息。            

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)      

flow支持通配符，优先级，多表数据结构。我们输出一个`OpenStack Neutron+Vlan`网络模式下控制节点`br-int`网桥中的流表信息，可以看到其有三个流表(table=0/23/24),流表匹配顺序是`0->n`，其中table 0有6条流表项。这6条流表项的匹配顺序是`priority`字段值越大，优先级越高。`priority`字段值相同的就按顺序匹配。要注意数据包并不总是严格按照table从小到大的顺序往下匹配，假如数据包匹配到的某条流表项动作`action`是转发到指定的table，那此时数据包就会被转发到指定的table中去。因此数据包的匹配规则是由table、priority、action共同决定的    

``` shell
#ovs-ofctl dump-flows br-int 
NXST_FLOW reply (xid=0x4):
 cookie=0xa132bd1e50f6ef27, duration=974057.527s, table=0, n_packets=716807, n_bytes=70185583, idle_age=0, hard_age=65534, priority=3,in_port=1,dl_vlan=1023 actions=mod_vlan_vid:2,NORMAL
 cookie=0xa132bd1e50f6ef27, duration=974057.518s, table=0, n_packets=697433, n_bytes=65506238, idle_age=0, hard_age=65534, priority=3,in_port=2,vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:1,NORMAL
 cookie=0xa132bd1e50f6ef27, duration=974059.789s, table=0, n_packets=2, n_bytes=188, idle_age=65534, hard_age=65534, priority=2,in_port=2 actions=drop
 cookie=0xa132bd1e50f6ef27, duration=974060.307s, table=0, n_packets=2749, n_bytes=135739, idle_age=184, hard_age=65534, priority=0 actions=NORMAL
 cookie=0xa132bd1e50f6ef27, duration=974060.305s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa132bd1e50f6ef27, duration=974060.303s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
#上面省略了部分重复流表项
```    
   
上面的流表是Neutron实现的OpenFlow控制器下发到OVS中的。作为学习测试，我们不需要通过连接控制器去生成流表项，而是使用`ovs-ofctl`去操作OVS中的流表项。`ovs-ofctl add-flow `接收多个逗号或空格分开的`field=value`型字段作为参数，其作用是添加一条流表项到指定bridge，  看下面这条添加流表命令    

``` shell
ovs-ofctl add-flow br0 "priority=3,in_port=100,dl_vlan=0xffff,actions=mod_vlan_vid:101,normal"
```

解释一下就是，向网桥`br0`中添加一条流表项，这条流表项的匹配字段指定的规则为：①从port 100进入交换机`br0`(可以用)，②不包含任何VLAN tag(dl_vlan=0xffff)，若所有条件匹配，则执行action：①先给数据包打上vlan tag 101，②之后走OVS自身转发策略，不再受openflow flow影响，可以看到action可以有多个，按顺序执行，这里是先对流表有一个简单了解，下面具体说明OVS中的openflow flow语法         
     

# OVS中的OpenFLow flow语法      

flow中的每条流表项包含多个匹配字段(match fields)、以及指令集(action set)，这里先列举下常用的匹配字段，关于当前OVS版本支持的所有匹配字段，可以查看`man ovs-ofctl`中`Flow Syntax`部分   

**flow匹配字段**   

| 表头1  | 表头2|
| --------- | --------|
| 表格单元  | 表格单元 |
| 表格单元  | 表格单元 |

| in_port=port	| 传递数据包的端口的 OpenFlow 端口编号 |    
dl_vlan=vlan	数据包的 VLAN Tag 值，范围是 0-4095，0xffff 代表不包含 VLAN Tag 的数据包
dl_src=<MAC>	匹配源或者目标的 MAC 地址
dl_dst=<MAC>	匹配源或者目标的 MAC 地址

01:00:00:00:00:00/01:00:00:00:00:00 代表广播地址
00:00:00:00:00:00/01:00:00:00:00:00 代表单播地址
dl_type=ethertype	匹配以太网协议类型，其中：
dl_type=0x0800 代表 IPv4 协议
dl_type=0x086dd 代表 IPv6 协议
dl_type=0x0806 代表 ARP 协议

完整的的类型列表可以参见以太网协议类型列表
nw_src=ip[/netmask]
nw_dst=ip[/netmask]	当 dl_typ=0x0800 时，匹配源或者目标的 IPv4 地址，可以使 IP 地址或者域名
nw_proto=proto	和 dl_type 字段协同使用。
当 dl_type=0x0800 时，匹配 IP 协议编号
当 dl_type=0x086dd 代表 IPv6 协议编号

完整的 IP 协议编号可以参见IP 协议编号列表
table=number	指定要使用的流表的编号，范围是 0-254。在不指定的情况下，默认值为 0。通过使用流表编号，可以创建或者修改多个 Table 中的 Flow
reg<idx>=value[/mask]	交换机中的寄存器的值。当一个数据包进入交换机时，所有的寄存器都被清零，用户可以通过 Action 的指令修改寄存器中的值
  
 
# OVS工作模式      

OVS有多种工作模式，默认情况下使用`ovs-vsctl`创建的OVS bridge没有连接任何openflow控制器，其内部也没有openflow flow规则，此时OVS就像物理世界中的二层交换机一样(类似Linux bridge)，其数据包转发完全依靠MAC地址学习完成。当然，OVS也可以连接OpenFLow控制器作为一个SDN交换机(比如OpenStack Neutron OVS+vxlan/gre网络模式)，这是OVS比Linux Bridge强大之处，这里根据OpenStack Neutron中对OVS的使用总结一下OVS不同的工作模式                    
  
**使用OpenFlow flows的转发策略**            

- 此时OVS连接有OpenFLow控制器，控制器下发flows到OVS，OVS按照下发的flows执行数据转发。当有新的MAC地址加入(新建vm)，或者MAC地址从一个Port移到另一个Port上时(vm迁移)，控制器会更新流表规则以匹配此改变，可见OpenFLow控制器决定着OVS中的数据包转发规则，需要注意的是一个控制器可以管理多台计算节点上的OVS     

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

上面介绍了三种工作模式，总结下就是若OVS中有openflow flow的存在，数据包就会先匹配流表规则      

     