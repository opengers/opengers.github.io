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

[上篇文章](http://www.isjian.com/openstack/openstack-base-use-openvswitch/) 介绍了OVS架构和主要概念，本篇主要介绍下OpenFlow在OVS中的应用，以及分析下openstack Neutron+OVS网络模式下的流表项             

# 什么是OpenFlow               

OpenFlow技术最早由斯坦福大学提出，旨在基于现有TCP/IP技术条件，以创新的网络互联理念解决当前网络面对新业务产生的种种瓶颈，它的核心思想很简单，就是将原本完全由交换机/路由器控制的数据包转发过程，转化为由OpenFlow交换机(OpenFlow Switch)和OpenFlow控制器(Controller)分别完成的独立过程。转变背后进行的实际上是控制权的更迭：传统网络中数据包的流向是人为指定的，虽然交换机、路由器拥有控制权，却没有数据流的概念，只进行数据包级别的交换；而在OpenFlow网络中，统一的控制器取代路由，决定了所有数据包在网络中传输路径。 本段参考[什么是OpenFlow](http://network.51cto.com/art/201105/264181.htm)             

现在应该能明白网上关于OpenFlow资料经常提到的一句话`OpenFlow协议实现了控制层面和转发层面的分离`，控制层面就是指OpenFlow控制器，分离就是说控制器负责控制转发规则，交换机只负责执行转发工作，他们可以通过IP网络使用OpenFlow协议连接，不需要位于同一台主机上         

OpenFlow交换机会在本地维护一份从控制器获取的流表(Flow Table),如果要转发的数据包在本地流表中有对应项，则直接进行快速转发，若流表中没有此项，数据包就会被发送到控制器进行传输路径的确认，再根据下发结果进行转发。OpenFlow协议原理本身就很复杂，而其控制器的研究实现更为复杂。因此，本文关注的是OpenFlow这一协议在OVS中的具体应用              

# OVS中的OpenFlow           

OVS支持OpenFlow协议，OVS可以通过连接OpenFlow控制器获取流表，也可以使用其提供的命令行工具`ovs-ofctl`手动添加流表项。流表控制着OVS中数据包的转发/修改或DROP。基于这点，OpenStack这类云平台要实现网络虚拟化，OVS是一个比`Linux Bridge`更好的选择，前一篇介绍OVS的文章说过，OVS中有多种flow存在，本文中使用的`OpenFlow flow`或者`flow`都是指由OpenFlow协议实现的flow      

下图是一个进入OVS的数据包(Packet)被流表处理的过程。OVS中可以有一个或者多个流表(flow table)，每个流表包括多条流表项(Flow entry)，每条流表项主要包含多个匹配字段(match fields)、匹配成功后要执行的指令集(action set)和统计信息。        

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)      

flow支持通配符，优先级，多表数据结构。我们输出一个`OpenStack Neutron+vlan`网络模式下控制节点`br-int`网桥中的流表信息，可以看到其有三个流表(table=0/23/24),流表匹配顺序是从`0->n`，其中table 0有6条流表项。这6条流表项匹配优先级是`priority`字段值越大，优先级越高。`priority`字段值相同的就按顺序匹配。要注意数据包并不总是严格按照table从小到大的顺序往下走，因为某条流表项的action部分可能改变数据包匹配流程，例如数据包匹配到的某条流表项动作`action`是转发到指定的table，那此时数据包就会被转发到指定的table中去匹配。因此数据包的匹配规则是由table、priority、action共同决定的     

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
   
上面的流表是Neutron实现的OpenFlow控制器下发到OVS中的。作为学习测试，我们不需要连接控制器去生成流表项，可以使用`ovs-ofctl`工具去操作OVS中的流表项。`ovs-ofctl add-flow`接收多个逗号或空格分开的`field=value`型字段作为参数，其作用是添加一条流表项到OVS bridge，看下面这条添加流表命令    

``` shell
#ovs-vsctl add-br br0
ovs-ofctl add-flow br0 "priority=3,in_port=100,dl_vlan=0xffff,actions=mod_vlan_vid:101,normal"
```

解释一下就是，向网桥`br0`中添加一条流表项(Flow entry)，这条流表项在其table中优先级为3，其匹配字段指定的规则为：①数据包从port 100进入交换机`br0`(可以用ovs-ofctl show br0查看port)，②数据包不带VLAN tag(dl_vlan=0xffff)。对于这两个条件都匹配的数据包，执行如下action：①先给数据包打上vlan tag 101，②之后交给OVS自身转发，不再受openflow流表控制。可以看到action可以有多个并且按顺序执行，这里对flow有一个简单了解，下面具体说明OVS中的openflow flow语法       

# OVS几种工作模式          

先来看下Linux系统自带的`Linux Bridge`是怎样完成数据转发的。它类似物理交换机那样是一个普通的MAC地址学习交换机，其内部维护一张MAC地址和端口映射表，并依靠MAC地址学习方式不断更新此映射关系，`Linux Bridge`就是靠这张映射表完成数据转发。默认使用`ovs-vsctl`创建的bridge没有连接任何openflow控制器，其内部也没有流表存在，此时OVS也是使用这种简单的MAC地址学习方式转发数据包。但是若OVS连接有OpenFlow控制器或OVS中配置有流表，则此时OVS数据转发策略就完全由流表来决定。只有当某条流表项指定action为`normal`，此时匹配此条流表项的数据包才会脱离OpenFlow的控制，由OVS根据MAC地址学习方式完成后续转发，后面数据包如何被处理就跟flow没关系了  

因此OVS可以有多种不同工作模式，这里根据OpenStack Neutron中对OVS的使用总结一下OVS不同的工作模式      
  
**使用流表的OpenFlow交换机**            

- 此时OVS连接有OpenFlow控制器，控制器下发流表到OVS，OVS按照下发的flow执行数据转发。当有新的MAC地址加入(新建vm)，或者MAC地址从一个Port移到另一个Port上时(vm迁移)，控制器会更新流表以匹配此改变，需要注意的是一个控制器可以管理多台计算节点上的OVS     

- 比如openstack OVS+Vxlan网络部署模式下的`br-tun`网桥就是一个OpenFlow交换机，其连接了一个OpenFlow控制器`Controller "tcp:127.0.0.1:6633"`                 

- 当外部控制器由于网络故障无法连接时，OVS会根据`fail_mode: secure`中的设置项决定要如何处理，后面会提到           

**OpenFlow交换机+MAC地址学习**      

- 这种模式就是先用流表对数据包做必要的调整，之后再把数据包交给OVS依靠MAC地址学习完成转发。流表中的aciton:`normal`给数据包提供了一种脱离流表控制的方法         

- openstack OVS+Vxlan网络部署模式下的网桥`br-int / br-ext`就是这种模式，其连接了Neutron实现的OpenFlow控制器，也使用了`normal`这一action。           

- 跟上种模式一个明显的区别是OpenFlow只对数据包做必要的干预，干预后的数据包可以交由OVS自身处理     
         
**普通MAC地址学习交换机**               

- 没有流表的存在，类似Linux Bridge，只是简单的基于MAC地址完成转发     

- 再来解释下MAC地址学习，考虑第一个数据包进入OVS的情况，由于之前没有任何数据包进入，OVS无法知道第一个数据包应该从哪个端口发出，此时只能依靠MAC地址学习喽，OVS会把数据包转发到除了进入Port之外的所有Port，然后记录应答数据包的进入Port及其MAC。这样一条映射关系就建立了           

- 物理设备中有带vlan功能的交换机和不带vlan功能的交换机。在虚拟机交换机中，Linux Bridge不支持设置Vlan，OVS支持设置vlan，关于OVS中使用vlan，下一篇文章会单独介绍           

**普通MAC地址学习交换机+手动添加流表**               

- 前面提到`ovs-ofctl`工具可以配置OVS中的flow，那我们就自己`add-br`一个网桥，然后建立一些流表项观察数据包转发规则，测试或学习OpenFlow协议时可以这么干    

上面介绍了四种工作模式，几种模式并不是互斥的，实际使用是很灵活的。         

# flow语法          

flow中的每条流表项包含多个匹配字段(match fields)、以及指令集(action set)，先总结下常用的匹配字段      

**flow匹配字段**   

| 匹配字段 | 作用 |
| --------- | -------- |
| in_port=port | int类型，数据包进入的openflow端口号，`ovs-ofctl show br0`可以查看port number |    
| dl_vlan=vlan | 0-4095,或0xffff，数据包的VLAN Tag，0xffff表示此数据包不带VLAN Tag |    
| dl_src=xx:xx:xx:xx:xx:xx | 数据包源MAC(e.g. 00:0A:E4:25:6B:B0) |      
| dl_dst=xx:xx:xx:xx:xx:xx | 数据包目的MAC(e.g. 00:0A:E4:25:6B:B0) |    
| 01:00:00:00:00:00/01:00:00:00:00:00 | 匹配所有多播或广播数据包 |   
| 00:00:00:00:00:00/01:00:00:00:00:00 | 匹配所有单播数据包 |   
| dl_type=ethertype | 0到65535的长整数或者16进制表示，匹配以太网数据包类型，EX：0x0800(IPv4数据包) 0x0806(arp数据包) |  
| nw_src=ip[/netmask] nw_dst=ip[/netmask] | `dl_type`字段为`0x0800`时就匹配源或目的ip地址，为`0x0806`就匹配ar_spa或ar_tpa，若dl_type字段为通配符，这两个参数会被忽略 |    
| tcp_src=port tcp_dst=port udp_src=port udp_dst=port |    匹配TCP或UDP的源或目的端口，当然，若dl_type字段为通配符或者未明确协议类型时，这些字段会忽略 |   
{:.mbtablestyle}       
<br />

这里列举了常用的匹配字段，还有很多其它匹配字段，比如可以匹配TCP数据包flag SYN/ACK，可以匹配ICMP协议类型，若一个数据包从tunnel(gre/vxlan)进入的，还可以匹配其`tunnel id`；关于当前OVS版本支持的所有匹配字段，可以查看`man ovs-ofctl`中`Flow Syntax`部分有很详细的解释，主要是掌握编写flow的语法，这样具体用到某字段可以很快用man手册找到并测试其具体用法         

上面提到flow支持通配符，添加流表项时只能指定有限的几个字段，对于未指定的字段则默认为通配符，因此若某条添加flow命令中所有匹配字段都为通配符，那么这条flow将匹配所有数据包，下面新建一个网桥`br-test`测试一下                  

``` shell
#添加br-test
ovs-vsctl add-br br-test
```

然后查看`br-test`中的默认flow    

``` shell  
#ovs-ofctl dump-flows br-test
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=403.569s, table=0, n_packets=28, n_bytes=1800, idle_age=208, priority=0 actions=NORMAL
```

可以看到新创建的`br-test`中默认只有一条flow，也未连接任何控制器，而且此flow中没有匹配字段(priority不是)，因此这条默认添加的flow将匹配所有进入`br-test`的数据包，而其动作NORMAL表明数据包完全按照MAC地址学习方式完成转发        

**flow动作**     

action字段语法为`actions=[action][,action...]`，多个action用逗号隔开，指定匹配某条流表项的数据包要执行的指令集，要注意的是，若未指定任何action，数据包会被DROP     

| action指令 | 作用 |   
| --------- | -------- |     
| output:PORT | 数据包从端口PORT发出，PORT为openflow端口号，若PORT为数据包进入端口，则不执行此action |    
| normal | 数据包交由OVS自身的转发规则完成转发，不再匹配任何openflow flow |    
| all | 数据包从网桥上所有端口发出，除了其进入端口 |     
| drop | 丢弃数据包，当然，drop之后不能再跟其它action |    
| mod_vlan_vid:vlan_vid | 添加或修改数据包中的VLAN tag为此处指定的tag |     
| strip_vlan | 移除数据包中的VLAN tag，如果有的话 |       
| mod_dl_src:mac mod_dl_dst:mac | 修改源或目的MAC地址 |    
| mod_nw_src:ip mod_nw_dst:ip | 修改源或者目的ip地址 |    
mod_tp_src:port mod_tp_dst:port | 修改TCP或UDP数据包的源或目的端口号(注意不是openflow端口号) |     
| resubmit([port],[table]) | 若port指定,替换数据包in_port字段,并重新匹配；若table指定，提交数据包到指定table，并匹配 |    
{:.mbtablestyle}      

同样，还有很多其它的action未列出，这里需要注意NORMAL这一action。上面新建的`br-test`中那条默认flow其action部分就是NORMAL，这也说明新建的OVS网桥默认是被当做一个普通的二层交换机            

# 添加flow实例                 

这里用几个实例说明下添加flow的命令以及验证方法。为了后续测试方便，可以使用文件格式磁盘快速新建两台虚拟机test1和test2，桥接到上面新建的网桥`br-test`上，然后通过抓包来验证我们添加的flow是否生效，虚拟机信息如下       

| 虚拟机 | interface | OpenFlow端口号 | IP地址 |    
| --------- | -------- | ----- | -------- | 
| test1 | vnet12 | 7 | 172.16.1.11 |
| test2 | vnet13 | 8 | 172.16.1.12 |
{:.mbtablestyle}  

test1的虚拟网卡为vnet12，vnet12挂载到OVS Port `vnet12`上，这个Port的OpenFlow端口号为7       
<br />     

**屏蔽广播包**     

添加一条flow：屏蔽进入`br-test`中的所有广播包            

``` shell
ovs-ofctl add-flow br-test "table=0, dl_dst=01:00:00:00:00:00/01:00:00:00:00:00, actions=drop"   
```

添加flow之后，在test1中ping一个不存在的IP，比如172.16.1.13，test1会发送arp广播包以获取172.16.1.13的MAC地址，arp广播包从test1网卡vnet12进入`br-test`后会匹配到此条flow，之后就被DROP，因此test2中抓不到广播包。 作为验证可以再删除这条flow          

``` shell
ovs-ofctl del-flows br-test "dl_src=01:00:00:00:00:00/01:00:00:00:00:00"
```

可以看到，删除之后test2中马上就可以收到test1发出的arp广播包         

**添加vlan tag**          

添加一条flow：对于从test1的数据包，添加vlan tag 11，然后再正常转发       

``` shell
#也就是从vnet12网卡进入    
ovs-ofctl add-flow br-test "in_port=7,dl_vlan=0xffff,actions=mod_vlan_vid:11,normal"

#test1中ping test2 IP
ping 172.16.1.12
```

之后在test2中抓包，抓到的数据包都带有`vlan 101`，test2无法replay是因为test2会丢弃带有vlan tag的数据包       

``` shell
tcpdump -i eth0 -e
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
08:00:47.338428 52:54:00:4d:db:39 (oui Unknown) > Broadcast, ethertype 802.1Q (0x8100), length 46: vlan 101, p 0, ethertype ARP, Request who-has test2 tell 172.16.1.11, length 28
```

**修改数据包源IP地址**            

添加一条flow：修改从test1进入的数据包的源IP地址为172.16.1.111,并转发到test2     

``` shell   
#先删除上一条添加的vlan flow
ovs-ofctl del-flows br-test "vlan_tci=0x0000"

#test1对应vnet12网卡，vnet12网卡挂载到OpenFlow端口7上
ovs-ofctl add-flow br-test "in_port=7,actions=mod_nw_src:172.16.1.111,output=8"
```

在test2中抓包可以清楚看到，源IP已变为172.16.1.111     

``` shell
#tcpdump -i eth0 -e
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
08:12:17.815841 52:54:00:4d:db:39 (oui Unknown) > 52:54:00:f3:bc:d5 (oui Unknown), ethertype IPv4 (0x0800), length 98: 172.16.1.111 > test2: ICMP echo request, id 3003, seq 3, length 64
08:12:17.820427 52:54:00:f3:bc:d5 (oui Unknown) > Broadcast, ethertype ARP (0x0806), length 42: Request who-has 172.16.1.111 tell test2, length 28
```

# Neutron实现的OpenFlow控制器           

对于新建的bridge，默认是没有控制器存在的。但OVS可以连接OpenFlow控制器配置为一个SDN交换机。OpenStack Neutron中实现了一个OpenFlow控制器来管理OVS，在每一个运行`neutron-openvswitch-agent`的计算节点上，Neutron默认都建立了一个本地控制器`Controller "tcp:127.0.0.1:6633"`，该节点上的所有Bridge `br-int/br-tun/br-ext`等都连接到此Controller上，相关配置参考`/etc/neutron/plugins/ml2/openvswitch_agent.ini`中`[OVS]`      

``` shell
cat /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
...
# Address to listen on for OpenFlow connections. Used only for 'native' driver.
# (IP address value)
#of_listen_address = 127.0.0.1

# Port to listen on for OpenFlow connections. Used only for 'native' driver.
# (port value)
# Minimum value: 0
# Maximum value: 65535
#of_listen_port = 6633
...
```   

比如在运行`neutron-openvswitch-agent`的计算节点中，可以看到`br-tun`连接了一个控制器`tcp:127.0.0.1:6633`，`is_connected: true`表示控制器处于连接状态。            

``` shell
ovs-vsctl show
a9fc1666-0bb4-48a6-8f5c-1c8b92431ef6
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port "vxlan-080058ca"
            Interface "vxlan-080058ca"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="8.0.88.204", out_key=flow, remote_ip="8.0.88.202"}
...
```        

当控制器处于连接状态时，OVS中的所有流表(当然是指OpenFlow flow)都由控制器下发和维护，这没问题。但如果OVS到控制器的连接中断了，OVS中的流表无法得到更新，此时OVS该如何处理呢，这就是`fail_mode: secure`配置的作用，这个参数决定了OVS在连接控制器异常时该如何操作，可选值为`standalone`,或`secure`      

-  standalone   

OVS每隔`inactivity_probe`秒尝试连接一次控制器，重试三次，三次仍失败之后，OVS会转变为一个普通的MAC地址学习交换机(上面提到的OVS工作模式)。但是OVS仍会在后台尝试连接Controller，一旦连接成功，就会重新转变为OpenFlow交换机，依靠流表完成转发    

- secure      

当Controller连接失败或没配置Controller时，OVS自身不会设置任何flow，它只会尝试重连Controller        

`fail_mode`的默认值为standalone，当配置有多个Controller时，只有所有的Controller都连接失败，此参数才会起作用     

下面是设置Controller命令     

``` shell
#获取br0网桥Controller 
ovs-vsctl get-controller br0
#设置OpenFlow控制器,控制器地址为192.168.1.10，端口为6633
ovs-vsctl set-controller br0 tcp:192.168.1.10:6633
#删除controller
ovs-vsctl del-controller br0
```       

关于OVS中vlan的使用，下篇文章会单独介绍(本文完)      


