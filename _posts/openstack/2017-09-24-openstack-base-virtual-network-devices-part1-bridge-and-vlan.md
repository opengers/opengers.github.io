---
title: openstack底层技术-各种虚拟网络设备之一(Bridge,VLAN)
author: opengers
layout: post
permalink: /openstack/openstack-base-virtual-network-devices-part1-bridge-and-vlan/
categories: openstack
tags:
  - openstack
  - tun/tap
  - veth
---

* TOC
{:toc}    

><small>IBM网站上有一篇高质量文章[Linux 上的基础网络设备详解](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/)。本文会参考文章部分内容，文中主要侧重于介绍OpenStack是如何使用这些虚拟网络设备构建出灵活的云网络，这些网络设备包括Bridge，VLAN，tun/tap, veth，vxlan/gre。本文先介绍Bridge和VLAN相关，其它在下一篇part2中介绍</small>        

OpenStack一般分为计算，存储，网络三部分。考虑构建一个灵活的可扩展的云网络环境，而物理网络架构一般是固定和难于扩展的，因此虚拟网络设备将更有优势。Linux平台上实现了各种不同功能的虚拟网络设备，包括`Bridge,Vlan,tun/tap,veth pair,vxlan/gre，...`，这些虚拟设备就像一个个积木块一样，被OpenStack组合用于构建虚拟网络。 还有火热的Docker，其技术实现脱胎于Linux平台上的`namspace`,以及更早的`chroot`。    

文中会牵涉虚拟机，所以用名词"主机"明确表示一台物理机，"接口"指挂载到网桥上的网络设备，环境如下：             

``` shell
CentOS Linux release 7.3.1611 (Core) 
Linux controller 3.10.0-514.16.1.el7.x86_64 #1 SMP Wed Apr 12 15:04:24 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
OpenStack社区版 Newton
```   

# Linux Bridge   

内核模块`bridge`     

``` shell
[root@controller ~]# modinfo bridge
filename:       /lib/modules/3.10.0-514.16.1.el7.x86_64/kernel/net/bridge/bridge.ko
```

Bridge是Linux上工作在内核协议栈二层的虚拟交换机，它可以绑定若干个网络设备(em1,eth0,tap,..)，这些网络设备被虚拟为接口连接到Bridge上，从接口收到的数据包都被直接送到Bridge中(bridge内核模块)，Bridge会进行一个类似物理交换机的查MAC端口映射表，转发，更新MAC端口映射表这样的处理逻辑。从而数据包可以被转发到另一个接口/丢弃/广播/发往上层协议栈。由此Bridge实现了数据交换的功能。如果使用`tcpdump`在Bridge接口上抓包，是可以抓到桥上所有接口进出的包            

跟物理交换机不同的是，运行Bridge的是一个Linux主机，Linux主机本身也需要IP地址与其它设备通信。但被添加到Bridge上的接口设备是不能配置IP地址的，其接口设备被设置为接收二层数据包，对路由系统不可见。不过Bridge本身可以设置IP地址，可以认为当使用`brctl addbr br0`新建一个`br0`网桥时，系统自动创建了一个同名的隐藏`br0`网络设备，因此我们给`br0`设置IP地址，Bridge收到目的地址为此IP的数据包会直接将数据包发往主机协议栈上层(下图)，也即主机收到了发往自身的数据包。             

![bridge](/images/openstack/openstack-virtual-devices/bridge.png)  

上图网桥`br0`有`vnet0`,`vnet1`两个接口，他们工作在链路层(Link Layer)，来看数据从`vnet0`发往`vnet1`所连接的vm这一过程，首先`vnet0`把收到的二层数据帧发给网桥`br0`，经过`bridging decision`以及netfilter规则(若设置有防火墙)，数据包最后从`vnet1`接口发出，此时vm的eth0网卡收到数据包。`bridging decision`中Bridge对数据包的处理，有以下几种：     

- 包目的MAC为Bridge本身MAC地址，收到发往主机自身的数据包，交给上层协议栈(图中蓝色向上箭头)       
- 广播包，转发到Bridge上的所有接口      
- 单播&&存在于MAC端口映射表，查表直接转发到对应接口(vnet1)          
- 单播&&不存在于MAC端口映射表，泛洪到Bridge连接的所有接口           
- 数据包目的地址接口不是网桥接口，桥不处理，交给上层协议栈           

