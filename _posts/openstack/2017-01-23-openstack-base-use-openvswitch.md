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

## Open vSwitch介绍     

在过去，数据中心的服务器是直接连在硬件交换机上，后来VMware实现了服务器虚拟化技术，使虚拟服务器(VMs)能够连接在虚拟交换机上，借助这个虚拟交换机，可以为服务器上运行的VMs或容器提供逻辑的虚拟的以太网接口，这些逻辑接口都连接到虚拟交换机上，有三种比较流行的虚拟交换机: VMware virtual switch, Cisco Nexus 1000V,和Openv Switch     

Open vSwitch(OVS)是运行在虚拟化平台上的虚拟交换机，其支持OpenFlow协议，也支持gre/vxlan/IPsec等隧道技术。在OVS之前，基于Linux的虚拟化平台比如KVM或Xen上，缺少一个功能丰富的虚拟交换机，因此OVS迅速崛起并开始在Xen中流行起来，并且应用于越来越多的开源项目，比如openstack neutron中的网络解决方案   

在虚拟交换机的控制器或管理工具方面，一些商业产品都集成有控制器或管理工具，比如Cisco 1000V的`Virtual Supervisor Manager(VSM)`，VMware的分布式交换机中的`vCenter`。而OVS需要借助第三方控制器或管理工具进行管理。例如OVS提供了对OpenFlow 协议的支持，我们就可以使用任何支持OpenFlow协议的控制器对OVS进行远程管理。OpenStack Neutron中的ML2插件也实现了对OVS的管理。但这并不意味着必须要有一个控制器对OVS进行管理，在一些简单场景下，没有控制器，OVS本身就可以作为一个基于MAC地址学习实现转发功能的二层交换机，就像普通物理交换机那样    

在基于Linux系统上，应用最广泛的还是系统自带的虚拟交换机Linux Bridge，它是一个单纯的基于MAC地址学习的二层交换机，简单高效，但同时缺乏一些高级特性，比如VLAN tag,QOS,流表之类，而且在隧道协议支持方面，Linux Bridge只支持vxlan，OVS是gre/vxlan/IPsec等，这也决定了OVS更适用于实现SDN技术   

OVS有以下特点   
  
- 精细的ACL和QoS策略   
- 可以使用OpenFlow和OVSDB协议进行控制     
- Port绑定，隧道协议gre,vxlan,IPsec支持   
- 适用于Xen，KVM，VirtualBox等虚拟化平台   
- 支持标准的802.1Q VLAN    
- 支持组播功能   
- NetFlow, IPFIX, sFlow, SPAN/RSPAN等协议的支持   
- 内核中基于通配符的流匹配策略    
- ...    

文章使用环境`centos7  openvswitch 2.5  OpenFlow 1.4`   

## OVS架构    

先看下OVS整体架构          

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-arch.png)   

用户空间主要组件有数据库服务ovsdb-server，守护进程ovs-vswitchd，以及kernel中的datapath内核模块。最上面的Controller表示OVS的控制器，控制器与OVS是通过IP协议进行连接，也就是说控制器不一定位于OVS主机上，下面分别介绍下各组件   

**ovsdb-server**    

`ovsdb-server`是实现OVS的数据库服务进程,存放OVS所有配置信息，像网桥名,Port,interface,tunnel等这些，OVS主进程`ovs-vswitchd`需要连接此数据库         

``` shell 
root     22166 22165  0 Jan17 ?        00:02:32 ovsdb-server /etc/openvswitch/conf.db -vconsole:emer -vsyslog:err -vfile:info --remote=punix:/var/run/openvswitch/db.sock --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --no-chdir --log-file=/var/log/openvswitch/ovsdb-server.log --pidfile=/var/run/openvswitch/ovsdb-server.pid --detach --monitor
```

- `/etc/openvswitch/conf.db`是数据库文件存放位置，这种文件形式存储，保证了服务器重启不会影响其配置信息，可以使用`ovsdb-tool create`命令创建并初始化此数据库文件    
- `--remote=punix:/var/run/openvswitch/db.sock` 实现了一个Unix sockets连接方式，`ovs-vswitchd`或其它工具通过socket连接管理此db    
- `/var/log/openvswitch/ovsdb-server.log`是日志记录      

**ovs-vswitchd**   

`ovs-vswitchd`是OVS主进程，通过与ovsdb(`ovsdb-server`)交互实现OVS的所有功能特性，像MAC地址学习,Port bonding,802.1Q VLAN支持,sFlow,连接外部OpenFlow controller等等。可以查看其进程信息          

``` shell
# ps -ef |grep ovs-vs
root     22176 22175  0 Jan17 ?        00:16:56 ovs-vswitchd unix:/var/run/openvswitch/db.sock -vconsole:emer -vsyslog:err -vfile:info --mlockall --no-chdir --log-file=/var/log/openvswitch/ovs-vswitchd.log --pidfile=/var/run/openvswitch/ovs-vswitchd.pid --detach --monitor
...
```   

- `ovs-vswitchd`管理此主机上所有OVS实现的虚拟网桥，其通过socket`/var/run/openvswitch/db.sock`连接ovsdb，并且有自己的日志文件`/var/log/openvswitch/ovs-vswitchd.log`      
- `ovs-vswitchd`在启动时会读取ovsdb中配置信息，然后配置内核中的`datapaths`和所有的虚拟网桥，当ovsdb中的配置信息改变时(例如使用ovs-vsctl工具)，`ovs-vswitchd`也会自动更新其配置以保持与数据库同步    
- 通过`netlink`与内核模块`datapath`通信     

