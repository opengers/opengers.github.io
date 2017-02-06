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

* TOC
{:toc} 

# Open vSwitch介绍     

在过去，数据中心的服务器是直接连在硬件交换机上，后来VMware实现了服务器虚拟化技术，使虚拟服务器(VMs)能够连接在虚拟交换机上，借助这个虚拟交换机，可以为服务器上运行的VMs或容器提供逻辑的虚拟的以太网接口，这些逻辑接口都连接到虚拟交换机上，有三种比较流行的虚拟交换机: VMware virtual switch, Cisco Nexus 1000V,和Openv Switch     

Open vSwitch(OVS)是运行在虚拟化平台上的虚拟交换机，其支持OpenFlow协议，也支持gre/vxlan/IPsec等隧道技术。在OVS之前，基于Linux的虚拟化平台比如KVM或Xen上，缺少一个功能丰富的虚拟交换机，因此OVS迅速崛起并开始在Xen中流行起来，并且应用于越来越多的开源项目，比如openstack neutron中的网络解决方案   

在虚拟交换机的控制器或管理工具方面，一些商业产品都集成有控制器或管理工具，比如Cisco 1000V的`Virtual Supervisor Manager(VSM)`，VMware的分布式交换机中的`vCenter`。而OVS需要借助第三方控制器或管理工具进行管理。例如OVS支持OpenFlow 协议，我们就可以使用任何支持OpenFlow协议的控制器来对OVS进行远程管理。OpenStack Neutron中的ML2插件也能够实现对OVS的管理。但这并不意味着OVS必须要有一个控制器才能工作，在一些简单场景下，没有控制器，OVS本身就可以作为一个基于MAC地址学习实现转发功能的二层交换机，就像普通物理交换机那样    

在基于Linux内核的系统上，应用最广泛的还是系统自带的虚拟交换机`Linux Bridge`，它是一个单纯的基于MAC地址学习的二层交换机，简单高效，但同时缺乏一些高级特性，比如OpenFlow,VLAN tag,QOS,ACL,Flow等，而且在隧道协议支持上，Linux Bridge只支持vxlan，OVS支持gre/vxlan/IPsec等，这也决定了OVS更适用于实现SDN技术     

OVS支持以下[features](http://openvswitch.org/features/)         

- 支持NetFlow, IPFIX, sFlow, SPAN/RSPAN等流量监控协议     
- 精细的ACL和QoS策略         
- 可以使用OpenFlow和OVSDB协议进行集中控制     
- Port bonding，LACP，tunneling(vxlan/gre/Ipsec)  
- 适用于Xen，KVM，VirtualBox等hypervisors           
- 支持标准的802.1Q VLAN协议   
- 基于VM interface的流量管理策略        
- 支持组播功能       
- flow-caching engine(datapath模块)    

文章使用环境      

``` shell
centos7
openvswitch 2.5
OpenFlow 1.4`
```  

# OVS概念    

先看一个OpenStack neutron+vxlan部署模式下网络节点OVS网桥    

``` shell
# ovs-vsctl show
e44abab7-2f65-4efd-ab52-36e92d9f0200
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-ext
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-ext
            Interface br-ext
                type: internal
        Port "eth1"
            Interface "eth1"
        Port phy-br-ext
            Interface phy-br-ext
                type: patch
                options: {peer=int-br-ext}
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-080058ca"
            Interface "vxlan-080058ca"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="8.0.88.201", out_key=flow, remote_ip="8.0.88.202"}
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port "qr-11591618-c4"
            tag: 3
            Interface "qr-11591618-c4"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ext
            Interface int-br-ext
                type: patch
                options: {peer=phy-br-ext}