*Bridge设置防火墙*        

Linux防火墙是通过`netfiler`这个内核框架实现，`netfiler`用于管理网络数据包。不仅具有网络地址转换(NAT)的功能，也具备数据包内容修改、以及数据包过滤等防火墙功能。利用运作于用户空间的应用软件，如iptables等来控制`netfilter`。Netfilter在内核协议栈中指定了五个处理数据包的钩子(hook)，分别是PRE_ROUTING、INPUT、OUTPUT、FORWARD与POST_ROUTING，通过iptables/ebtables等用户层工具向这些hook点注入一些数据包处理函数来实现包过滤等功能。iptables工作在IP层，只能过滤IP数据包；ebtables工作在数据链路层，只能过滤以太网帧(比如匹配源目的MAC地址)。Bridge的出现使Linux上设置防火墙变得复杂。没有Bridge存在时，Linux主机从某个网卡收到的数据包当然都是发往本机的，数据包会依次经过...,链路层(ebtables)，IP层(iptables),...解封装最后到达应用层，这样当数据到达IP层时可以很方便用iptables过滤。但是若Bridge存在时，可以参考上图，Bridge上的接口vnet0从外部收到二层数据包，关键在于此数据包不一定是发往本机，可能是发往Bridge上的另一个接口`vnet1`(tap设备，连接用户层程序，比如`qemu-kvm`)，此时`bridging decision`之后，数据包会经过链路层防火墙规则之后被转发到连接`vnet1`的vm(此时数据包进入虚拟机内部)。可以看出来，整个过程此数据包不会进入主机内核协议栈IP层(上图中向上的箭头路径)，因此位于主机IP层的iptables根本无法过滤从vnet0到vnet1的数据包。如果从防火墙角度看，在`bridging decision`之后，数据包会走链路层防火墙ebtables的`forward`链(上图中蓝色方框的`filter-forward`)，但是ebtables只能过滤数据包的二层地址。   

><small>What is bridge-nf?          
It is the infrastructure that enables {ip,ip6,arp}tables to see bridged IPv4, resp. IPv6, resp. ARP packets. Thanks to bridge-nf, you can use these tools to filter bridged packets, letting you make a transparant firewall. Note that bridge-nf is also referred to as bridge-netfilter and br-nf, the term bridge-nf should be preferred.<small>      
  