`ovs-vswitchd`在运行时需要加载datapath内核模块，datapath内核模块信息如下        

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

**OpenFlow**   

OpenFlow是一种用于管理交换机流表的协议，OpenFlow在OVS中的地位可以参考上面架构图，它是Controller和ovs-vswitched间的通信协议。需要注意的是，OpenFlow是一个独立的完整的流表协议，不依赖于OVS，OVS只是提供了对OpenFlow协议的支持，有了支持，我们可以使用支持OpenFlow的控制器来管理OVS中的流表，OpenFlow不仅仅支持虚拟交换机，某些硬件交换机也支持OpenFlow协议        

OpenFlow中的流表定义了交换机端口之间数据包的交换规则，支持OpenFlow协议的交换机可以包括一个或者多个流表(Table)，每个流表包括多个流表项(Flow entrys)，每条流表项中的条目包含：数据包头的信息、匹配成功后要执行的指令和统计信息。当数据包进入OVS后，OVS会将数据包和Tables中的流表项进行匹配以决定此数据包是被转发/修改或是DROP。     

![openflow](/images/openstack/openstack-use-openvswitch/openvswitch-openflow-match.png)    

流表是基于Flow规则的交换机进行转发策略控制的核心数据结构，之所以说是基于Flow规则的交换机是因为，OVS也可以不使用Flow规则来控制数据包的转发，而仅仅依靠自身的MAC地址学习完成转发，此时不需要连接外部Flow控制器`Controller`，因此，总结一下OVS可以有两种转发策略     
  
- 基于Flow规则的转发   

此时需要为OVS设置外部控制器(Controller)。控制器下发流表规则到OVS，OVS按照下发的流表规则完成数据转发。当有新的MAC地址加入，或者MAC地址从一个Port移到另一个Port上时，控制器会更新流表规则以匹配此改变，可见外部控制器决定着OVS中的流表规则   

还有一些其它话题，比如某条流表项中的动作action为`normal`，或者外部控制器由于网络故障无法连接时，这些话题到后面再讨论  

- 基于MAC地址学习的虚拟交换机    

不需要连接外部控制器，依靠MAC地址学习完成转发，比如第一个数据包进入OVS时，由于之前没有任何数据包进入，OVS无法知道数据包目的MAC地址应该从哪个Port发出，此时它就会把数据包转发到除了进入Port之外的所有Port，就像Linux Bridge那样。这种情况下OVS依然可以为Port设置Vlan，但Linux Bridge不支持设置Vlan     

OpenStack Neutron中使用Flow来管理OVS，其为OVS设置了一个控制器Controller，位于本地主机上，本地主机上所有Bridge(`br-int/br-tun/br-ext`)都连接到此Controller上，相关配置参考`/etc/neutron/plugins/ml2/openvswitch_agent.ini`中`[OVS]`   

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

查看网桥`br-tun`上连接的控制器   

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

**几个管理工具的作用及区别**   

上面介绍了用户空间两个进程，这里说下对其进程操作的命令行工具           

`ovs-vsctl`是最常用的工具，比如增删Port/Bridge，设置Interface/Vlan tag/Controller等，其直接读取或更新ovsdb，之后`ovs-vswitchd`会自动读取并应用更改，然后`ovs-vsctl`命令才会结束返回         

- 添加网桥br0 `ovs-vsctl add-br br0`
- 列出所有网桥 `ovs-vsctl list-br`
- 添加一个Port到网桥br0 `ovs-vsctl add-port br0 p1`
- 查看网桥br0上所有端口 `ovs-vsctl list-ports br0`
- 获取OVS控制器地址 `ovs-vsctl get-controller <bridge>`
- 删除网桥br0 `ovs-vsctl del-br br0`

`ovsdb-tool`是一个专门管理OVS数据库文件的工具，它不直接与`ovsdb-server`进程通信    

- 可以使用此工具创建并初始化database文件 `ovsdb-tool create [db] [schema]` ，具体见`man ovsdb-tool`    
- 可以查看数据库更改记录，具体到操作命令 `ovsdb-tool show-log -m`    

`ovsdb-client`是`ovsdb-server`进程的命令行工具，主要是从正在运行的`ovsdb-server`中查询信息      

- 列出主机上的所有databases `ovsdb-client list-dbs` ，默认只有一个库`Open_vSwitch`   
- 获取指定数据库的schema信息 `ovsdb-client get-schema [DATABASE]` ，JSON格式输出   
- 列出指定数据库的所有表 `ovsdb-client list-tables [DATABASE]`    
- dump指定数据库所有数据 `ovsdb-client dump [DATABASE] [TABLE]` 默认dump所有table数据，如果指定table，只dump指定table数据    
- 监控指定数据库中的指定表， `ovsdb-client monitor DATABASE TABLE`   

`ovs-vsctl`是一个综合的配置管理工具，提供了比`ovsdb-client`更高一级的封装，`ovsdb-client`倾向于从数据库中查询某些信息，而`ovsdb-tool`是维护数据库文件工具    

**Kernel Datapath**   

这里介绍下OVS内核空间


更详细的图   

![ovs1](/images/openstack/openstack-use-openvswitch/openvswitch-details.png)  

OVS中的


















https://www.sdxcentral.com/cloud/open-source/definitions/what-is-open-vswitch/
http://openvswitch.org/features/
