---
title: openstack底层技术-openflow在OVS中的应用   
author: opengers
layout: post
permalink: /openstack/openstack-base-openflow-in-openvswitch/
categories: openstack
tags:
  - openstack
  - openflow
  - openvswitch
---

* TOC
{:toc}    

[上篇文章](http://www.isjian.com/openstack/openstack-base-use-openvswitch/) 介绍了OVS和OpenFlow一些基础知识，本篇主要介绍下OVS中的OpenFlow语法，以及分析下openstack Neutron+OVS网络模式下的流表项             

# OpenFlow概念               

OpenFlow技术最早由斯坦福大学提出，旨在基于现有TCP/IP技术条件，以创新的网络互联理念解决当前网络面对新业务产生的种种瓶颈，它的核心思想很简单，就是将原本完全由交换机/路由器控制的数据包转发过程，转化为由OpenFlow交换机(OpenFlow Switch)和OpenFlow控制器(Controller)分别完成的独立过程。转变背后进行的实际上是控制权的更迭：传统网络中数据包的流向是人为指定的，虽然交换机、路由器拥有控制权，却没有数据流的概念，只进行数据包级别的交换；而在OpenFlow网络中，统一的控制器取代路由，决定了所有数据包在网络中传输路径。 本段参考[什么是OpenFlow](http://network.51cto.com/art/201105/264181.htm)             

现在应该能明白网上关于OpenFlow资料经常提到的一句话`OpenFlow协议实现了控制层面和转发层面的分离`，控制层面就是指OpenFlow控制器，分离就是说控制器负责控制转发规则，交换机只负责执行转发工作，他们可以通过IP网络使用OpenFlow协议连接，不需要位于同一台主机上         

OpenFlow交换机会在本地维护一份从控制器获取的流表(Flow Table),如果要转发的数据包在本地流表中有对应项，则直接进行快速转发，若流表中没有此项，数据包就会被发送到控制器进行传输路径的确认，再根据下发结果进行转发。OpenFLow协议原理本身就很复杂，而其控制器的研究实现更为复杂。因此，本文关注的是OpenFLow这一协议在OVS中的具体应用              

# OVS中的OpenFlow flow介绍       

OpenFlow在OVS中被用于管理流表，其通过灵活强大的流(flow)规则来对进入OVS的数据包进行转发/修改或DROP，基于这点，OpenStack这类云平台要实现网络虚拟化，OVS是一个比`Linux Bridge`更好的选择，前一篇介绍OVS的文章说过，OVS中有多种flow存在，本文中使用的`OpenFlow flow`或者`flow`都是指由OpenFlow协议实现的flow      

flow支持通配符，优先级，多表数据结构。下图是一个进入OVS的数据包(Packet)被flow处理的过程。OVS中可以有一个或者多个流表(flow table)，每个流表包括多条流表项(Flow entry)，每条流表项主要包含多个匹配字段(match fields)、匹配成功后要执行的指令集(action set)和统计信息。        

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)      

我们输出一个`OpenStack Neutron+Vlan`网络模式下控制节点`br-int`网桥中的流表信息，可以看到其有三个流表(table=0/23/24),流表匹配顺序是`0->n`，其中table 0有6条流表项。这6条流表项匹配优先级是`priority`字段值越大，优先级越高。`priority`字段值相同的就按顺序匹配。要注意数据包并不总是严格按照table从小到大的顺序往下走，因为某条流表项的action部分可能改变数据包匹配流程，假如数据包匹配到的某条流表项动作`action`是转发到指定的table，那此时数据包就会被转发到指定的table中去匹配。因此数据包的匹配规则是由table、priority、action共同决定的    

``` shell
#ovs-ofctl dump-flows br-int 
NXST_FLOW reply (xid=0x4):
 cookie=0xa132bd1e50f6ef27, duration=974057.527s, table=0, n_packets=716807, n_bytes=70185583, idle_age=0, hard_age=65534, priority=3,in_port=1,dl_vlan=1023 actions=mod_vlan_vid:2,NORMAL
 cookie=0xa132bd1e50f6ef27, duration=974057.518s, table=0, n_packets=697433, n_bytes=65506238, idle_age=0, hard_age=65534, priority=3,in_port=2,vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:1,NORMAL
 cookie=0xa132bd1e50f6ef27, duration=974059.789s, table=0, n_packets=2, n_bytes=188, idle_age=65534, hard_age=65534, priority=2,in_port=2 actions=drop
 cookie=0xa132bd1e50f6ef27, duration=974060.305s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa132bd1e50f6ef27, duration=974060.303s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
#省略了部分流表项     
```    
   
上面的流表是Neutron实现的OpenFlow控制器下发到OVS中的。作为学习测试，我们不需要通过连接控制器去生成流表项，可以使用`ovs-ofctl`工具去操作OVS中的流表项。`ovs-ofctl add-flow `接收多个逗号或空格分开的`field=value`型字段作为参数，其作用是添加一条流表项到OVS bridge，看下面这条添加流表命令    

``` shell
#ovs-vsctl add-br br0
ovs-ofctl add-flow br0 "priority=3,in_port=100,dl_vlan=0xffff,actions=mod_vlan_vid:101,normal"
```

解释一下就是，向网桥`br0`中添加一条流表项(Flow entry)，这条流表项在其table中优先级为3，其匹配字段指定的规则为：①数据包从port 100进入交换机`br0`(可以用ovs-ofctl show br0查看port)，②数据包不带VLAN tag(dl_vlan=0xffff)。对于这两个条件都匹配的数据包，执行如下action：①先给数据包打上vlan tag 101，②之后交给OVS自身转发，不再受openflow流表控制。可以看到action可以有多个并且按顺序执行，这里对flow有一个简单了解，下面具体说明OVS中的openflow flow语法            

# OVS中的flow语法          

flow中的每条流表项包含多个匹配字段(match fields)、以及指令集(action set)，先总结下常用的匹配字段      

**flow匹配字段**   

| 匹配字段 | 解释 |
| --------- | -------- |
| in_port=port | int类型，数据包进入的端口号，`ovs-ofctl show br0`可以查看port number |    
| dl_vlan=vlan | 0-4095,或0xffff，数据包的VLAN Tag，0xffff表示此数据包不带VLAN Tag |    
| dl_src=xx:xx:xx:xx:xx:xx | MAC地址格式，数据包源MAC(e.g. 00:0A:E4:25:6B:B0) |      
| dl_dst=xx:xx:xx:xx:xx:xx | MAC地址格式，数据包目的MAC(e.g. 00:0A:E4:25:6B:B0) |    
| 01:00:00:00:00:00/01:00:00:00:00:00 | 匹配所有多播或广播数据包 |   
| 00:00:00:00:00:00/01:00:00:00:00:00 | 匹配所有单播数据包 |   
| dl_type=ethertype | 0到65535的长整数或者16进制表示，匹配以太网数据包类型，EX：0x0800(IPv4数据包) 0x0806(arp数据包) | 
| nw_src=ip[/netmask] / nw_dst=ip[/netmask] | `dl_type`字段为IPv4数据包就匹配源或目的ip地址，为arp数据包就匹配ar_spa或ar_tpa，若dl_type字段为通配符，这两个参数会被忽略 |    
| tcp_src=port / tcp_dst=port / udp_src=port / udp_dst=port | 匹配TCP或UDP的源或目的端口，当然，若dl_type字段为通配符或者未明确协议类型是，这些字段会忽略 |  
{:.mbtablestyle}   

这里列举了常用的几个匹配字段，还有很多其它匹配字段，比如可以匹配TCP数据包flag SYN/ACK，可以匹配ICMP协议类型，若一个数据包从tunnel(gre/vxlan)进入的，还可以匹配其`tunnel id`；关于当前OVS版本支持的所有匹配字段，可以查看`man ovs-ofctl`中`Flow Syntax`部分有很详细的解释，主要是掌握编写flow的语法，这样具体用到某字段可以很快用man手册找到并测试其具体用法         

上面提到flow支持通配符，添加flow只能指定有限的几个字段，对于未指定的字段则默认为通配符，因此若某条添加flow命令中所有匹配字段都为通配符，那么这条flow将匹配所有数据包        

**flow指令集**     

action字段语法为`actions=[action][,action...]`，多个action用逗号隔开，指定匹配某条流表项的数据包要执行的指令集，要注意的是，若未指定任何action，数据包会被DROP     

| 匹配字段 | 解释 |   
| --------- | -------- |     
| output:PORT | PORT为openflow端口号，数据包从端口PORT发出，若PORT为数据包进入端口，则不执行 |    
| normal | 数据包交由OVS自身的转发规则完成转发，不在匹配任何openflow flow |    
| all | 数据包从网桥上所有端口发出，除了其进入端口 |     
| drop | 丢弃数据包，当然，drop之后不能再跟其它action |    
| mod_vlan_vid:vlan_vid | 添加或修改数据包中的VLAN tag |     
| strip_vlan | 移除数据包中的VLAN tag，如果有的话 |       
| mod_dl_src:mac / mod_dl_dst:mac | 修改源或目的MAC地址 |    
| mod_nw_src:ip / mod_nw_dst:ip | 修改源或者目的ip地址 |    
mod_tp_src:port / mod_tp_dst:port | 修改TCP或UDP数据包的源或目的端口号(注意不是openflow端口号) |     
| resubmit([port],[table]) | 若port指定,替换数据包in_port字段,并重新匹配,若table指定，提交数据包到指定table，并匹配 |    

同样，还有很多其它的action未列出，这里的`normal`需要解释一下，我们说`Linux Bridge`是一个简单的二层交换机，它像物理交换机那样依靠MAC地址学习在其内部生成一张MAC地址与port对应表，并依靠这张表完成数据包转发。OVS在未配置任何openflow flow的情况下，也是使用这种简单的MAC地址方式转发。但是若OVS配置有OpenFlow flow，则此时进入OVS的数据包会根据OpenFlow flow规则进行转发，只有当某条流表项指定action为`normal`，此时匹配此条流表项的数据包才会脱离OpenFlow的控制，交给OVS使用MAC地址学习完成转发(normal模式)，后面数据包如何被处理，就跟flow没关系了      

# OVS中的控制器      

pass

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

# OVS中使用vlan   


|-----------------+------------+-----------------+----------------|
| Default aligned |Left aligned| Center aligned  | Right aligned  |
|-----------------|:-----------|:---------------:|---------------:|
| First body part |Second cell | Third cell      | fourth cell    |
| Second line     |foo         | **strong**      | baz            |
| Third line      |quux        | baz             | bar            |
|-----------------+------------+-----------------+----------------|
| Second body     |            |                 |                |
| 2 line          |            |                 |                |
|=================+============+=================+================|
| Footer row      |            |                 |                |
|-----------------+------------+-----------------+----------------|