来自：[Bridge-nf Frequently Asked Questions](http://ebtables.netfilter.org/misc/brnf-faq.html)     

为了解决以上问题，Linux内核引入了`bridge_netfilter`，简称`bridge_nf`。还是参考上图说明，它使得主机IP层防火墙工具{ip,ip6,arp}tables能够"看见"Bridge中的数据包，从而我们能够在主机上使用iptables过滤Bridge中的数据包，不管此数据包是发给主机本身，还是通过Bridge转发给某台虚拟机。上图中右侧绿色方框的`managle-forward`和`filter-forward`即是iptables插入到链路层的chain。因此完整的说，在`bridging decision`之后，数据包会依次经过链路层防火墙ebtables的`forward`链(蓝色方框的`filter-forward`)，iptables的`managle-forward`和`filter-forward`链，之后被送入`vnet1`接口     

从Linux 2.6.1内核开始，可以通过设置内核参数开启`bridge_netfilter`机制。看名字就很容易知道具体作用       

``` shell
[root@controller ~]# sysctl -a |grep 'bridge-nf-'
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
...
``` 

然后在iptables中使用`-m physdev`引入相应模块，比如下面iptables规则：不跟踪从网桥`br1`的`vnet1`接口进入的数据包      

``` shell
#网桥
#brctl show
bridge name     bridge id               STP enabled     interfaces
br1             8000.f8bc1212c3a0       no              em1
                                                        vnet1
                                                        vnet3
#查看help
#iptables -m physdev -h
...
                             
#ptables -t raw -A PREROUTING -m physdev --physdev-in vnet1  -j NOTRACK    
```

在OpenStack部署中，若使用Bridge实现虚拟网络，其安全组功能就是依靠`bridge_nf`实现，计算节点上iptables才能"看见"并过滤发往其上所有instance的数据包，如下是OpenStack计算节点上部分iptables规则，当启用安全组时，OpenStack会自动设置`net.bridge.bridge-nf-call-iptables = 1`等内核参数，我们不用再明确设置。 `tap10f15e45-aa`为该计算节点上某instance网卡(就是上图中vnet0，vnet1之类的tap设备)     

``` shell
...
-A neutron-filter-top -j neutron-linuxbri-local
-A neutron-linuxbri-FORWARD -m physdev --physdev-out tap10f15e45-aa --physdev-is-bridged -m comment --comment "Direct traffic from the VM interface to the security group chain." -j neutron-linuxbri-sg-chain
-A neutron-linuxbri-FORWARD -m physdev --physdev-in tap10f15e45-aa --physdev-is-bridged -m comment --comment "Direct traffic from the VM interface to the security group chain." -j neutron-linuxbri-sg-chain
...
```
 
Bridge+netfilter内容很多，下次会专门用一篇文章介绍OpenStack中的安全组实现，关键字有 `iptables+bridge+netfilter+conntrack`        

Linux上与Bridge类似都可以作为虚拟交换机的是OVS，主要区别是OVS支持vlan tag以及流表(例如openflow)等一些高级特性，Bridge只是单纯二层交换机也不支持vlan tag，OVS具体介绍参考这里[openstack底层技术-使用openvswitch](http://www.isjian.com/openstack/openstack-base-use-openvswitch/)         

# VLAN       

这部分先介绍VLAN设备原理及配置，然后介绍VLAN在openstack中的应用    

上面简单介绍过Bridge和OVS区别，要在Linux上实现一个带VLAN功能的虚拟交换机，OVS可以通过给不同port打不同tag实现vlan功能，而Bridge需要结合VLAN设备才能实现。   

VLAN又称虚拟网络，其基本原理是在二层协议里插入额外的VLAN协议数据（称为 802.1.q VLAN Tag)，同时保持和传统二层设备的兼容性。Linux 里的VLAN设备是对 802.1.q 协议的一种内部软件实现，模拟现实世界中的 802.1.q 交换机。详细介绍参考文章开头给出的IBM文章中"VLAN device for 802.1.q"部分，这里不再重复    

下面使用使用VLAN结合Bridge新建两个网桥。最终的效果是，桥接到br100上的接口将属于vlan 100，桥接到br101上的接口属于vlan 101。前提是此主机上有一块网卡设备比如eth1，不管eth1为物理网卡或虚拟网卡，eth1所连接的交换机端口都必须设置为trunk。                        

``` shell
#网卡eth1
#cat /etc/sysconfig/network-scripts/ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=none
IPV4_FAILURE_FATAL=no
NAME=eth1
UUID=4f2cfd28-ba78-4f25-afa1-xxxxxxxxxxxxx
DEVICE=eth1
ONBOOT=yes

#vlan子设备eth1.100
#cat /etc/sysconfig/network-scripts/ifcfg-eth1.100
DEVICE=eth1.100
BOOTPROTO=static
ONBOOT=yes
VLAN=yes

#vlan子设备eth1.101
#cat /etc/sysconfig/network-scripts/ifcfg-eth1.101
DEVICE=eth1.101
BOOTPROTO=static
ONBOOT=yes
VLAN=yes

#查看eth1.101，可以看到，其driver为VALN
#ethtool -i eth1.101
driver: 802.1Q VLAN Support
version: 1.8
...

#重启网卡
#/etc/init.d/network restart

#设置网桥
#brctl addbr br100
#brctl addbr br101
#brctl addif br100 eth1.100
#brctl addif br100 vnet1
#brctl addif br101 eth1.101
#brctl show
bridge name     bridge id               STP enabled     interfaces
br100         8000.525400315e23       no              eth1.100   
													  vnet1
br101         8000.525400315e23       no              eth1.101
```

VLAN设备的作用是建立一个个带不同vlan tag的子设备，它并不能建立多个可以交换转发数据的接口，因此需要借助于Bridge，把VLAN建立的子设备例如eth1.100桥接到网桥br100上，这样凡是桥接到br100上的接口就自动加入了vlan 100子网。对比一台带有两个vlan 100，101的物理交换机，这里br100所连接的接口相当于物理交换机上那些划分到vlan 100的端口，而br101所连接的接口相当于物理交换机上那些划分到vlan 101的端口。因此Bridge加VLAN能在功能层面完整模拟现实世界里的802.1.q交换机。     

下面，我们从网桥br100上vnet1接口角度，来看下具体的数据收发流程：  

- vnet1发送数据      

vnet1发送数据到br100 --> br100把数据从eth1.100发出 --> 母设备收到eth1.100发来的数据 --> 母设备eth1给数据打上100的vlan tag --> eth1将带有100 tag的数据发出 --> eth1连接的交换机收到数据(trunk口)        

- vnet1接收数据

eth1从所连接的交换机收到tag 100的数据 --> eth1发现此数据带有tag 100,移除数据包中tag --> 不带tag的数据发给eth1.100 --> br100收到从eth1.100进入的数据包 --> br100转发数据到vnet1            

上面忽略了对于数据包是否带tag，以及数据包所带tag的子设备是否存在的检查。这些属于vlan基础知识      

**openstack中的vlan设置**    

openstack中虚拟机网络使用VLAN模式的话，就会用到VLAN设备。openstack中配置vlan+bridge模式如下

``` shell
#neutron-server节点(网络节点)配置 
#cat /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
#neutron-server启动时，加载flat，vlan两种网络类型驱动   
type_drivers = flat,vlan
#vlan模式不需要tenant_network，留空    
tenant_network_types =
#neutron-server启动时加载linuxbridge和openvswitch网桥驱动    
mechanism_drivers = linuxbridge,openvswitch

[ml2_type_flat]
#在命令行或控制台新建flat类型网络时需要指定的名称，此名称会配置映射到计算节点上某块做外网的网卡，下面会设置
flat_networks = proext

[ml2_type_vlan]
#在命令行或控制台新建vlan类型网络时需要指定的名称，此名称会配置映射到计算节点上某块做vlan网络的网卡，下面会设置
network_vlan_ranges = provlan

#重启neutron-server服务  

#使用Bridge+vlan网络模式的nova-compute节点(计算节点)配置 
#cat /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
#provlan名称映射到此计算节点eth2网卡，因为使用vlan模式，eth2需要设置为trunk
#proext名称映射到此计算节点eth3网卡，eth3网卡为虚拟机外网网络接口       
physical_interface_mappings = provlan:eth2,proext:eth3

#重启此计算节点nova-compute服务   
#配置中只需要指定vlan要用的母设备eth2，后续控制台新建带tag的网络时，neutron会自动建立eth2.{TAG}子设备并加入到网桥    
```    

在控制台新建一个vlan tag为1023的虚拟机网络subvlan-1023，使用此subvlan-1023网络新建几台虚拟机，看下计算节点上网桥配置       

``` shell
[root@compute03 neutron]# brctl show
bridge name     bridge id               STP enabled     interfaces
brq82405415-7a          8000.52540048b1a9       no              eth2.1023
                                                        tap10f15e45-aa
                                                        tapa659a214-b1
brqf5808b72-44          8000.5254001ac83d       no              eth3
                                                        tapd3388a60-ae
                                              
[root@compute03 neutron]# virsh domiflist instance-00000145
Interface  Type       Source     Model       MAC
-------------------------------------------------------
tapa659a214-b1 bridge     brq82405415-7a virtio      fa:16:3e:bc:c9:e0
```

虚拟机`instance-00000145`的网卡`tapa659a214-b1`桥接到`brq82405415-7a`。跟上面介绍的类似，桥接到`brq82405415-7a`上的接口设备就自动加入了vlan 1023子网，因此`instance-00000145`属于vlan 1023子网。   

假如在控制台或命令行再新建一个tag为1024的子网，则网桥配置如下    

``` shell
[root@compute03 neutron]# brctl show
bridge name     bridge id               STP enabled     interfaces
brq82405415-7a          8000.52540048b1a9       no              eth2.1023
                                                        tap10f15e45-aa
                                                        tapa659a214-b1
brq7d59440b-cc          8000.525400aabbcc       no              eth2.1024
                                                        tap20ffafb2-1b
brqf5808b72-44          8000.5254001ac83d       no              eth3
                                                        tapd3388a60-ae
                                                        
[root@compute03 neutron]# virsh domiflist instance-00000147
Interface  Type       Source     Model       MAC
-------------------------------------------------------
tap20ffafb2-1b bridge     brq7d59440b-cc virtio      fa:16:3e:bd:12:40
```  

虚拟机`instance-00000147`的网卡`tap20ffafb2-1b`桥接到`brq7d59440b-cc`,`instance-00000147`属于vlan 1024子网,这就实现了属于不同vlan子网的`instance-00000145`与`instance-00000147`的隔离性。         

感谢Bridge和VLAN设备，他们让openstack配置vlan网络成了可能，BUT!, Bridge+VLAN不是唯一的选择，openstack也支持OVS，OVS中是靠给不同instance接口打不同tag来实现instance的多vlan环境，OVS模式除了配置部分跟Bridge+VLAN不同之外，使用上并没有什么区别，这里的设置`mechanism_drivers = linuxbridge,openvswitch`加载相应驱动，屏蔽掉了底层操作的差别   

与Bridge中`provlan，proext`映射到计算节点网卡的配置不同，OVS配置文件中映射关系为vlan类型网络`provlan`映射到网桥`br-vlan`，flat类型网络`proext`映射到网桥`br-ext`。至于`br-vlan`桥接eth2网卡，`br-ext`桥接eth3网卡则需要预先手动配置好，来看一个使用OVS的计算节点网桥     
 
``` shell
[root@compute01 neutron]# ovs-vsctl show
dd7ccaae-6a24-4d28-8577-9e5e6b5dfbd3
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-ext
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port phy-br-ext
            Interface phy-br-ext
                type: patch
                options: {peer=int-br-ext}
        Port br-ext
            Interface br-ext
                type: internal
        Port "eth3"
            Interface "eth3"
    Bridge br-vlan
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-vlan
            Interface br-vlan
                type: internal
        Port phy-br-vlan
            Interface phy-br-vlan
                type: patch
                options: {peer=int-br-vlan}
        Port "eth2"
            Interface "eth2"
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port int-br-ext
            Interface int-br-ext
                type: patch
                options: {peer=phy-br-ext}
        Port br-int
            Interface br-int
                type: internal
        Port int-br-vlan
            Interface int-br-vlan
                type: patch
                options: {peer=phy-br-vlan}
        Port "qvo7d59440b-cc"
            tag: 1
            Interface "qvo7d59440b-cc"
    ovs_version: "2.5.0"
[root@compute01 neutron]# brctl show
bridge name     bridge id               STP enabled     interfaces
qbr7d59440b-cc          8000.26d03016fcf6       no              qvb7d59440b-cc
                                                        tap7d59440b-cc
                                                        
#查看虚拟机网卡
[root@compute01 neutron]# virsh domiflist instance-00000149
Interface  Type       Source     Model       MAC
-------------------------------------------------------
tap7d59440b-cc bridge     qbr7d59440b-cc virtio      fa:16:3e:12:ba:e6
``` 

`instance-00000149`出数据流向为`tap7d59440b-cc --> qbr7d59440b-cc --> qvo7d59440b-cc(tag 1) --> br-int --> br-vlan --> eth2`。qbr7d59440b-cc 与 qvo7d59440b-cc 为一对veth设备    

这其中牵涉OVS流表和OVS内外部tag转换问题，又足够写一篇文章来介绍了，本文暂不继续介绍。还有一点，在使用OVS做网桥的同时又开启安全组功能时，会多出一个Bridge网桥用于设置安全组，如上面的`qbr7d59440b-cc`, 因为目前iptables不支持OVS，只能在虚拟机与OVS网桥之间加进一个Bridge网桥用于设置iptables规则      

其它网络设备会在另一篇文章介绍，本文完     
