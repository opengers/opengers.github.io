---
title: openstack底层技术-各种虚拟网络设备二(tun/tap,veth,vxlan/grep)      
author: opengers
layout: post
permalink: /openstack/openstack-base-virtual-network-devices-part2-tuntap-gre-vxlan/
categories: openstack
tags:
  - openstack
  - tun/tap
  - veth
  - vxlan/gre
---

[openstack底层技术-各种虚拟网络设备一(Bridge,VLAN)](http://www.isjian.com/openstack/openstack-base-virtual-network-devices-part1-bridge-and-vlan/)     
[openstack底层技术-各种虚拟网络设备二(tun/tap,veth,vxlan/grep)](http://www.isjian.com/openstack/openstack-base-virtual-network-devices-part2-tuntap-gre-vxlan/)     

><small>第一篇文章介绍了Bridgge和VLAN设备，本文继续介绍tun/tap，veth，vxlan/gre等设备，很多时候这些设备不是独立使用的，下文会看到</small>      

* TOC
{:toc}    

我们知道KVM虚拟化中单个虚拟机是主机上的一个普通`qemu-kvm`进程，虚拟机当然也需要网卡，最常见的虚拟网卡就是使用主机上的tap设备。那从主机的角度看，这个`qemu-kvm`进程是如何使用tap设备呢，下面先介绍下`tun/tap`设备概念，然后分别用一个实例来解释`tun/tap`的具体用途                              

# tun/tap     

`tun/tap`设备是操作系统内核中的虚拟网络设备，他们为用户层程序提供数据的接收与传输。`tun/tap`设备使用内核模块为`tun`，其模块介绍为`Universal TUN/TAP device driver`，该模块提供了一个设备接口`/dev/net/tun`供用户层程序读写，用户层程序通过读写`/dev/net/tun`接口来向主机内核协议栈注入数据或接收来自主机内核协议栈的数据。**可以把tun/tap看成数据管道，它一端连接主机协议栈，另一端连接用户程序** **                

``` shell
[root@compute01 ~]# modinfo tun
filename:       /lib/modules/3.10.0-514.16.1.el7.x86_64/kernel/drivers/net/tun.ko
alias:          devname:net/tun
...
description:    Universal TUN/TAP device driver
...

[root@compute01 ~]# ls /dev/net/tun 
/dev/net/tun
```

为了使用`tun/tap`设备，用户层程序需要通过系统调用打开`/dev/net/tun`获得一个读写该设备的文件描述符(FD)，并且调用ioctl()向内核注册一个TUN或TAP类型的虚拟网卡(实例化一个tun/tap设备)，其名称可能是`tap7b7ee9a9-c1/vnetXX/tunXX/tap0`等。此后，用户程序可以通过该虚拟网卡与主机内核协议栈交互。当用户层程序关闭后，其注册的TUN或TAP虚拟网卡以及路由表相关条目(使用tun可能会产生路由表条目，比如openvpn)都会被内核释放。可以把用户层程序看做是网络上另一台主机，他们通过tap虚拟网卡相连       

TUN和TAP设备区别在于他们工作的协议栈层次不同，TAP等同于一个以太网设备，用户层程序向tap设备读写的是二层数据包如以太网数据帧，tap设备最常用的就是作为虚拟机网卡。TUN则模拟了网络层设备，操作第三层数据包比如IP数据包，`openvpn`使用TUN设备在C/S间建立VPN隧道                    

# tap设备作为虚拟机网卡     

tap设备最常见的用途就是作为虚拟机网卡，这里以一个具体的虚拟机P2为例，来看看它与其所在主机，及网络上其它主机通信的数据流向                                     

本文依然使用下面这张图作为参考                                     

![bridge](/images/openstack/openstack-virtual-devices/bridge.png)     

虚拟机P2使用桥接模式，网桥为br0，先启动虚拟机                    

``` shell
#virsh start P2
#virsh list --all
 Id    Name                           State
----------------------------------------------------
 23    P2                             running
 
#virsh domiflist 9
Interface  Type       Source     Model       MAC
-------------------------------------------------------
tap0       bridge     br0        virtio      fa:16:3e:b1:70:52
...
```

虚拟机启动后，主机上多了一张虚拟网卡`tap0`，`tap0`桥接在br0网桥上，查看此虚拟机进程        

``` shell
#ps -ef |grep P2
qemu      7748     1  0 Nov07 ?        00:22:09 /usr/libexec/qemu-kvm -name guest=P2 ... -netdev tap,fd=26,id=hostnet0,vhost=on,vhostfd=28 ...      
```

进程PID为7748，其网络部分参数中，`-netdev tap,fd=26` 表示连接主机上的tap设备，`fd=26`为读写`/dev/net/tun`的文件描述符。使用`lsof -p 7748`也可以验证，如下，PID为7748的进程打开了文件/dev/net/tun，分配的文件描述符为26，打开设备文件类型为CHR    

``` shell
# lsof -p 7748
COMMAND    PID USER   FD      TYPE             DEVICE    SIZE/OFF     NODE NAME
...
qemu-kvm 7748  qemu   26u      CHR             10,200         0t0    17439 /dev/net/tun
...             
```

因此，在虚拟机P2启动时，其打开了设备文件`/dev/net/tun`并获得了读写该文件的文件描述符(FD)26，同时向内核注册了一个tap类型虚拟网卡`tap0`，`tap0`与FD 26关联，虚拟机关闭时`tap0`设备会被内核释放     

**从虚拟机P2发送 数据到外部网络**              

1. 虚拟机通过其网卡eth0向外发送数据，从主机角度看，就是`qemu-kvm`进程使用文件描述符(FD)26向`/dev/net/tun`写入数据 `P2 --> write(fd,...) --> N`    
1. 文件描述符26与虚拟网卡`tap0`关联，主机从`tap0`网卡收到数据 `N --> E`       
1. `tap0`为网桥`br0`上一个接口，进行`Bridging decision`  `E --> D`             
1. P2是与网络上其它主机通信，`br0`转发该数据到`em2.100`，最后从物理网卡em2发出  `D --> C --> B -- > A`           

这个过程中，虚拟机发出的数据通过`tap0`虚拟网卡直接注入主机链路层网桥处理逻辑中，然后被转发到外部网络。可以看出数据包没有穿过主机协议栈，这是当然的也是必须的，因为虚拟机通信对象不是此主机，明白这点之后，可以对tap设备有更深的认识               

# openvpn中使用的tun设备              

TUN设备与上面介绍的TAP设备很类似，只是TUN设备连接的是IP层         

openvpn是使用TUN设备的一个常见例子，启动openvpn server，主机中多了一个虚拟网卡`tun0`        

``` shell
#ifconfig tun0
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
          inet addr:10.5.0.1  P-t-P:10.5.0.2  Mask:255.255.255.255
...
```

可以看到`tun0`网卡使用内核模块`tun`，类型为`tun`设备      
#ethtool -i tun0
driver: tun
...
bus-info: tun
...

```

使用`lsof`查看openvpn进程打开的所有文件        

``` shell
# ps -ef |grep openvpn
root      3056     1  4 Nov14 ?        03:18:28 /usr/sbin/openvpn --daemon --config /etc/openvpn/config/server.conf

#lsof -p 3056
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF     NODE NAME
...
openvpn 3056 root    6u   CHR 10,200      0t0     8590 /dev/net/tun
openvpn 3056 root    7u  IPv4  26744      0t0      UDP *:9112
...
```

很明显，openvpn进程打开了`/dev/net/tun`并获得了文件描述符6，还有一个udp协议的socket，监听在`9112`端口。       

**客户端使用openvpn访问web服务**        

1. 客户端通过vpn访问web服务      
1. openvpn client封装IP数据包，通过udp协议发送vpn封包到openvpn server上的9112端口  `A -- > T --> K --> R --> P1`      
1. openvpn进程收到vpn封包，解包，使用文件描述符6写数据到`/dev/net/tun` `P1 --> write(fd,...) --> N`     
1. 文件描述符6与虚拟网卡`tun0`关联，主机从`tun0`网卡收到数据包 `N --> H ---> I`     
1. 主机进行`Routing decision`，根据数据包目的IP从相应网卡发出 `I --> K --> T --> M --> A`             

# veth设备    

veth设备对的作用是反转数据流方向，一个用途是连接多个namespace。openstack虚拟网络实现中，dhcp agent和l3 agent都用到了veth设备对             

``` shell
#tap设备
ip tuntap add mode tap
#tun设备
ip tuntap add mode tun
#veth设备veth-a,veth-b
ip link add veth-a type veth peer name veth-b
#namespace
ip netns add nstest

#添加一个tun设备，并配置ip
ip tuntap add mode tun
ip link set tun0 up
ip addr add 172.16.1.2/24 dev tun0
ip link del dev tun0

#将 veth-b 添加到 network namespace
ip link set veth-b netns nstest

#设置ip地址
ip netns exec nstest ip addr add 10.0.0.2/32 dev veth-b
ip netns exec nstest ip link set dev veth-b up

#查看路由表和 iptbales
# ip netns exec netns1 route
# ip netns exec netns1 iptables -L
http://www.opencloudblog.com/?p=66


查看系统veth设备对
[root@controller yum.repos.d]# ethtool -S veth-b
NIC statistics:
     peer_ifindex: 14
[root@controller yum.repos.d]# ip a |grep '^14 '
[root@controller yum.repos.d]# ip a |grep '^14:'
14: veth-a@veth-b: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN qlen 1000

#所有接口详细信息
ip -d link show
```

(本文完)    