```

这部分说下OVS中重要概念   

#### Bridge    

Bridge代表一个以太网交换机(Switch)，一个主机中可以创建一个或者多个Bridge设备。其功能是根据一定规则，把从端口收到的数据包转发到另一个或多个端口。上图中有两个Bridge，`br-tun`和`br-int` 

#### Port     

端口Port与物理交换机的端口概念类似，Port是Bridge上创建的一个虚拟端口，每个Port都隶属于一个Bridge。Port有以下几种类型  

- Normal   

可以把操作系统中已有的网卡(物理网卡em1/eth0,或虚拟机的虚拟网卡tapxxx)绑定到ovs上，ovs会生成一个同名普通端口处理这块网卡进出的数据包。此时端口类型为Normal。  

``` shell
ovs-vsctl add-port br-ext eth1
```

把物理网卡`eth1`绑定到OVS网桥`br-ext`上，OVS会自动创建同名Port `eth1`。      

- Internal     

下面创建一个网桥br0，并添加一个端口类型为Internal的Port `p10`   

``` shell 
ovs-vsctl add-br br0   
ovs-vsctl add-port br0 p10 -- set Interface p10 type=internal
```

端口p10类型指定为internal时，ovs会创建一块同名虚拟网卡p10，端口收到的所有数据包都会交给该网卡，发出的包会通过该端口交给ovs。当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port。  

当Port为Internal类型时，OVS会自动创建一个同名网卡挂载到新创建的Port上，而Normal类型是把一个系统中已有的网卡添加到OVS中           

- Patch    

当主机中有多个ovs网桥时，可以使用Patch Port把两个网桥连起来。Patch Port总是成对出现，分别连接在两个网桥上，从一个Patch Port收到的数据包会被转发到另一个Patch Port，类似于Linux系统中的`veth`    

比如，网桥`br-ext`中的Port `phy-br-ext`与`br-int`中的Port `int-br-ext`是一对Patch Port   

- Tunnel   

Port为tunnel端口，有两种类型`gre`或`vxlan`，支持使用gre或vxlan等隧道技术与位于网络上其他位置的远程端口通讯。上面网桥`br-tun`中`Port "vxlan-080058ca"`就是一个`vxlan`类型tunnel端口       
 
#### Interface      

接口是ovs与外部交换数据包的组件。一个接口就是操作系统中的一块网卡，这块网卡可能是ovs生成的虚拟网卡(Internal)，也可能是物理网卡挂载在ovs上，也可能是操作系统的虚拟网卡(TUN/TAP)挂载在ovs上。   

`Interface`是系统中一块网卡(物理或虚拟)，`Port`是OVS网桥上一个虚拟端口，Interface挂载在Port上。    

#### Controller       

OpenFlow控制器。OVS可以同时接受一个或者多个OpenFlow控制器的管理。  

#### datapath       

OVS内核模块，负责执行数据交换，根据其流表缓存或向用户空间`ovs-vswitchd`查询流表对数据包执行匹配到的动作(从另一个Port发出/DROP/添加Vlan tag)       

#### 流表        

支持OpenFlow协议的交换机应该包括一个或者多个流表，流表中的条目包含：数据包头的信息、匹配成功后要执行的指令和统计信息。 当数据包进入OVS后，会将数据包和流表中的流表项进行匹配，如果发现了匹配的流表项，则执行该流表项中的指令集(actions)。   

# OVS架构    

先看下OVS整体架构，用户空间主要组件有数据库服务ovsdb-server和守护进程ovs-vswitchd。kernel中是datapath内核模块。最上面的Controller表示OVS的控制器，控制器与OVS是通过OpenFlow协议进行连接，但控制器不一定位于OVS主机上，下面分别介绍图中各组件       

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-arch.png)   

## ovsdb-server      

`ovsdb-server`是实现OVS的数据库服务进程,存放OVS所有配置信息，像网桥名,Port,interface,tunnel等这些，OVS主进程`ovs-vswitchd`需要连接此数据库，下面是`ovsdb-server`进程            

``` shell 
ps -ef |grep ovsdb-server
root     22166 22165  0 Jan17 ?        00:02:32 ovsdb-server /etc/openvswitch/conf.db -vconsole:emer -vsyslog:err -vfile:info --remote=punix:/var/run/openvswitch/db.sock --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --no-chdir --log-file=/var/log/openvswitch/ovsdb-server.log --pidfile=/var/run/openvswitch/ovsdb-server.pid --detach --monitor
```

`/etc/openvswitch/conf.db`是数据库文件存放位置，文件形式存储保证了服务器重启不会影响其配置信息，`ovsdb-server`需要文件才能启动，可以使用`ovsdb-tool create`命令创建并初始化此数据库文件        
`--remote=punix:/var/run/openvswitch/db.sock` 实现了一个Unix sockets连接，OVS主进程`ovs-vswitchd`或其它命令工具(ovsdb-client)通过此socket连接管理ovsdb       
`/var/log/openvswitch/ovsdb-server.log`是日志记录        

## ovs-vswitchd          

`ovs-vswitchd`是OVS主进程，它管理主机上所有OVS switches，它通过socket`/var/run/openvswitch/db.sock`连接ovsdb，从而与ovsdb数据库交互实现像增删/Bridge/Port/Interface/VLan tag等功能，其通过OpenFlow协议连接OpenFlow控制器。也有自己的日志文件`/var/log/openvswitch/ovs-vswitchd.log`    

``` shell
# ps -ef |grep ovs-vs
root     22176 22175  0 Jan17 ?        00:16:56 ovs-vswitchd unix:/var/run/openvswitch/db.sock -vconsole:emer -vsyslog:err -vfile:info --mlockall --no-chdir --log-file=/var/log/openvswitch/ovs-vswitchd.log --pidfile=/var/run/openvswitch/ovs-vswitchd.pid --detach --monitor
```   
  
`ovs-vswitchd`在启动时会读取ovsdb中配置信息，然后配置内核中的`datapaths`和所有OVS switches，当ovsdb中的配置信息改变时(例如使用ovs-vsctl工具)，`ovs-vswitchd`也会自动更新其配置以保持与数据库同步   

`ovs-vswitchd`需要加载`datapath`内核模块才能正常运行，其通过`netlink`与`datapath`通信以便对其进行管理，比如初始化`datapath`或缓存flow匹配结果到`datapath`，因此我们不必再使用`ovs-dpctl`去手动操作`datapath`，但`ovs-dpctl`仍可用于调试场合      

在OVS中，`ovs-vswitchd`从OpenFlow控制器获取流表规则，然后把从`datapath`中收到的数据包在流表中进行匹配，找到符合某条flow规则的数据包需要应用的actions，然后缓存这些actions到`datapath`模块，对于`datapath`来说，其并不知道OpenFlow的存在，datapath内核模块信息如下      

``` shell
# modinfo openvswitch
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/net/openvswitch/openvswitch.ko
license:        GPL
description:    Open vSwitch switching datapath
rhelversion:    7.2
srcversion:     F75F2B83324DCC665887FD5
depends:        libcrc32c
intree:         Y
...
```

## OpenFlow && Controller    

OpenFlow是一种用于管理交换机流表的协议，OpenFlow在OVS中的地位可以参考上面架构图，它是Controller和ovs-vswitched间的通信协议。需要注意的是，OpenFlow是一个独立的完整的流表协议，不依赖于OVS，OVS只是提供了对OpenFlow协议的支持，有了支持，我们可以使用任何支持OpenFlow的控制器来管理OVS中的流表，OpenFlow不仅仅支持虚拟交换机，某些硬件交换机也支持OpenFlow协议 

Controller指OpenFlow控制器。OpenFlow控制器可以通过OpenFlow协议连接到任何支持OpenFlow 的交换机，控制器通过向交换机下发流表规则来控制数据流向。OVS可以同时接受一个或者多个OpenFlow控制器的管理。在没有配置 OpenFlow控制器的模式下，依然可以使用OVS提供的`ovs-ofctl`命令通过OpenFlow协议去连接OVS，创建、修改或删除OVS 中的流表项，也能够对OVS的运行状况进行动态监控。     

`ovs-ofctl`是一个监控和管理OpenFlow交换机的命令行工具，它支持任何使用OpenFlow协议的交换机，不仅仅是OVS    

OpenFlow的介绍上说的`OpenFlow协议实现了控制层面和转发层面分离`，控制层面就是指这里的OpenFlow控制器，分离就是说控制器负责控制转发规则，OVS负责转发的具体实现，他们是分离的两个软件，但是可以通过OpenFLow远程连接，不需要位于同一台主机上    

OpenFlow中的流表(Tables)定义了交换机端口之间数据包的交换规则，以OVS为例，OVS交换机中可以有一个或者多个流表，每个流表包括多个流表项(Flow entrys)，每条流表项中的条目包含：数据包头的信息、匹配成功后要执行的指令和统计信息。当数据包进入OVS后，OVS会将数据包和Tables中的流表项进行匹配以决定此数据包是被转发/修改或是DROP。     

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)    

流表是OpenFlow交换机进行转发策略控制的核心数据结构，之所以说是OpenFlow交换机是因为，OVS也可以不使用OpenFlow来控制数据包的转发，而仅仅依靠自身的MAC地址学习完成转发，此时不需要连接OpenFlow控制器，上面讨论的都是在OVS使用OpenFlow的情况下，这里总结一下OVS的转发策略方案        
  
**使用OpenFlow控制器的转发策略**                

 + 此时OVS需要一个OpenFlow控制器来下发流表规则到OVS，OVS按照下发的流表规则完成数据转发。当有新的MAC地址加入(新建VM)，或者MAC地址从一个Port移到另一个Port上时(虚拟机迁移)，控制器会更新流表规则以匹配此改变，可见外部控制器决定着OVS中的流表规则，需要注意的是可以是同一个控制器管理多台计算节点上的OVS   
 
 + 关于此转发策略，可以参考openstack OVS+Vxlan网络部署模式下的`br-tun`网桥中的flow tables        
 
  + 还有一些其它话题，比如当某条流表项中的执行动作为`normal`时，OpenFlow会把匹配到这条规则的数据包丢给OVS自身处理，这些数据包就不再匹配其它的流表规则。还有当外部控制器由于网络故障无法连接时， 这些情况到后面介绍流表规则时再讨论       
 
**基于MAC地址学习的转发策略**               

 + 在没有OpenFlow控制器存在的情况下，OVS依靠MAC地址学习完成转发，考虑第一个数据包进入OVS的情况，由于之前没有任何数据包进入，也没了控制器的存在，OVS无法知道第一个数据包应该从哪个端口发出，此时只能依靠学习喽，OVS会把数据包转发到除了进入Port之外的所有Port，然后根据应答数据包的进入Port来学习MAC地址对应的Port，就像Linux Bridge那样。这种情况下OVS依然可以为Port设置Vlan tag，但Linux Bridge不支持设置Vlan    

**OpenFlow控制器+MAC地址学习**    

 + 上面说的两种转发策略方案并不是对立的，在使用OpenFlow控制器的转发策略情况下，如果某条流表项中的执行动作`actions`为`normal`时，控制器会立即把此数据包交给OVS自身处理，之后此数据包就根据MAC地址学习完成转发，不再受流表规则控制，openstack OVS+Vxlan网络部署模式下的网桥`br-int`就是这种情况     

**手动建立流表规则**             

 + 前面提到`ovs-ofctl`工具可以通过OpenFlow协议去连接OVS，创建、修改或删除OVS网桥中的流表项，那我们就自己`add-br`一个网桥，然后建立一些流表项观察数据包转发规则，测试或学习OpenFlow协议时可以这么干            

## Neutron实现的OpenFLow控制器        

OpenStack Neutron中实现了一个OpenFlow控制器，来管理OVS和其上的VMs，在每一个运行`neutron-openvswitch-agent`的计算节点上，Neutron默认都建立了一个本地控制器`Controller "tcp:127.0.0.1:6633"`，该节点上的所有Bridge `br-int/br-tun/br-ext`等都连接到此Controller上，相关配置参考`/etc/neutron/plugins/ml2/openvswitch_agent.ini`中`[OVS]`      

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

运行`neutron-openvswitch-agent`的计算节点中网桥`br-tun`上连接的控制器       

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

## OVS中管理工具的使用及区别    

上面介绍了OVS用户空间进程以及控制器和OpenFlow协议，这里说下相关的命令行工具的使用及区别               

**ovs-vsctl**   

`ovs-vsctl`是一个管理或配置`ovs-vswitchd`的高级命令行工具，高级是说其操作对用户友好，封装了对数据库的操作细节。它是最常用的命令，除了管理流表功能外，其它大部分操作比如Bridge/Port/Interface/Controller/Database/Vlan等都可以完成               

``` shell
#添加网桥br0
ovs-vsctl add-br br0
#列出所有网桥 
ovs-vsctl list-br
#添加一个Port p1到网桥br0
ovs-vsctl add-port br0 p1
#查看网桥br0上所有Port   
ovs-vsctl list-ports br0
#获取br0网桥的OpenFlow控制器地址，没有控制器则返回空 
ovs-vsctl get-controller br0
#删除网桥br0
ovs-vsctl del-br br0
#设置端口p1的vlan tag为100
ovs-vsctl set Port p1 tag=100
#设置Port p0类型为internal
ovs-vsctl set Interface p0 type=internal
#添加vlan10端口，并设置vlan tag为10，Port类型为Internal
ovs-vsctl add-port br0 vlan10 tag=10 -- set Interface vlan10 type=internal
#添加隧道端口gre0，类型为gre，远端IP为1.2.3.4
ovs-vsctl add-port br0 gre0 -- set Interface gre0 type=gre options:remote_ip=1.2.3.4
```

**ovsdb-tool**    

`ovsdb-tool`是一个专门管理OVS数据库文件的工具，不常用，它不直接与`ovsdb-server`进程通信       

``` shell
#可以使用此工具创建并初始化database文件
ovsdb-tool create [db] [schema]
#可以使用ovsdb-client get-schema [database]获取某个数据库的schema(json格式)
#可以查看数据库更改记录，具体到操作命令，这个比较有用   
ovsdb-tool show-log -m   
record 48: 2017-01-07 03:34:15.147 "ovs-vsctl: ovs-vsctl --timeout=5 -- --if-exists del-port tapcea211ae-10"
        table Interface row "tapcea211ae-10" (151f66b6):
                delete row
        table Port row "tapcea211ae-10" (cc9898cd):
                delete row
        table Bridge row "br-int" (fddd5e27):
        table Open_vSwitch row a9fc1666 (a9fc1666):

record 49: 2017-01-07 04:18:23.671 "ovs-vsctl: ovs-vsctl --timeout=5 -- --if-exists del-port tap5b4345ea-d5 -- add-port br-int tap5b4345ea-d5 -- set Interface tap5b4345ea-d5 "external-ids:attached-mac=\"fa:16:3e:50:1b:5b\"" -- set Interface tap5b4345ea-d5 "external-ids:iface-id=\"5b4345ea-d5ea-4285-be99-0e4cadf1600a\"" -- set Interface tap5b4345ea-d5 "external-ids:vm-id=\"0aa2d71e-9b41-4c88-9038-e4d042b6502a\"" -- set Interface tap5b4345ea-d5 external-ids:iface-status=active"
        table Port insert row "tap5b4345ea-d5" (4befd532):
        table Interface insert row "tap5b4345ea-d5" (b8a5e830):
        table Bridge row "br-int" (fddd5e27):
        table Open_vSwitch row a9fc1666 (a9fc1666):
...
```

**ovsdb-client**   

`ovsdb-client`是`ovsdb-server`进程的命令行工具，主要是从正在运行的`ovsdb-server`中查询信息，操作的是数据库相关   

``` shell
#列出主机上的所有databases，默认只有一个库Open_vSwitch
ovsdb-client list-dbs
#获取指定数据库的schema信息
ovsdb-client get-schema [DATABASE]
#列出指定数据库的所有表
ovsdb-client list-tables [DATABASE]
#dump指定数据库所有数据,默认dump所有table数据，如果指定table，只dump指定table数据  
ovsdb-client dump [DATABASE] [TABLE]
#监控指定数据库中的指定表记录改变  
ovsdb-client monitor DATABASE TABLE
```

`ovs-vsctl`是一个综合的配置管理工具，`ovsdb-client`倾向于从数据库中查询某些信息，而`ovsdb-tool`是维护数据库文件工具   

## Kernel Datapath           

下面讨论场景是OVS作为一个OpenFlow交换机    

datapath是一个Linux内核模块，它负责执行数据交换，同时也缓存flow的actions以提高OVS数据交换性能，关于datapath，[The Design and Implementation of Open vSwitch](http://benpfaff.org/papers/ovs.pdf)中有描述     

><small>The datapath module in the kernel receives the packets first, from a physical NIC or a VM’s virtual NIC. Either ovs-vswitchd has instructed the datapath how to handle packets of this type, or it has not. In the former case, the datapath module simply follows the instructions, called actions, given by ovs-vswitchd, which list physical ports or tunnels on which to transmit the packet. Actions may also specify packet modifications, packet sampling, or instructions to drop the packet. In the other case, where the datapath has not been told what to do with the packet, it delivers it to ovs-vswitchd. In userspace, ovs-vswitchd determines how the packet should be handled, then it passes the packet back to the datapath with the desired handling. Usually, ovs-vswitchd also tells the datapath to cache the actions, for handling similar future packets. </small>   

为了说明datapath，来看一张更详细的架构图，图中的大部分组件上面都有提到      

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-details.png)   

OVS中的两个组件`ovs-vswitchd`和`datapath`决定了数据包的转发，首先，`datapath`内核模块收到进入数据包(物理网卡或虚拟网卡)，然后查找其缓存，若缓存中有匹配此类数据包所对应的actions，则执行其actions，否则`datapath`会把该数据包送入用户空间由`ovs-vswitchd`负责在其流表项中查询(图1中的First Packet)，`ovs-vswitchd`查询flow tables后把此数据包连带actions返回给`datapath`并缓存此actions到`datapath`中，这样后续进入的同类型的数据包(图1中的Subsequent Packets)因为缓存匹配到会被`datapath`直接处理，不用再次进入用户空间，文章上面在介绍`ovs-vswitchd` 部分也有提到      

`datapath`专注于数据交换，它不需要知道OpenFlow的存在。与OpenFlow打交道的是`ovs-vswitchd`，`ovs-vswitchd`存储所有Flow规则供`datapath`查询和缓存。`datapath`根据其缓存或由`ovs-vswitchd`查询负责执行数据交换          

文章地址http://www.isjian.com/openstack/openstack-base-use-openvswitch/     

参考文章     

><small>
https://www.sdxcentral.com/cloud/open-source/definitions/what-is-open-vswitch/     
http://openvswitch.org/features/      
https://www.ibm.com/developerworks/cn/cloud/library/1401_zhaoyi_openswitch/    
http://openvswitch.org/slides/OpenStack-131107.pdf      
http://horms.net/projects/openvswitch/2010-10/openvswitch.en.pdf   
http://benpfaff.org/papers/ovs.pdf   
https://networkheresy.com/category/open-vswitch/</small>     

